# Casper via WhatsApp: Baileys Socket as Client Transport

> **Exploration.** WhatsApp is just another client. The same Casper entity
> API (`App`, `Window`, `Element`, `Keyboard`, `Mouse`, `Screen`, etc.)
> that an agent client uses can be called from a WhatsApp message handler.
> Both clients import from `casper/mod.ts`; the Rust FFI dylib
> (`libcasper.dylib` via `Deno.dlopen`) is the service underneath.

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
│  │  Casper Entity API  (casper/mod.ts)                    │   │
│  │                                                        │   │
│  │  App  Window  Element  Snapshot                        │   │
│  │  Keyboard  Mouse  Screen  Clipboard  Permissions       │   │
│  └────────────────────────┬───────────────────────────────┘   │
│                           │  FFI handles + Deno.dlopen        │
│                           ▼                                   │
│  ┌────────────────────────────────────────────────────────┐   │
│  │  Rust: libcasper.dylib                                 │   │
│  │                                                        │   │
│  │  ffi.rs → handles.rs, input.rs, ax.rs, capture.rs, ... │   │
│  └────────────────────────┬───────────────────────────────┘   │
└───────────────────────────┼───────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│  macOS (Accessibility, CoreGraphics, AppKit)                  │
└──────────────────────────────────────────────────────────────┘
```

There is no separate service process. The Deno process loads
`libcasper.dylib` in-process. WhatsApp and agent clients are two consumers
of the same loaded library, sharing the same handle table, the same TCC
permissions, the same address space.

---

## Why This Works

The Casper tech design defines an entity API where everything is an object
with methods — `App`, `Window`, `Element`, `Keyboard`, `Mouse`, `Screen`.
The agent client uses it directly:

```typescript
// Agent client
import { App, Keyboard, Screen } from "./casper/mod.ts";

const safari = await App.launch("com.apple.Safari");
const win = (await safari.windows())[0];
const buttons = await win.findAll({ role: "AXButton" });
await buttons[0].click();
await Keyboard.type("hello world");
const png = await Screen.capture();
```

A WhatsApp client does the exact same thing — it just gets its instructions
from a Baileys message handler instead of an agent loop:

```typescript
// WhatsApp client — same imports, same entity calls
import { App, Keyboard, Mouse, Screen, Clipboard, Permissions } from "./casper/mod.ts";

// Baileys message comes in: "screenshot"
const png = await Screen.capture();
// → send png bytes back via sock.sendMessage

// Baileys message comes in: "click 500 300"
await Mouse.click({ x: 500, y: 300 });

// Baileys message comes in: "snap" → "snapclick 5"
const win = await (await App.frontmost()).focusedWindow();
const snap = await win.snapshot();
// → send snap.text as WhatsApp message
// user replies "snapclick 5"
await snap.click(5);
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
  documents, reactions, replies. Casper entities return screenshots as PNG
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

// Image reply (PNG bytes from Screen.capture or Window.capture)
await sock.sendMessage(jid, {
  image: screenshotBuffer,
  caption: "Current screen",
});

