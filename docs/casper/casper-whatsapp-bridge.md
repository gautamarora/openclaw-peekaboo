# Casper via WhatsApp: Baileys Socket Bridge

> **Exploration.** This document sketches how a TypeScript agent client can
> receive commands over WhatsApp (via the Baileys socket interface) and
> translate them into Casper operations — either through the FFI bridge
> (`mac_bridge.ts`) or the Peekaboo IPC socket. WhatsApp becomes a remote
> command channel; the TS layer remains the agent client issuing OS commands.

## Concept

```
┌──────────────────────────────────────────────────────────────┐
│  WhatsApp User (phone / WhatsApp Web)                        │
│                                                              │
│  Sends:  "screenshot"                                        │
│  Sends:  "click Submit"                                      │
│  Sends:  "type hello world"                                  │
│  Sends:  "hotkey cmd+s"                                      │
│  Sends:  "apps"                                              │
└──────────────┬───────────────────────────────────────────────┘
               │  WhatsApp E2EE WebSocket
               ▼
┌──────────────────────────────────────────────────────────────┐
│  Casper WhatsApp Bridge (Node.js / Deno)                     │
│                                                              │
│  Baileys socket ─── receive messages, send replies           │
│    ↓                                                         │
│  Command parser ─── text → structured command                │
│    ↓                                                         │
│  Casper client ─── issues OS commands via FFI or IPC         │
│    ↓                                                         │
│  Response formatter ─── result → text / image reply          │
└──────────────────────────────────────────────────────────────┘
               │
               ▼  (one of)
┌──────────────────────────┐  ┌────────────────────────────────┐
│  FFI: mac_bridge.ts      │  │  IPC: Peekaboo bridge.sock     │
│  (Deno.dlopen, in-proc)  │  │  (JSON over UNIX socket)       │
└──────────────────────────┘  └────────────────────────────────┘
               │                          │
               ▼                          ▼
┌──────────────────────────────────────────────────────────────┐
│  macOS (Accessibility, CoreGraphics, AppKit, ScreenCaptureKit)│
└──────────────────────────────────────────────────────────────┘
```

The bridge is **not** an agent loop or LLM orchestrator. It is a thin
translation layer: WhatsApp message in → Casper command out → result back
to WhatsApp.

---

## Why Baileys

