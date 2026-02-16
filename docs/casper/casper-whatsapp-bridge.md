# Casper via WhatsApp: Baileys Socket as Client Transport

> **Exploration.** WhatsApp is just another client. The same TS commands
> that an agent client issues through `mac_bridge.ts` can be issued from a
> WhatsApp message handler. Both clients call the same Casper API; the Rust
> FFI dylib (`libcasper.dylib` via `Deno.dlopen`) is the service.

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  WhatsApp User (phone / WhatsApp Web)                        │
│                                                              │
│  "screenshot"  "click Submit"  "type hello"  "hotkey cmd+s" │
└──────────────┬───────────────────────────────────────────────┘
               │  WhatsApp E2EE WebSocket
               ▼
┌──────────────────────────────────────────────────────────────┐
│  Deno Process                                                │
│                                                              │
│  ┌─────────────────────┐    ┌─────────────────────────┐     │
│  │  WhatsApp Client     │    │  Agent Client            │     │
│  │  (Baileys socket)    │    │  (TS agent loop)         │     │
│  │                      │    │                          │     │
│  │  messages.upsert     │    │  agent.step()            │     │
│  │    ↓                 │    │    ↓                     │     │
│  │  parse → command     │    │  plan → command          │     │
│  └──────────┬───────────┘    └────────────┬─────────────┘     │
│             │                             │                   │
│             ▼                             ▼                   │
│  ┌────────────────────────────────────────────────────────┐   │
│  │  Casper TS API  (mac_bridge.ts)                        │   │
│  │                                                        │   │
│  │  click()  typeText()  hotkey()  captureScreen()        │   │
│  │  listApplications()  frontmostApplication()  ...       │   │
│  └────────────────────────┬───────────────────────────────┘   │
│                           │  Deno.dlopen (C ABI)              │
│                           ▼                                   │
│  ┌────────────────────────────────────────────────────────┐   │
│  │  Rust: libcasper.dylib                                 │   │
│  │                                                        │   │
│  │  ffi.rs → input.rs, ax.rs, capture.rs, apps.rs, ...   │   │
│  └────────────────────────┬───────────────────────────────┘   │
└───────────────────────────┼───────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│  macOS (Accessibility, CoreGraphics, AppKit)                  │
└──────────────────────────────────────────────────────────────┘
```

The critical point: **there is no separate service process.** The Deno
process loads `libcasper.dylib` in-process. WhatsApp and agent clients
are two consumers of the same loaded library, sharing the same handle
table, the same TCC permissions, the same address space.

---

## Why This Works

The Casper FFI bridge (`casper-ffi-bridge.md`) defines a typed TS API in
`mac_bridge.ts` that wraps `Deno.dlopen` calls. The agent client
(`client.ts`) imports and calls these functions directly:

```typescript
// Agent client — from casper-ffi-bridge.md
import { captureScreen, click, typeText, hotkey } from "./mac_bridge.ts";

const screenshot = captureScreen();
click({ x: 500, y: 300 });
typeText("hello world");
hotkey("cmd+s");
```

A WhatsApp client does the exact same thing — it just gets its instructions
from a Baileys message handler instead of an agent loop:

```typescript
// WhatsApp client — same imports, same calls
import { captureScreen, click, typeText, hotkey } from "./mac_bridge.ts";

// Baileys message comes in: "screenshot"
const screenshot = captureScreen();
// → send screenshot bytes back via sock.sendMessage
```

Same API. Same dylib. Different input source.

---

## Why Baileys

[Baileys](https://github.com/WhiskeySockets/Baileys) (`@whiskeysockets/baileys`)
implements the WhatsApp Web socket protocol in TypeScript — no browser, no
Puppeteer, no official API key.

- **Deno-compatible** — works via `npm:@whiskeysockets/baileys` specifier.
  Runs in the same Deno process as the Casper FFI.
- **E2EE built-in** — Signal protocol handled transparently.
- **Rich message types** — can send/receive text, images (PNG/JPEG),
  documents, reactions, replies. Casper returns screenshots as PNG
  `Uint8Array` — send them directly as image messages.
- **Event-driven** — `sock.ev.on('messages.upsert', ...)` for incoming
  messages. No polling.
- **Session persistence** — `useMultiFileAuthState` saves credentials to
  disk. After the first QR scan, reconnections are automatic.

---

## Baileys Socket Essentials

### Setup and Connection

```typescript
import makeWASocket, {
  DisconnectReason,
  useMultiFileAuthState,
} from "npm:@whiskeysockets/baileys";
import { Boom } from "npm:@hapi/boom";
import pino from "npm:pino";