// Quote-reply to the original command
await sock.sendMessage(jid, { text: "Clicked Submit." }, { quoted: msg });
```

---

## Command Protocol

Plain-text WhatsApp messages map to Casper entity API calls. First word is
the verb, rest is the argument.

### Command Table

| WhatsApp message | Casper entity API call | Response |
|---|---|---|
| `screenshot` | `Screen.capture()` | PNG image reply |
| `screenshot window` | `win.capture()` | PNG image reply |
| `snap` | `win.snapshot()` | Snapshot text with `[ref=N]` tags |
| `snapclick 5` | `snapshot.click(5)` | "Clicked ref 5." |
| `snaptype 3 hello` | `snapshot.type(3, "hello")` | "Typed into ref 3." |
| `click 500 300` | `Mouse.click({ x: 500, y: 300 })` | "Clicked (500, 300)." |
| `dblclick 500 300` | `Mouse.doubleClick({ x: 500, y: 300 })` | "Double-clicked." |
| `rightclick 500 300` | `Mouse.rightClick({ x: 500, y: 300 })` | "Right-clicked." |
| `move 500 300` | `Mouse.move({ x: 500, y: 300 })` | "Moved." |
| `drag 100 100 500 500` | `Mouse.drag(from, to)` | "Dragged." |
| `scroll down` | `Mouse.scroll("down", 3)` | "Scrolled down." |
| `scroll up 5` | `Mouse.scroll("up", 5)` | "Scrolled up 5." |
| `type hello world` | `Keyboard.type("hello world")` | "Typed: hello world" |
| `hotkey cmd+s` | `Keyboard.hotkey("cmd+s")` | "Pressed cmd+s." |
| `press return` | `Keyboard.press("return")` | "Pressed return." |
| `apps` | `App.all()` | Text list of running apps |
| `frontmost` | `App.frontmost()` | App name + PID |
| `find Safari` | `App.find("Safari")` | App info |
| `launch com.apple.TextEdit` | `App.launch(bundleId)` | "Launched TextEdit." |
| `activate com.apple.Safari` | `app.activate()` | "Activated Safari." |
| `quit com.apple.Safari` | `app.quit()` | "Quit Safari." |
| `windows` | `app.windows()` | Window list |
| `buttons` | `win.findAll({ role: "AXButton" })` | Button list |
| `fields` | `win.findAll({ role: "AXTextField" })` | Text field list |
| `findall <role>` | `win.findAll({ role })` | Element list |
| `clipboard` | `Clipboard.read()` | Clipboard text |
| `clipboard set <text>` | `Clipboard.write(text)` | "Clipboard set." |
| `permissions` | `Permissions.check()` | Permission status |
| `help` | — | Command reference |

Every command maps to entity methods from `casper/mod.ts`. The WhatsApp
client calls the same typed API the agent client uses.

---

## Session State

Unlike the agent client (which runs a script and exits), the WhatsApp client
is a long-lived daemon with conversational interactions. It needs per-chat
session state to support snapshot workflows and keep references to live
entities.

```typescript
import type { Snapshot } from "./casper/mod.ts";

interface ChatSession {
  snapshot: Snapshot | null;   // last snapshot for this chat (holds live Element handles)
  app: App | null;             // last-referenced app
}

const sessions = new Map<string, ChatSession>();

function getSession(jid: string): ChatSession {
  let session = sessions.get(jid);
  if (!session) {
    session = { snapshot: null, app: null };
    sessions.set(jid, session);
  }
  return session;
}

function disposeSession(jid: string): void {
  const session = sessions.get(jid);
  if (session?.snapshot) {
    session.snapshot.dispose();
  }
  sessions.delete(jid);
}
```

When a user sends `snap`, the handler creates a snapshot and stores it.
When they send `snapclick 5`, it looks up that snapshot's ref map.

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

This imports from `casper/mod.ts` — the entity API from the tech design.

```typescript
// whatsapp_client.ts — WhatsApp command handler

import {
  App,
  Window,
  Element,
  Keyboard,
  Mouse,
  Screen,
  Clipboard,
  Permissions,
  shutdown,
  type ElementQuery,
} from "./casper/mod.ts";

import type { WASocket, WAMessage } from "npm:@whiskeysockets/baileys";