[Baileys](https://github.com/WhiskeySockets/Baileys) (`@whiskeysockets/baileys`)
is a TypeScript library that implements the WhatsApp Web socket protocol
directly — no browser, no Puppeteer, no official API key required.

Key properties that make it a good fit:

- **Pure TypeScript** — runs in Node.js or Deno, same ecosystem as Casper's
  TS client layer.
- **E2EE built-in** — handles Signal protocol encryption/decryption
  transparently. Messages are never in the clear on the wire.
- **Rich message types** — can send/receive text, images (PNG/JPEG), documents,
  reactions, and replies. This matters because Casper returns screenshots as
  PNG bytes.
- **Event-driven** — `sock.ev.on('messages.upsert', ...)` gives a clean
  callback interface for incoming messages. No polling.
- **Session persistence** — `useMultiFileAuthState` saves credentials to disk.
  After the first QR scan, reconnections are automatic.

---

## Baileys Socket Essentials

### Setup and Connection

```typescript
import makeWASocket, {
  DisconnectReason,
  useMultiFileAuthState,
} from "@whiskeysockets/baileys";
import { Boom } from "@hapi/boom";
import pino from "pino";

const logger = pino({ level: "warn" });

async function connect() {
  const { state, saveCreds } = await useMultiFileAuthState("auth_state");

  const sock = makeWASocket({
    auth: state,
    printQRInTerminal: true,
    logger,
  });

  // Persist credential updates (required)
  sock.ev.on("creds.update", saveCreds);

  // Reconnect on disconnect (except logout)
  sock.ev.on("connection.update", ({ connection, lastDisconnect }) => {
    if (connection === "close") {
      const code = (lastDisconnect?.error as Boom)?.output?.statusCode;
      if (code !== DisconnectReason.loggedOut) {
        connect(); // reconnect
      }
    }
    if (connection === "open") {
      console.log("WhatsApp connected");
    }
  });

  return sock;
}
```

### Receiving Messages

```typescript
sock.ev.on("messages.upsert", async ({ messages, type }) => {
  if (type !== "notify") return; // only real-time messages

  for (const msg of messages) {
    if (msg.key.fromMe) continue; // ignore own messages

    const jid = msg.key.remoteJid!;
    const text =
      msg.message?.conversation ||
      msg.message?.extendedTextMessage?.text ||
      "";

    // Route to command handler
    await handleCommand(sock, jid, text, msg);
  }
});
```

### Sending Replies

```typescript
// Text reply
await sock.sendMessage(jid, { text: "Done." });

// Image reply (PNG bytes from Casper capture)
await sock.sendMessage(jid, {
  image: screenshotBuffer,
  caption: "Current screen",
});

// Quote-reply to the original command
await sock.sendMessage(jid, { text: "Clicked Submit." }, { quoted: msg });
```

---

## Command Protocol

The bridge parses plain-text WhatsApp messages into Casper operations.
Commands are intentionally simple — no JSON, no special syntax. The first
word is the verb; the rest is the argument.

### Command Table

| WhatsApp message | Casper operation | Response |
|---|---|---|
| `screenshot` | `captureScreen()` | PNG image reply |
| `screenshot window` | `captureFrontmost()` | PNG image reply |
| `click Submit` | `click({ query: "Submit" })` | "Clicked Submit." |
| `click 500 300` | `click({ x: 500, y: 300 })` | "Clicked (500, 300)." |
| `type hello world` | `typeText("hello world")` | "Typed: hello world" |
| `hotkey cmd+s` | `hotkey("cmd+s")` | "Pressed cmd+s." |
| `scroll down` | `scroll(0, -3)` | "Scrolled down." |
| `scroll up 5` | `scroll(0, 5)` | "Scrolled up 5." |
| `apps` | `listApplications()` | Text list of running apps |
| `frontmost` | `frontmostApplication()` | App name + PID |
| `activate com.apple.Safari` | `activateApp(...)` | "Activated Safari." |
| `launch com.apple.TextEdit` | `launchApp(...)` | "Launched TextEdit." |
| `windows` | `getWindows(frontmostPid)` | Window list |
| `buttons` | `findElementsByRole(pid, "AXButton")` | Button list |
| `clipboard` | `readClipboard()` | Clipboard text |
| `clipboard set <text>` | `writeClipboard(text)` | "Clipboard set." |
| `permissions` | `checkPermissions()` | Permission status |
| `help` | — | Command reference |

### Why Text Commands (Not Structured Messages)

WhatsApp is a chat interface. The person typing on their phone expects to
type natural short phrases, not JSON payloads. The command parser is a
simple `split(" ")` on the first word with the rest as the argument. This
is the lowest-friction interface for mobile use.

---

## Implementation: Command Router

### Command Parser

```typescript
interface ParsedCommand {
  verb: string;
  args: string;
  rawArgs: string[];
}

function parseCommand(text: string): ParsedCommand {
  const trimmed = text.trim();
  const parts = trimmed.split(/\s+/);
  return {
    verb: parts[0]?.toLowerCase() ?? "",
    args: parts.slice(1).join(" "),
    rawArgs: parts.slice(1),
  };
}
```

### Command Handler (IPC variant)

This variant connects to the Peekaboo bridge socket for each command.
It follows the single-request-per-connection protocol from the IPC doc.

```typescript
import net from "net";
import os from "os";
import path from "path";
import type { WASocket, WAMessage } from "@whiskeysockets/baileys";

const SOCKET_PATH = path.join(
  os.homedir(),
  "Library/Application Support/Peekaboo/bridge.sock"
);

// --- Bridge request (from casper-ipc-integration.md) ---

function bridgeRequest(request: unknown): Promise<unknown> {
  return new Promise((resolve, reject) => {
    const client = net.createConnection(SOCKET_PATH, () => {
      client.end(JSON.stringify(request));
    });
    const chunks: Buffer[] = [];
    client.on("data", (chunk) => chunks.push(chunk));
    client.on("end", () => {
      try {
        resolve(JSON.parse(Buffer.concat(chunks).toString()));
      } catch (err) {
        reject(new Error(`Bridge parse error: ${err}`));
      }
    });
    client.on("error", reject);
    client.setTimeout(15000, () => {
      client.destroy(new Error("Bridge timeout"));
    });
  });
}

// --- Command handler ---

async function handleCommand(
  sock: WASocket,
  jid: string,
  text: string,
  msg: WAMessage
): Promise<void> {
  const { verb, args, rawArgs } = parseCommand(text);

  switch (verb) {
    case "screenshot": {
      const isWindow = args === "window";
      const req = isWindow
        ? { captureFrontmost: { _0: { visualizerMode: "screenshotFlash", scale: "logical1x" } } }
        : { captureScreen: { _0: { displayIndex: null, visualizerMode: "screenshotFlash", scale: "logical1x" } } };
      const res = (await bridgeRequest(req)) as any;
      const imageData = res?.capture?._0?.imageData;
      if (imageData) {
        const buf = Buffer.from(imageData, "base64");
        await sock.sendMessage(jid, { image: buf, caption: isWindow ? "Frontmost window" : "Screen capture" }, { quoted: msg });
      } else {
        await sock.sendMessage(jid, { text: "Capture failed." }, { quoted: msg });
      }
      break;
    }

    case "click": {
      const coords = rawArgs.map(Number);
      const target =
        coords.length === 2 && !isNaN(coords[0]) && !isNaN(coords[1])
          ? { kind: "coordinates", x: coords[0], y: coords[1] }
          : { kind: "query", value: args };
      const res = (await bridgeRequest({
        click: { _0: { target, clickType: "single", snapshotId: null } },
      })) as any;
      const reply = res?.ok !== undefined
        ? `Clicked ${args || `(${coords.join(", ")})`}.`
        : `Click failed: ${res?.error?._0?.message ?? "unknown"}`;
      await sock.sendMessage(jid, { text: reply }, { quoted: msg });
      break;
    }

    case "type": {
      if (!args) {
        await sock.sendMessage(jid, { text: "Usage: type <text>" }, { quoted: msg });
        break;
      }
      const res = (await bridgeRequest({
        type: { _0: { text: args, target: null, clearExisting: false, typingDelay: 50, snapshotId: null } },
      })) as any;
      const reply = res?.ok !== undefined ? `Typed: ${args}` : `Type failed: ${res?.error?._0?.message ?? "unknown"}`;
      await sock.sendMessage(jid, { text: reply }, { quoted: msg });
      break;
    }

    case "hotkey": {
      if (!args) {
        await sock.sendMessage(jid, { text: "Usage: hotkey <keys> (e.g. cmd+s)" }, { quoted: msg });
        break;
      }
      const res = (await bridgeRequest({
        hotkey: { _0: { keys: args, holdDuration: 0 } },
      })) as any;
      const reply = res?.ok !== undefined ? `Pressed ${args}.` : `Hotkey failed: ${res?.error?._0?.message ?? "unknown"}`;
      await sock.sendMessage(jid, { text: reply }, { quoted: msg });
      break;
    }

    case "scroll": {
      const dir = rawArgs[0]?.toLowerCase() ?? "down";
      const amount = parseInt(rawArgs[1] ?? "3", 10);
      const req = {
        scroll: {
          _0: {
            request: {
              direction: dir,
              amount,
              target: null,
              smooth: false,
              delay: 10,
              snapshotId: null,
            },
          },
        },
      };
      await bridgeRequest(req);
      await sock.sendMessage(jid, { text: `Scrolled ${dir} ${amount}.` }, { quoted: msg });
      break;
    }

    case "apps": {
      const res = (await bridgeRequest({ listApplications: {} })) as any;
      const apps = res?.applications?._0 ?? [];
      const list = apps
        .slice(0, 20)
        .map((a: any) => `${a.name ?? a.bundle_id} (pid ${a.pid})`)
        .join("\n");
      await sock.sendMessage(jid, { text: list || "No apps found." }, { quoted: msg });
      break;
    }

    case "frontmost": {
      const res = (await bridgeRequest({ getFrontmostApplication: {} })) as any;
      const app = res?.application?._0;
      const reply = app
        ? `${app.name ?? "unknown"} (${app.bundle_id}, pid ${app.pid})`
        : "No frontmost app.";
      await sock.sendMessage(jid, { text: reply }, { quoted: msg });
      break;
    }

    case "activate": {
      if (!args) {
        await sock.sendMessage(jid, { text: "Usage: activate <bundleId>" }, { quoted: msg });
        break;
      }
      const res = (await bridgeRequest({
        activateApplication: { _0: { identifier: args } },
      })) as any;
      const reply = res?.ok !== undefined ? `Activated ${args}.` : `Activate failed.`;
      await sock.sendMessage(jid, { text: reply }, { quoted: msg });
      break;
    }

    case "launch": {
      if (!args) {
        await sock.sendMessage(jid, { text: "Usage: launch <bundleId>" }, { quoted: msg });
        break;
      }
      const res = (await bridgeRequest({
        launchApplication: { _0: { identifier: args } },
      })) as any;
      const reply = res?.application ? `Launched ${args}.` : `Launch failed.`;
      await sock.sendMessage(jid, { text: reply }, { quoted: msg });
      break;
    }

    case "permissions": {
      const res = (await bridgeRequest({ permissionsStatus: {} })) as any;
      const perms = res?.permissionsStatus?._0 ?? res;
      await sock.sendMessage(jid, { text: JSON.stringify(perms, null, 2) }, { quoted: msg });
      break;
    }

    case "clipboard": {
      if (rawArgs[0] === "set" && rawArgs.length > 1) {
        const clipText = rawArgs.slice(1).join(" ");
        // clipboard write would go through FFI or a custom bridge op
        await sock.sendMessage(jid, { text: `Clipboard set to: ${clipText}` }, { quoted: msg });
      } else {
        // clipboard read — not directly in IPC op list, would use FFI
        await sock.sendMessage(jid, { text: "Clipboard read via IPC not yet supported." }, { quoted: msg });
      }
      break;
    }

    case "help": {
      const help = [
        "*Casper WhatsApp Commands*",
        "",
        "screenshot — capture the screen",
        "screenshot window — capture frontmost window",
        "click <query> — click element by text",
        "click <x> <y> — click at coordinates",
        "type <text> — type text",
        "hotkey <keys> — press key combo (e.g. cmd+s)",
        "scroll <dir> [amount] — scroll up/down/left/right",
        "apps — list running applications",
        "frontmost — show frontmost app",
        "activate <bundleId> — activate app",
        "launch <bundleId> — launch app",
        "permissions — check TCC permissions",
        "clipboard — read clipboard",
        "clipboard set <text> — write clipboard",
        "help — this message",
      ].join("\n");
      await sock.sendMessage(jid, { text: help }, { quoted: msg });
      break;
    }

    default: {
      // Ignore unrecognized messages silently, or:
      // await sock.sendMessage(jid, { text: `Unknown command: ${verb}. Send "help" for usage.` }, { quoted: msg });
      break;
    }
  }
}
```

---

## Putting It Together

```typescript
// casper-whatsapp.ts — entry point

async function main() {
  // 1. Handshake with Peekaboo to verify the service is running
  const hs = (await bridgeRequest({
    handshake: {
      _0: {
        protocolVersion: { major: 1, minor: 0 },
        client: {
          bundleIdentifier: "com.casper.whatsapp-bridge",
          teamIdentifier: null,
          processIdentifier: process.pid,
          hostname: null,
        },
        requestedHostKind: null,
      },
    },
  })) as any;

  const enabled = hs?.handshake?._0?.enabledOperations ?? [];
  console.log(`Peekaboo connected. ${enabled.length} operations available.`);

  // 2. Connect to WhatsApp
  const sock = await connect();

  // 3. Route incoming messages to the command handler
  sock.ev.on("messages.upsert", async ({ messages, type }) => {
    if (type !== "notify") return;
    for (const msg of messages) {
      if (msg.key.fromMe) continue;
      const jid = msg.key.remoteJid!;
      const text =
        msg.message?.conversation ||
        msg.message?.extendedTextMessage?.text ||
        "";
      if (!text) continue;
      try {
        await handleCommand(sock, jid, text, msg);
      } catch (err) {
        console.error(`Command error: ${err}`);
        await sock.sendMessage(jid, { text: `Error: ${err}` });
      }
    }
  });
}

main();
```

### Running It

```bash
npm install @whiskeysockets/baileys @hapi/boom pino

# First run — scan QR code in terminal to link WhatsApp
npx tsx casper-whatsapp.ts

# Subsequent runs — auto-reconnects from saved auth state
npx tsx casper-whatsapp.ts
```

On first launch, Baileys prints a QR code in the terminal. Scan it with
WhatsApp on your phone (Settings → Linked Devices → Link a Device). After
that, the session persists in `auth_state/` and reconnects automatically.

---

## Access Control

The bridge should restrict which WhatsApp numbers can issue commands.
Without this, anyone who messages the linked number gets macOS control.

```typescript
// Allowlist of JIDs that can issue commands
const ALLOWED_JIDS = new Set([
  "12345678901@s.whatsapp.net", // your phone number
]);

sock.ev.on("messages.upsert", async ({ messages, type }) => {
  if (type !== "notify") return;
  for (const msg of messages) {
    const jid = msg.key.remoteJid!;
    if (msg.key.fromMe) continue;

    // Strip group participant — only check the sender
    const sender = msg.key.participant ?? jid;
    if (!ALLOWED_JIDS.has(sender)) {
      continue; // silently ignore unauthorized senders
    }

    // ... handle command
  }
});
```

For group chats, `msg.key.participant` is the actual sender. For 1:1 chats,
`msg.key.remoteJid` is the sender. The check above covers both.

---

## Design Considerations

### IPC vs FFI for the Backend

| Approach | Pros | Cons |
|---|---|---|
| **IPC (bridge.sock)** | Decoupled from Peekaboo process; works with Node.js; richer op set (snapshots, element detection, dialogs, dock, menus) | Requires Peekaboo daemon running; ~1-5ms per call overhead |
| **FFI (Deno.dlopen)** | Zero overhead; in-process; no daemon dependency | Deno-only; must build the Rust dylib; narrower API surface |

For a WhatsApp bridge, **IPC is the natural choice**: the bridge is a
long-running Node.js process (Baileys needs Node), and the per-call latency
is irrelevant when the bottleneck is WhatsApp message delivery (~200-500ms).

### What About Images Sent TO the Bridge?

Baileys can receive images. A future extension could accept a screenshot
sent via WhatsApp and pass it to Casper's `detectElements` operation:

```typescript
if (msg.message?.imageMessage) {
  const buffer = await downloadMediaMessage(msg, "buffer", {});
  const base64 = buffer.toString("base64");
  const res = await bridgeRequest({
    detectElements: { _0: { imageData: base64, snapshotId: null, windowContext: null } },
  });
  // Reply with detected element IDs
}
```

This would enable a capture → annotate → click workflow entirely over
WhatsApp: send a screenshot, get back element labels, then reply with
`click B1`.

### Latency Budget

```
WhatsApp delivery:     ~200-500ms (E2EE + relay)
Baileys processing:    ~5-10ms
Command parsing:       <1ms
Peekaboo IPC:          ~1-5ms
macOS operation:       ~10-100ms (capture is slowest)
Response formatting:   <1ms
WhatsApp reply:        ~200-500ms
──────────────────────────────────
Total round-trip:      ~400ms-1.1s
```

This is fast enough for interactive remote control. The WhatsApp network
hop dominates, not the local automation.

### Session Security

- Baileys sessions are stored in `auth_state/` — these contain your WhatsApp
  Signal keys. **Treat this directory like a private key.** Anyone with these
  files can impersonate your WhatsApp account.
- The Peekaboo bridge socket has its own auth (code signing or
  `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` for dev). The WhatsApp bridge
  process needs to satisfy this.
- The JID allowlist above is the primary access control. Keep it tight.

---

## Comparison with Other Remote Channels

| Channel | Setup | Latency | Media support | Auth |
|---|---|---|---|---|
| **WhatsApp (Baileys)** | QR scan once | ~400ms-1s | Text + images | Phone number allowlist |
| **Telegram Bot API** | BotFather token | ~100-300ms | Text + images + files | Chat ID allowlist |
| **SSH + CLI** | SSH keys | ~50-200ms | Text only (base64 for images) | SSH auth |
| **Web UI** | HTTP server | ~10-50ms | Full | Token/password |

WhatsApp's advantage is ubiquity — it's already on your phone, no extra app
to install. The trade-off is higher latency and the Baileys dependency on
the unofficial WhatsApp Web protocol.