const logger = pino({ level: "warn" });

async function connectWhatsApp() {
  const { state, saveCreds } = await useMultiFileAuthState("auth_state");

  const sock = makeWASocket({
    auth: state,
    printQRInTerminal: true,
    logger,
  });

  sock.ev.on("creds.update", saveCreds);

  sock.ev.on("connection.update", ({ connection, lastDisconnect }) => {
    if (connection === "close") {
      const code = (lastDisconnect?.error as Boom)?.output?.statusCode;
      if (code !== DisconnectReason.loggedOut) {
        connectWhatsApp(); // reconnect
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
  if (type !== "notify") return;

  for (const msg of messages) {
    if (msg.key.fromMe) continue;

    const jid = msg.key.remoteJid!;
    const text =
      msg.message?.conversation ||
      msg.message?.extendedTextMessage?.text ||
      "";

    await handleCommand(sock, jid, text, msg);
  }
});
```

### Sending Replies

```typescript
// Text reply
await sock.sendMessage(jid, { text: "Done." });

// Image reply (PNG bytes from Casper captureScreen)
await sock.sendMessage(jid, {
  image: screenshotBuffer,
  caption: "Current screen",
});

// Quote-reply to the original command
await sock.sendMessage(jid, { text: "Clicked Submit." }, { quoted: msg });
```

---

## Command Protocol

Plain-text WhatsApp messages map to `mac_bridge.ts` function calls. First
word is the verb, rest is the argument.

### Command Table

| WhatsApp message | mac_bridge.ts call | Response |
|---|---|---|
| `screenshot` | `captureScreen()` | PNG image reply |
| `screenshot <windowId>` | `captureWindow(id)` | PNG image reply |
| `click 500 300` | `click({ x: 500, y: 300 })` | "Clicked (500, 300)." |
| `click right 500 300` | `rightClick({ x: 500, y: 300 })` | "Right-clicked (500, 300)." |
| `dblclick 500 300` | `doubleClick({ x: 500, y: 300 })` | "Double-clicked." |
| `type hello world` | `typeText("hello world")` | "Typed: hello world" |
| `hotkey cmd+s` | `hotkey("cmd+s")` | "Pressed cmd+s." |
| `scroll down` | `scrollDown(3)` | "Scrolled down." |
| `scroll up 5` | `scrollUp(5)` | "Scrolled up 5." |
| `apps` | `listApplications()` | Text list of running apps |
| `frontmost` | `frontmostApplication()` | App name + PID |
| `activate com.apple.Safari` | `activateApp(...)` | "Activated Safari." |
| `launch com.apple.TextEdit` | `launchApp(...)` | "Launched TextEdit." |
| `quit com.apple.TextEdit` | `quitApp(...)` | "Quit TextEdit." |
| `windows` | `getWindows(frontmostPid)` | Window list |
| `buttons` | `findElementsByRole(pid, "AXButton")` | Button list |
| `find AXTextField` | `findElementsByRole(pid, role)` | Element list |
| `element 500 300` | `elementAtPosition(pid, ...)` | Element info at point |
| `clipboard` | `readClipboard()` | Clipboard text |
| `clipboard set <text>` | `writeClipboard(text)` | "Clipboard set." |
| `permissions` | `checkPermissions()` | Permission status |
| `drag 100 100 500 500` | `drag(from, to)` | "Dragged." |
| `move 500 300` | `mouseMove(...)` | "Moved." |
| `help` | — | Command reference |

Every command maps 1:1 to a `mac_bridge.ts` export. The WhatsApp client
adds no capabilities beyond what the agent client already has.

---

## Implementation

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

### Command Handler

This imports `mac_bridge.ts` directly — the same module the agent client
uses. No IPC, no bridge socket, no separate process.

```typescript
// whatsapp_client.ts — WhatsApp command handler
//
// Same imports as client.ts in casper-ffi-bridge.md

import {
  checkPermissions,
  captureScreen,
  captureWindow,
  click,
  doubleClick,
  rightClick,
  mouseMove,
  typeText,
  hotkey,
  scroll,
  scrollDown,
  scrollUp,
  drag,
  listApplications,
  frontmostApplication,
  activateApp,
  launchApp,
  quitApp,
  readClipboard,
  writeClipboard,
  getWindows,
  findElementsByRole,
  elementAtPosition,
} from "./mac_bridge.ts";

import type { WASocket, WAMessage } from "npm:@whiskeysockets/baileys";

async function handleCommand(
  sock: WASocket,
  jid: string,
  text: string,
  msg: WAMessage,
): Promise<void> {
  const { verb, args, rawArgs } = parseCommand(text);

  switch (verb) {
    case "screenshot": {
      const windowId = parseInt(rawArgs[0], 10);
      const png = isNaN(windowId) ? captureScreen() : captureWindow(windowId);
      await sock.sendMessage(
        jid,
        { image: png, caption: isNaN(windowId) ? "Screen" : `Window ${windowId}` },
        { quoted: msg },
      );
      break;
    }

    case "click": {
      const [xStr, yStr] = rawArgs;
      const x = parseFloat(xStr);
      const y = parseFloat(yStr);
      if (isNaN(x) || isNaN(y)) {
        await sock.sendMessage(jid, { text: "Usage: click <x> <y>" }, { quoted: msg });
        break;
      }
      click({ x, y });
      await sock.sendMessage(jid, { text: `Clicked (${x}, ${y}).` }, { quoted: msg });
      break;
    }

    case "dblclick": {
      const [xStr, yStr] = rawArgs;
      const x = parseFloat(xStr);
      const y = parseFloat(yStr);
      if (isNaN(x) || isNaN(y)) {
        await sock.sendMessage(jid, { text: "Usage: dblclick <x> <y>" }, { quoted: msg });
        break;
      }
      doubleClick({ x, y });
      await sock.sendMessage(jid, { text: `Double-clicked (${x}, ${y}).` }, { quoted: msg });
      break;
    }

    case "rightclick": {
      const [xStr, yStr] = rawArgs;
      const x = parseFloat(xStr);
      const y = parseFloat(yStr);
      if (isNaN(x) || isNaN(y)) {
        await sock.sendMessage(jid, { text: "Usage: rightclick <x> <y>" }, { quoted: msg });
        break;
      }
      rightClick({ x, y });
      await sock.sendMessage(jid, { text: `Right-clicked (${x}, ${y}).` }, { quoted: msg });
      break;
    }

    case "type": {
      if (!args) {
        await sock.sendMessage(jid, { text: "Usage: type <text>" }, { quoted: msg });
        break;
      }
      typeText(args);
      await sock.sendMessage(jid, { text: `Typed: ${args}` }, { quoted: msg });
      break;
    }

    case "hotkey": {
      if (!args) {
        await sock.sendMessage(jid, { text: "Usage: hotkey <keys> (e.g. cmd+s)" }, { quoted: msg });
        break;
      }
      hotkey(args);
      await sock.sendMessage(jid, { text: `Pressed ${args}.` }, { quoted: msg });
      break;
    }

    case "scroll": {
      const dir = rawArgs[0]?.toLowerCase() ?? "down";
      const amount = parseInt(rawArgs[1] ?? "3", 10);
      if (dir === "up") scrollUp(amount);
      else if (dir === "down") scrollDown(amount);
      else if (dir === "left") scroll(amount, 0);
      else if (dir === "right") scroll(-amount, 0);
      await sock.sendMessage(jid, { text: `Scrolled ${dir} ${amount}.` }, { quoted: msg });
      break;
    }

    case "move": {
      const x = parseFloat(rawArgs[0]);
      const y = parseFloat(rawArgs[1]);
      if (isNaN(x) || isNaN(y)) {
        await sock.sendMessage(jid, { text: "Usage: move <x> <y>" }, { quoted: msg });
        break;
      }
      mouseMove({ x, y });
      await sock.sendMessage(jid, { text: `Moved to (${x}, ${y}).` }, { quoted: msg });
      break;
    }

    case "drag": {
      const coords = rawArgs.map(parseFloat);
      if (coords.length < 4 || coords.some(isNaN)) {
        await sock.sendMessage(jid, { text: "Usage: drag <x1> <y1> <x2> <y2>" }, { quoted: msg });
        break;
      }
      drag({ x: coords[0], y: coords[1] }, { x: coords[2], y: coords[3] });
      await sock.sendMessage(jid, { text: `Dragged (${coords[0]},${coords[1]}) → (${coords[2]},${coords[3]}).` }, { quoted: msg });
      break;
    }

    case "apps": {
      const apps = listApplications();
      const list = apps
        .slice(0, 20)
        .map((a) => `${a.name ?? a.bundle_id} (pid ${a.pid})`)
        .join("\n");
      await sock.sendMessage(jid, { text: list || "No apps." }, { quoted: msg });
      break;
    }

    case "frontmost": {
      const app = frontmostApplication();
      const reply = app
        ? `${app.name} (${app.bundle_id}, pid ${app.pid})`
        : "No frontmost app.";
      await sock.sendMessage(jid, { text: reply }, { quoted: msg });
      break;
    }

    case "activate": {
      if (!args) {
        await sock.sendMessage(jid, { text: "Usage: activate <bundleId>" }, { quoted: msg });
        break;
      }
      activateApp(args);
      await sock.sendMessage(jid, { text: `Activated ${args}.` }, { quoted: msg });
      break;
    }

    case "launch": {
      if (!args) {
        await sock.sendMessage(jid, { text: "Usage: launch <bundleId>" }, { quoted: msg });
        break;
      }
      launchApp(args);
      await sock.sendMessage(jid, { text: `Launched ${args}.` }, { quoted: msg });
      break;
    }

    case "quit": {
      if (!args) {
        await sock.sendMessage(jid, { text: "Usage: quit <bundleId>" }, { quoted: msg });
        break;
      }
      const force = rawArgs.includes("--force");
      const bundleId = rawArgs.filter((a) => a !== "--force").join(" ");
      quitApp(bundleId, force);
      await sock.sendMessage(jid, { text: `Quit ${bundleId}.` }, { quoted: msg });
      break;
    }

    case "windows": {
      const app = frontmostApplication();
      if (!app) {
        await sock.sendMessage(jid, { text: "No frontmost app." }, { quoted: msg });
        break;
      }
      const wins = getWindows(app.pid);
      const list = wins
        .map((w) => `"${w.title}" (${w.width}x${w.height} at ${w.x},${w.y})`)
        .join("\n");
      await sock.sendMessage(jid, { text: list || "No windows." }, { quoted: msg });
      break;
    }

    case "buttons": {
      const app = frontmostApplication();
      if (!app) {
        await sock.sendMessage(jid, { text: "No frontmost app." }, { quoted: msg });
        break;
      }
      const buttons = findElementsByRole(app.pid, "AXButton", 8);
      const list = buttons
        .slice(0, 15)
        .map((b) => `"${b.title ?? b.label ?? "?"}" at (${b.x}, ${b.y})`)
        .join("\n");
      await sock.sendMessage(jid, { text: list || "No buttons." }, { quoted: msg });
      break;
    }

    case "find": {
      const role = rawArgs[0];
      if (!role) {
        await sock.sendMessage(jid, { text: "Usage: find <AXRole>" }, { quoted: msg });
        break;
      }
      const app = frontmostApplication();
      if (!app) {
        await sock.sendMessage(jid, { text: "No frontmost app." }, { quoted: msg });
        break;
      }
      const elements = findElementsByRole(app.pid, role, 8);
      const list = elements
        .slice(0, 15)
        .map((e) => `${e.role}: "${e.title ?? e.label ?? "?"}" at (${e.x}, ${e.y})`)
        .join("\n");
      await sock.sendMessage(jid, { text: list || `No ${role} elements.` }, { quoted: msg });
      break;
    }

    case "element": {
      const x = parseFloat(rawArgs[0]);
      const y = parseFloat(rawArgs[1]);
      if (isNaN(x) || isNaN(y)) {
        await sock.sendMessage(jid, { text: "Usage: element <x> <y>" }, { quoted: msg });
        break;
      }
      const app = frontmostApplication();
      if (!app) {
        await sock.sendMessage(jid, { text: "No frontmost app." }, { quoted: msg });
        break;
      }
      const el = elementAtPosition(app.pid, { x, y });
      if (el) {
        const info = `${el.role}: "${el.title ?? el.label ?? "?"}" value="${el.value ?? ""}" at (${el.x},${el.y} ${el.width}x${el.height})`;
        await sock.sendMessage(jid, { text: info }, { quoted: msg });
      } else {
        await sock.sendMessage(jid, { text: "No element at that position." }, { quoted: msg });
      }
      break;
    }

    case "clipboard": {
      if (rawArgs[0] === "set" && rawArgs.length > 1) {
        const clipText = rawArgs.slice(1).join(" ");
        writeClipboard(clipText);
        await sock.sendMessage(jid, { text: `Clipboard set.` }, { quoted: msg });
      } else {
        const content = readClipboard();
        await sock.sendMessage(jid, { text: content ?? "(empty)" }, { quoted: msg });
      }
      break;
    }

    case "permissions": {
      const perms = checkPermissions();
      await sock.sendMessage(
        jid,
        { text: `Accessibility: ${perms.accessibility}\nScreen Recording: ${perms.screen_recording}` },
        { quoted: msg },
      );
      break;
    }

    case "help": {
      const help = [
        "*Casper Commands*",
        "",
        "screenshot — capture screen",
        "screenshot <windowId> — capture window",
        "click <x> <y> — click at coordinates",
        "dblclick <x> <y> — double-click",
        "rightclick <x> <y> — right-click",
        "type <text> — type text",
        "hotkey <keys> — press key combo (cmd+s)",
        "scroll <dir> [amount] — up/down/left/right",
        "move <x> <y> — move mouse",
        "drag <x1> <y1> <x2> <y2> — drag",
        "apps — list running apps",
        "frontmost — frontmost app",
        "activate <bundleId> — activate app",
        "launch <bundleId> — launch app",
        "quit <bundleId> — quit app",
        "windows — list frontmost app windows",
        "buttons — list buttons in frontmost app",
        "find <AXRole> — find elements by role",
        "element <x> <y> — inspect element at point",
        "clipboard — read clipboard",
        "clipboard set <text> — write clipboard",
        "permissions — check TCC status",
      ].join("\n");
      await sock.sendMessage(jid, { text: help }, { quoted: msg });
      break;
    }

    default:
      break;
  }
}
```

---

## Entry Point

```typescript
// casper-whatsapp.ts — WhatsApp client for Casper
//
// deno run --allow-ffi --allow-read --allow-write --allow-net casper-whatsapp.ts

import {
  checkPermissions,
  close as closeCasper,
} from "./mac_bridge.ts";

// --- Access control ---

const ALLOWED_JIDS = new Set([
  // Add your phone number JID here
  // "12345678901@s.whatsapp.net",
]);

function isAuthorized(msg: WAMessage): boolean {
  const sender = msg.key.participant ?? msg.key.remoteJid!;
  return ALLOWED_JIDS.has(sender);
}

// --- Main ---

async function main() {
  // 1. Verify Casper FFI is loaded and permissions are granted
  const perms = checkPermissions();
  console.log("Permissions:", perms);
  if (!perms.accessibility || !perms.screen_recording) {
    console.error("Missing TCC permissions. Grant in System Settings.");
    Deno.exit(1);
  }

  // 2. Connect to WhatsApp
  const sock = await connectWhatsApp();

  // 3. Route incoming messages to the Casper command handler
  sock.ev.on("messages.upsert", async ({ messages, type }) => {
    if (type !== "notify") return;
    for (const msg of messages) {
      if (msg.key.fromMe) continue;
      if (!isAuthorized(msg)) continue;

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

  // Cleanup on exit
  globalThis.addEventListener("unload", () => closeCasper());
}

main();
```

### Running It

```bash
# Build the Rust dylib first
cargo build --release
# → target/release/libcasper.dylib

# Run the WhatsApp client
deno run --allow-ffi --allow-read --allow-write --allow-net casper-whatsapp.ts

# First run: scan the QR code in terminal with WhatsApp
# Subsequent runs: auto-reconnects from auth_state/
```

---

## How It Compares to the Agent Client

Both clients are thin shells over the same API. The difference is where
instructions come from:

```
┌─────────────────────────────────────────────────────────────────┐
│                        mac_bridge.ts                            │
│  click()  typeText()  hotkey()  captureScreen()  ...           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Agent Client (client.ts)     WhatsApp Client (this doc)      │
│   ─────────────────────────    ──────────────────────────      │
│   Instructions from:           Instructions from:               │
│     LLM agent loop               WhatsApp messages             │
│     (plan → tool call)           (text parse → function call)  │
│                                                                 │
│   Output to:                   Output to:                       │
│     Agent context                WhatsApp reply                 │
│     (JSON / Uint8Array)          (text / image message)         │
│                                                                 │
│   Runs as:                     Runs as:                         │
│     Script or REPL               Long-lived daemon              │
│     (execute and exit)           (listen for messages)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

There is no protocol translation. No IPC. No serialization beyond what
`mac_bridge.ts` already does for the FFI boundary. A WhatsApp "click 500 300"
call resolves to the exact same `symbols.mac_click(500.0, 300.0, 0, 1)` C
function invocation as `click({ x: 500, y: 300 })` from the agent client.

---

## Entity API (Higher-Level)

The command table above maps to the low-level `mac_bridge.ts` API. But the
Casper tech design defines a richer entity model (`App`, `Window`, `Element`,
`Snapshot`). The WhatsApp client can use that too:

```typescript
// Using Casper entity API from WhatsApp
import { App, Screen, Keyboard, Clipboard } from "./casper/mod.ts";

case "safari": {
  const safari = await App.find("Safari");
  await safari.activate();
  const win = await safari.focusedWindow();
  const snap = await win.snapshot();
  // Send the snapshot text as a WhatsApp message
  await sock.sendMessage(jid, { text: snap.text }, { quoted: msg });
  break;
}

case "snap": {
  const app = await App.frontmost();
  const win = await app.focusedWindow();
  using snap = await win.snapshot();
  await sock.sendMessage(jid, { text: snap.text }, { quoted: msg });
  break;
}

case "snapclick": {
  // "snapclick 5" — click ref 5 in the most recent snapshot
  const ref = parseInt(rawArgs[0], 10);
  // (would need to track the last snapshot in session state)
  await lastSnapshot.click(ref);
  await sock.sendMessage(jid, { text: `Clicked ref ${ref}.` }, { quoted: msg });
  break;
}
```

This is where WhatsApp becomes a lightweight remote GUI inspector:
1. Send `snap` → get the AX tree with `[ref=N]` tags
2. Read the text on your phone, pick a ref
3. Send `snapclick 5` → click that element

The snapshot text is compact enough (~200-500 tokens) to read comfortably
in a WhatsApp message.

---

## Latency Budget

```
WhatsApp delivery:     ~200-500ms (E2EE + relay)
Baileys processing:    ~5-10ms
Command parsing:       <1ms
Casper FFI call:       <1ms (direct C function invocation)
macOS operation:       ~10-100ms (capture is slowest)
Response formatting:   <1ms
WhatsApp reply:        ~200-500ms
──────────────────────────────────
Total round-trip:      ~400ms-1.1s
```

The FFI call is negligible. WhatsApp network hops dominate entirely.

---

## Security

### Access Control

Without a JID allowlist, anyone who messages the linked WhatsApp number
gets full macOS control. The `ALLOWED_JIDS` set is the primary gate.

For group chats, `msg.key.participant` is the actual sender. For 1:1 chats,
`msg.key.remoteJid` is the sender. Always check the sender, not the chat.

### Session Keys

Baileys session state (`auth_state/`) contains WhatsApp Signal keys. Treat
this directory like an SSH private key. Anyone with these files can
impersonate your WhatsApp account.

### TCC Permissions

The Deno process needs Accessibility and Screen Recording grants. These are
the same grants the agent client needs — no additional permissions for the
WhatsApp transport.

---

## File Layout

```
casper/
├── deno/
│   ├── mod.ts                  # Deno.dlopen + symbol defs
│   ├── ffi_helpers.ts          # pointer/buffer utilities
│   ├── mac_bridge.ts           # typed TS API (shared by all clients)
│   ├── client.ts               # agent client (from casper-ffi-bridge.md)
│   ├── whatsapp_client.ts      # WhatsApp client (this doc)
│   ├── casper-whatsapp.ts      # entry point (connect + route)
│   └── auth_state/             # Baileys session (gitignored)
└── src/                        # Rust
    ├── ffi.rs
    ├── input.rs
    ├── ax.rs
    ├── capture.rs
    ├── apps.rs
    ├── clipboard.rs
    └── permissions.rs
```

Both `client.ts` and `whatsapp_client.ts` import from the same
`mac_bridge.ts`. The Rust crate is built once and loaded once.