async function handleCommand(
  sock: WASocket,
  jid: string,
  text: string,
  msg: WAMessage,
): Promise<void> {
  const { verb, args, rawArgs } = parseCommand(text);
  const session = getSession(jid);

  switch (verb) {
    // --- Screen & Window Capture ---

    case "screenshot": {
      if (args === "window") {
        const app = await App.frontmost();
        const win = await app.focusedWindow();
        const png = await win.capture();
        await sock.sendMessage(jid, { image: png, caption: `${app.name} — ${win.title}` }, { quoted: msg });
      } else {
        const png = await Screen.capture();
        await sock.sendMessage(jid, { image: png, caption: "Screen" }, { quoted: msg });
      }
      break;
    }

    // --- Snapshots (the core workflow for WhatsApp) ---

    case "snap": {
      // Dispose previous snapshot if any
      if (session.snapshot) session.snapshot.dispose();

      const app = await App.frontmost();
      const win = await app.focusedWindow();
      session.snapshot = await win.snapshot();
      session.app = app;

      const header = `*${app.name}* — ${win.title}\n\n`;
      await sock.sendMessage(jid, { text: header + session.snapshot.text }, { quoted: msg });
      break;
    }

    case "snapclick": {
      const ref = parseInt(rawArgs[0], 10);
      if (isNaN(ref) || !session.snapshot) {
        await sock.sendMessage(jid, { text: !session.snapshot ? "No snapshot. Send snap first." : "Usage: snapclick <ref>" }, { quoted: msg });
        break;
      }
      await session.snapshot.click(ref);
      await sock.sendMessage(jid, { text: `Clicked ref ${ref}.` }, { quoted: msg });
      break;
    }

    case "snaptype": {
      const ref = parseInt(rawArgs[0], 10);
      const typeText = rawArgs.slice(1).join(" ");
      if (isNaN(ref) || !typeText || !session.snapshot) {
        await sock.sendMessage(jid, { text: !session.snapshot ? "No snapshot. Send snap first." : "Usage: snaptype <ref> <text>" }, { quoted: msg });
        break;
      }
      await session.snapshot.type(ref, typeText);
      await sock.sendMessage(jid, { text: `Typed into ref ${ref}.` }, { quoted: msg });
      break;
    }

    case "snapref": {
      const ref = parseInt(rawArgs[0], 10);
      if (isNaN(ref) || !session.snapshot) {
        await sock.sendMessage(jid, { text: !session.snapshot ? "No snapshot. Send snap first." : "Usage: snapref <ref>" }, { quoted: msg });
        break;
      }
      const el = session.snapshot.get(ref);
      const info = `ref ${ref}: ${el.role} "${el.title ?? el.label ?? "?"}" value="${el.value ?? ""}"`;
      await sock.sendMessage(jid, { text: info }, { quoted: msg });
      break;
    }

    // --- Mouse ---

    case "click": {
      const [xStr, yStr] = rawArgs;
      const x = parseFloat(xStr);
      const y = parseFloat(yStr);
      if (isNaN(x) || isNaN(y)) {
        await sock.sendMessage(jid, { text: "Usage: click <x> <y>" }, { quoted: msg });
        break;
      }
      await Mouse.click({ x, y });
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
      await Mouse.doubleClick({ x, y });
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
      await Mouse.rightClick({ x, y });
      await sock.sendMessage(jid, { text: `Right-clicked (${x}, ${y}).` }, { quoted: msg });
      break;
    }

    case "move": {
      const x = parseFloat(rawArgs[0]);
      const y = parseFloat(rawArgs[1]);
      if (isNaN(x) || isNaN(y)) {
        await sock.sendMessage(jid, { text: "Usage: move <x> <y>" }, { quoted: msg });
        break;
      }
      await Mouse.move({ x, y });
      await sock.sendMessage(jid, { text: `Moved to (${x}, ${y}).` }, { quoted: msg });
      break;
    }

    case "drag": {
      const coords = rawArgs.map(parseFloat);
      if (coords.length < 4 || coords.some(isNaN)) {
        await sock.sendMessage(jid, { text: "Usage: drag <x1> <y1> <x2> <y2>" }, { quoted: msg });
        break;
      }
      await Mouse.drag({ x: coords[0], y: coords[1] }, { x: coords[2], y: coords[3] });
      await sock.sendMessage(jid, { text: `Dragged.` }, { quoted: msg });
      break;
    }

    case "scroll": {
      const dir = rawArgs[0]?.toLowerCase() ?? "down";
      const amount = parseInt(rawArgs[1] ?? "3", 10);
      await Mouse.scroll(dir as "up" | "down" | "left" | "right", amount);
      await sock.sendMessage(jid, { text: `Scrolled ${dir} ${amount}.` }, { quoted: msg });
      break;
    }

    // --- Keyboard ---

    case "type": {
      if (!args) {
        await sock.sendMessage(jid, { text: "Usage: type <text>" }, { quoted: msg });
        break;
      }
      await Keyboard.type(args);
      await sock.sendMessage(jid, { text: `Typed: ${args}` }, { quoted: msg });
      break;
    }

    case "hotkey": {
      if (!args) {
        await sock.sendMessage(jid, { text: "Usage: hotkey <keys> (e.g. cmd+s)" }, { quoted: msg });
        break;
      }
      await Keyboard.hotkey(args);
      await sock.sendMessage(jid, { text: `Pressed ${args}.` }, { quoted: msg });
      break;
    }

    case "press": {
      if (!args) {
        await sock.sendMessage(jid, { text: "Usage: press <key> (e.g. return, tab, escape)" }, { quoted: msg });
        break;
      }
      await Keyboard.press(args);
      await sock.sendMessage(jid, { text: `Pressed ${args}.` }, { quoted: msg });
      break;
    }

    // --- App ---

    case "apps": {
      const apps = await App.all();
      const list = apps
        .slice(0, 20)
        .map((a) => `${a.name} (${a.bundleId}, pid ${a.pid})`)
        .join("\n");
      await sock.sendMessage(jid, { text: list || "No apps." }, { quoted: msg });
      break;
    }

    case "frontmost": {
      const app = await App.frontmost();
      session.app = app;
      await sock.sendMessage(jid, { text: `${app.name} (${app.bundleId}, pid ${app.pid})` }, { quoted: msg });
      break;
    }

    case "find": {
      if (!args) {
        await sock.sendMessage(jid, { text: "Usage: find <name or bundleId>" }, { quoted: msg });
        break;
      }
      const app = await App.find(args);
      session.app = app;
      await sock.sendMessage(jid, { text: `${app.name} (${app.bundleId}, pid ${app.pid})` }, { quoted: msg });
      break;
    }

    case "launch": {
      if (!args) {
        await sock.sendMessage(jid, { text: "Usage: launch <bundleId>" }, { quoted: msg });
        break;
      }
      const app = await App.launch(args);
      session.app = app;
      await sock.sendMessage(jid, { text: `Launched ${app.name}.` }, { quoted: msg });
      break;
    }

    case "activate": {
      if (!session.app && !args) {
        await sock.sendMessage(jid, { text: "Usage: activate <name or bundleId>" }, { quoted: msg });
        break;
      }
      const app = args ? await App.find(args) : session.app!;
      await app.activate();
      session.app = app;
      await sock.sendMessage(jid, { text: `Activated ${app.name}.` }, { quoted: msg });
      break;
    }

    case "quit": {
      if (!session.app && !args) {
        await sock.sendMessage(jid, { text: "Usage: quit <name or bundleId>" }, { quoted: msg });
        break;
      }
      const app = args ? await App.find(args) : session.app!;
      const force = rawArgs.includes("--force");
      await app.quit(force);
      await sock.sendMessage(jid, { text: `Quit ${app.name}.` }, { quoted: msg });
      break;
    }

    // --- Window ---

    case "windows": {
      const app = session.app ?? await App.frontmost();
      const wins = await app.windows();
      const list = wins
        .map((w) => `"${w.title}" (id ${w.id})`)
        .join("\n");
      await sock.sendMessage(jid, { text: list || "No windows." }, { quoted: msg });
      break;
    }

    case "focus": {
      const app = session.app ?? await App.frontmost();
      const win = await app.focusedWindow();
      await win.focus();
      await sock.sendMessage(jid, { text: `Focused "${win.title}".` }, { quoted: msg });
      break;
    }

    // --- Element queries on the focused window ---

    case "buttons": {
      const app = session.app ?? await App.frontmost();
      const win = await app.focusedWindow();
      const buttons = await win.findAll({ role: "AXButton" });
      const list = buttons
        .slice(0, 15)
        .map((b) => `"${b.title ?? b.label ?? "?"}" at (${b.bounds.x}, ${b.bounds.y})`)
        .join("\n");
      await sock.sendMessage(jid, { text: list || "No buttons." }, { quoted: msg });
      break;
    }

    case "fields": {
      const app = session.app ?? await App.frontmost();
      const win = await app.focusedWindow();
      const fields = await win.findAll({ role: "AXTextField" });
      const list = fields
        .slice(0, 15)
        .map((f) => `"${f.title ?? f.label ?? "?"}" value="${f.value ?? ""}" at (${f.bounds.x}, ${f.bounds.y})`)
        .join("\n");
      await sock.sendMessage(jid, { text: list || "No text fields." }, { quoted: msg });
      break;
    }

    case "findall": {
      const role = rawArgs[0];
      if (!role) {
        await sock.sendMessage(jid, { text: "Usage: findall <AXRole>" }, { quoted: msg });
        break;
      }
      const app = session.app ?? await App.frontmost();
      const win = await app.focusedWindow();
      const elements = await win.findAll({ role });
      const list = elements
        .slice(0, 15)
        .map((e) => `${e.role}: "${e.title ?? e.label ?? "?"}" at (${e.bounds.x}, ${e.bounds.y})`)
        .join("\n");
      await sock.sendMessage(jid, { text: list || `No ${role} elements.` }, { quoted: msg });
      break;
    }

    // --- Clipboard ---

    case "clipboard": {
      if (rawArgs[0] === "set" && rawArgs.length > 1) {
        const clipText = rawArgs.slice(1).join(" ");
        Clipboard.write(clipText);
        await sock.sendMessage(jid, { text: "Clipboard set." }, { quoted: msg });
      } else {
        const content = Clipboard.read();
        await sock.sendMessage(jid, { text: content ?? "(empty)" }, { quoted: msg });
      }
      break;
    }

    // --- Permissions ---

    case "permissions": {
      const perms = Permissions.check();
      await sock.sendMessage(
        jid,
        { text: `Accessibility: ${perms.accessibility}\nScreen Recording: ${perms.screenRecording}` },
        { quoted: msg },
      );
      break;
    }

    // --- Help ---

    case "help": {
      const help = [
        "*Casper Commands*",
        "",
        "_Snapshot (inspect → act)_",
        "snap — snapshot frontmost window AX tree",
        "snapclick <ref> — click element by ref",
        "snaptype <ref> <text> — type into element",
        "snapref <ref> — inspect a ref's properties",
        "",
        "_Screen_",
        "screenshot — capture screen",
        "screenshot window — capture frontmost window",
        "",
        "_Mouse_",
        "click <x> <y> — click at coordinates",
        "dblclick <x> <y> — double-click",
        "rightclick <x> <y> — right-click",
        "move <x> <y> — move mouse",
        "drag <x1> <y1> <x2> <y2> — drag",
        "scroll <dir> [amount] — up/down/left/right",
        "",
        "_Keyboard_",
        "type <text> — type text",
        "hotkey <keys> — key combo (cmd+s)",
        "press <key> — single key (return, tab)",
        "",
        "_App_",
        "apps — list running apps",
        "frontmost — frontmost app",
        "find <name> — find app by name",
        "launch <bundleId> — launch app",
        "activate [name] — activate app",
        "quit [name] — quit app",
        "",
        "_Window & Elements_",
        "windows — list windows",
        "buttons — list buttons",
        "fields — list text fields",
        "findall <AXRole> — find elements by role",
        "",
        "_Other_",
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

import { Permissions, shutdown } from "./casper/mod.ts";

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
  // 1. Verify Casper permissions
  const perms = Permissions.check();
  console.log("Permissions:", perms);
  if (!perms.accessibility || !perms.screenRecording) {
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
  globalThis.addEventListener("unload", () => {
    // Dispose all session snapshots
    for (const [jid] of sessions) disposeSession(jid);
    shutdown();
  });
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

## The Snapshot Workflow

Snapshots are the key interaction pattern over WhatsApp. Coordinates require
knowing screen positions in advance; snapshots let you *see* the UI and
*act* on labeled elements from your phone.

```
You:     snap
Casper:  *Safari* — Google
         window "Google" [ref=1]
           group "Toolbar"
             button "Back" [ref=2]
             button "Forward" [ref=3]
             textfield "Address" value="https://google.com" [ref=4]
             button "Reload" [ref=5]
           group "Content"
             textfield "Search" [ref=6]
             button "Google Search" [ref=7]
             button "I'm Feeling Lucky" [ref=8]

You:     snapclick 6
Casper:  Clicked ref 6.

You:     type weather today
Casper:  Typed: weather today

You:     press return
Casper:  Pressed return.

You:     snap
Casper:  *Safari* — weather today - Google Search
         window "weather today - Google Search" [ref=1]
           ...
```

This is the same `Snapshot` entity from the tech design — `.text` gives the
compact AX tree with `[ref=N]` tags, `.click(ref)` acts on the live
`Element` handle behind that ref. The WhatsApp client just renders
`.text` as a chat message instead of feeding it to an LLM.

---

## How It Compares to the Agent Client

Both clients are thin shells over the same entity API. The difference is
where instructions come from and where output goes:

```
┌─────────────────────────────────────────────────────────────────┐
│  Casper Entity API  (casper/mod.ts)                             │
│  App  Window  Element  Snapshot  Keyboard  Mouse  Screen  ...  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Agent Client                   WhatsApp Client               │
│   ────────────                   ───────────────               │
│   Instructions from:             Instructions from:             │
│     LLM agent loop                 WhatsApp messages           │
│     (plan → entity method call)    (text parse → entity call)  │
│                                                                 │
│   Output to:                     Output to:                     │
│     Agent context                  WhatsApp reply              │
│     (Snapshot.text → LLM)          (Snapshot.text → message)   │
│     (Uint8Array → tool result)     (Uint8Array → image reply)  │
│                                                                 │
│   Runs as:                       Runs as:                       │
│     Script or REPL                 Long-lived daemon            │
│     (execute and exit)             (listen for messages)        │
│                                                                 │
│   State:                         State:                         │
│     `using snap = ...`             per-chat session map         │
│     (scoped to block)             (snapshot + app reference)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

The only real difference is session management. The agent client uses
`using` blocks for snapshot lifetime. The WhatsApp client tracks per-chat
sessions because the user's interaction is conversational — `snap`, then
later `snapclick 5` in a separate message.

---

## Latency Budget

```
WhatsApp delivery:     ~200-500ms (E2EE + relay)
Baileys processing:    ~5-10ms
Command parsing:       <1ms
Casper entity call:    <1ms (FFI → C function invocation)
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

### Handle Cleanup

Snapshot handles hold live Rust-side `AXUIElement` references. The
`disposeSession` function releases them when a chat goes idle or the
process exits. Without cleanup, the handle table grows unbounded.

---

## File Layout

```
casper/
├── deno/
│   ├── casper/
│   │   ├── mod.ts              # public API re-exports
│   │   ├── types.ts            # Point, Rect, Size
│   │   ├── ffi/
│   │   │   ├── symbols.ts      # Deno.dlopen + symbol defs
│   │   │   ├── handles.ts      # Handle base class
│   │   │   └── helpers.ts      # pointer/buffer utilities
│   │   └── entities/
│   │       ├── app.ts          # App (handle-based)
│   │       ├── window.ts       # Window (handle-based)
│   │       ├── element.ts      # Element (handle-based)
│   │       ├── snapshot.ts     # Snapshot (holds Element refs)
│   │       ├── keyboard.ts     # Keyboard (stateless singleton)
│   │       ├── mouse.ts        # Mouse (stateless singleton)
│   │       ├── screen.ts       # Screen (stateless singleton)
│   │       ├── clipboard.ts    # Clipboard (stateless singleton)
│   │       └── ...
│   ├── client.ts               # agent client
│   ├── whatsapp_client.ts      # WhatsApp client (this doc)
│   ├── casper-whatsapp.ts      # entry point (connect + route)
│   └── auth_state/             # Baileys session (gitignored)
└── src/                        # Rust
    ├── lib.rs
    ├── ffi.rs                  # extern "C" entry points
    ├── handles.rs              # handle table
    ├── input.rs, ax.rs, capture.rs, apps.rs, ...
```

Both `client.ts` and `whatsapp_client.ts` import from `casper/mod.ts`.
The Rust crate is built once and loaded once.
