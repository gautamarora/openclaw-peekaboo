# Casper: Entity-Based Mac Automation

Casper is a TypeScript API for macOS desktop automation, backed by Rust FFI.
Instead of flat function calls (`click(x, y)`, `listWindows(pid)`), everything
is an entity with methods: `App`, `Window`, `Element`, `Keyboard`.

```typescript
const safari = await App.launch("com.apple.Safari");
const win = (await safari.windows())[0];
await win.focus();

const buttons = await win.findAll({ role: "AXButton" });
const submit = buttons.find(b => b.title === "Submit");
await submit.click();

await Keyboard.type("hello world");
await Keyboard.hotkey("cmd+enter");

const screenshot = await Screen.capture();
await Deno.writeFile("/tmp/screen.png", screenshot);
```

---

## Entity Hierarchy

```
Casper
├── App
│   ├── .launch(bundleId) → App          (static)
│   ├── .frontmost() → App               (static)
│   ├── .all() → App[]                   (static)
│   ├── .find(name | bundleId) → App     (static)
│   ├── .pid, .name, .bundleId
│   ├── .activate(), .quit(), .hide()
│   ├── .windows() → Window[]
│   ├── .focusedWindow() → Window
│   └── .menu(path), .menus()
│
│   ├── Browser extends App
│   │   ├── .open(name?) → Browser       (static — defaults to default browser)
│   │   ├── .navigate(url), .url() → string
│   │   ├── .tabs() → Tab[], .activeTab() → Tab
│   │   ├── .newTab(url?), .closeTab(index?)
│   │   └── .search(query)
│   │
│   ├── Mail extends App
│   │   ├── .open() → Mail               (static — opens default mail client)
│   │   ├── .inbox(opts?) → Message[]
│   │   ├── .compose(to, subject, body) → void
│   │   ├── .reply(message, body) → void
│   │   └── .accounts() → string[]
│   │
│   └── MusicPlayer extends App
│       ├── .open(name?) → MusicPlayer   (static)
│       ├── .nowPlaying() → Track
│       ├── .play(uri?), .pause(), .next(), .previous()
│       ├── .search(query) → void
│       └── .volume(level?) → number
│
├── Window
│   ├── .focus(), .close(), .minimize(), .maximize()
│   ├── .move(point), .resize(size), .bounds()
│   ├── .capture() → Uint8Array
│   ├── .title, .id, .app
│   ├── .findAll(query) → Element[]
│   ├── .find(query) → Element | null
│   ├── .waitFor(query, timeout?) → Element
│   └── .snapshot(opts?) → Snapshot
│
├── Element
│   ├── .role, .title, .label, .value, .bounds
│   ├── .click(), .doubleClick(), .rightClick()
│   ├── .type(text), .clear(), .focus()
│   ├── .parent(), .children()
│   └── .findAll(query), .find(query)
│
├── Snapshot
│   ├── .text → string                   (compact AX tree with [ref=N] tags)
│   ├── .refs → Map<number, Element>     (ref → live handle)
│   ├── .click(ref), .type(ref, text)
│   └── .dispose()
│
├── Keyboard                             (stateless)
│   ├── .type(text), .hotkey(keys), .press(key)
│   └── .keyDown(key), .keyUp(key)
│
├── Mouse                                (stateless)
│   ├── .click(point), .doubleClick(point), .rightClick(point)
│   ├── .move(point), .drag(from, to)
│   └── .scroll(direction, amount?)
│
├── Screen                               (stateless)
│   └── .capture(displayId?) → Uint8Array
│
├── Clipboard                            (stateless)
│   ├── .read(), .write(text), .clear()
│   └── .readImage() → Uint8Array | null
│
├── Script                               (stateless — escape hatch for AppleScript)
│   ├── .tell(app, command) → string
│   ├── .eval(source) → Record
│   └── .canScript(app) → boolean
│
└── Permissions
    ├── .check() → PermissionsStatus
    ├── .accessibility → boolean
    └── .screenRecording → boolean
```

App-specific entities (Browser, Mail, MusicPlayer) inherit all of App's
handle-based powers (windows, focus, activate, snapshot) and add typed
methods for domain actions. Internally they use Script.tell() or AX — but
callers never see raw AppleScript strings.

Entities that hold state (App, Window, Element) have handles backed by live
Rust-side objects. Stateless entities (Keyboard, Mouse, Screen, Clipboard,
Script) are global singletons that call macOS APIs directly.

---

## Design Principles

### 1. Entities hold handles, not data

Each stateful entity holds a **handle** — an opaque numeric ID mapping to a
Rust-side object (an `AXUIElement`, a PID, a window ID). The handle stays
valid until explicitly released.

```typescript
const safari = await App.find("Safari");
safari._handle;       // 1 — points to a live AXUIElement in Rust

const win = (await safari.windows())[0];
win._handle;          // 2

const btn = await win.find({ role: "AXButton", title: "OK" });
btn._handle;          // 3
await btn.click();    // Rust reads btn's CURRENT position, clicks it
```

Without handles, every operation re-walks the AX tree and returns stale
coordinate snapshots. With handles, Rust holds a live reference and reads
the element's current position at action time.

### 2. Queries, not raw coordinates

Elements are found by query objects, not screen coordinates:

```typescript
interface ElementQuery {
  role?: string;           // "AXButton", "AXTextField"
  title?: string;          // exact match
  titleContains?: string;  // substring
  label?: string;          // AXDescription
  value?: string;
  identifier?: string;     // AXIdentifier
  enabled?: boolean;
}

const email = await win.find({ role: "AXTextField", label: "Email" });
const buttons = await win.findAll({ role: "AXButton", enabled: true });
const spinner = await win.waitFor({ role: "AXBusyIndicator" }, 5000);
```

### 3. Actions are async

All macOS API calls are `async`. This keeps the Deno event loop responsive.

### 4. Disposable handles

Entities that hold handles implement `Disposable`. Use `using` blocks or
explicit `.dispose()`:

```typescript
{
  using app = await App.launch("com.apple.TextEdit");
  const win = (await app.windows())[0];
  const field = await win.find({ role: "AXTextArea" });
  await field.type("Hello from Casper");
} // handles released
```

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  Deno (TypeScript)                                   │
│                                                      │
│  App  Window  Element  Keyboard  Mouse  Screen  ... │
│                    │                                  │
│              FFI (Deno.dlopen)                        │
└────────────────────┼─────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────┐
│  Rust (libcasper.dylib)                               │
│                                                      │
│  ffi.rs  ←  handles.rs  ←  ax.rs, input.rs, ...     │
│                                                      │
│  macOS: ApplicationServices, CoreGraphics, AppKit    │
└──────────────────────────────────────────────────────┘
```

The FFI boundary uses `extern "C"` functions. Stateful operations take a
handle ID (`u64`), look it up in the handle table, and operate on the live
object. Stateless operations (keyboard, mouse, screen) take raw values.

Returns use a pointer+length pattern: Rust allocates a buffer, writes JSON
or PNG bytes, returns the pointer and length. Deno reads and then frees.

---

## Rust: Handle Table

```rust
// handles.rs

use std::collections::HashMap;
use std::sync::Mutex;
use std::sync::atomic::{AtomicU64, Ordering};

static NEXT_HANDLE: AtomicU64 = AtomicU64::new(1);
static HANDLE_TABLE: Mutex<Option<HashMap<u64, HandleEntry>>> = Mutex::new(None);

pub enum HandleEntry {
    AXElement(crate::ax::AXElement),
    App { pid: i32, bundle_id: String, name: String, ax: crate::ax::AXElement },
    Window { window_id: u32, pid: i32, ax: crate::ax::AXElement },
}

pub fn insert(entry: HandleEntry) -> u64 {
    let id = NEXT_HANDLE.fetch_add(1, Ordering::Relaxed);
    table().as_mut().unwrap().insert(id, entry);
    id
}

pub fn release(handle: u64) {
    table().as_mut().unwrap().remove(&handle);
}

pub fn release_all() {
    table().as_mut().unwrap().clear();
}

fn table() -> std::sync::MutexGuard<'static, Option<HashMap<u64, HandleEntry>>> {
    let mut guard = HANDLE_TABLE.lock().unwrap();
    if guard.is_none() { *guard = Some(HashMap::new()); }
    guard
}
```

---

## Rust: FFI Surface

The complete `extern "C"` interface. Handle-based operations look up the
handle and dispatch to core modules. Stateless operations call core modules
directly.

```rust
// ffi.rs

use crate::handles;
use std::ptr;

// --- Buffer helpers ---

#[unsafe(no_mangle)]
pub extern "C" fn casper_free_buffer(ptr: *mut u8, len: u64) { /* ... */ }
fn vec_to_ffi(data: Vec<u8>, out_len: *mut u64) -> *mut u8 { /* ... */ }
fn json_to_ffi<T: serde::Serialize>(value: &T, out_len: *mut u64) -> *mut u8 { /* ... */ }
unsafe fn str_from_buf(ptr: *const u8, len: u32) -> &'static str { /* ... */ }

// --- Lifecycle ---

#[unsafe(no_mangle)]
pub extern "C" fn casper_release(handle: u64) { handles::release(handle); }

#[unsafe(no_mangle)]
pub extern "C" fn casper_release_all() { handles::release_all(); }

// --- App ---

/// List all running GUI apps. Returns JSON array, each with a handle field.
#[unsafe(no_mangle)]
pub extern "C" fn casper_app_all(out_len: *mut u64) -> *mut u8 {
    let apps = crate::apps::list_applications();
    let result: Vec<serde_json::Value> = apps.into_iter().map(|app| {
        let ax = crate::ax::application(app.pid);
        let handle = handles::insert(handles::HandleEntry::App {
            pid: app.pid,
            bundle_id: app.bundle_id.clone().unwrap_or_default(),
            name: app.name.clone().unwrap_or_default(),
            ax,
        });
        serde_json::json!({
            "handle": handle, "pid": app.pid,
            "bundleId": app.bundle_id, "name": app.name,
        })
    }).collect();
    json_to_ffi(&result, out_len)
}

#[unsafe(no_mangle)]
pub extern "C" fn casper_app_frontmost(out_len: *mut u64) -> *mut u8 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_app_launch(
    bundle_id: *const u8, bundle_id_len: u32, out_len: *mut u64,
) -> *mut u8 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_app_find(
    query: *const u8, query_len: u32, out_len: *mut u64,
) -> *mut u8 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_app_activate(handle: u64) -> i32 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_app_quit(handle: u64, force: u8) -> i32 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_app_windows(handle: u64, out_len: *mut u64) -> *mut u8 { /* ... */ }

// --- Window ---

#[unsafe(no_mangle)]
pub extern "C" fn casper_window_focus(handle: u64) -> i32 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_window_close(handle: u64) -> i32 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_window_capture(handle: u64, out_len: *mut u64) -> *mut u8 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_window_find_all(
    handle: u64, query_json: *const u8, query_len: u32, out_len: *mut u64,
) -> *mut u8 { /* ... */ }

// --- Element ---

/// Click an element by handle. Reads current position from AX at click time.
#[unsafe(no_mangle)]
pub extern "C" fn casper_element_click(handle: u64, click_type: u8) -> i32 {
    let table = handles::table();
    let map = table.as_ref().unwrap();
    let ax = match map.get(&handle) {
        Some(handles::HandleEntry::AXElement(ax)) => ax,
        _ => return -1,
    };
    let (x, y, w, h) = match ax.frame() { Some(f) => f, None => return -1 };
    let center_x = x + w / 2.0;
    let center_y = y + h / 2.0;
    drop(table);
    match crate::input::click(center_x, center_y, crate::input::MouseButton::Left, 1) {
        Ok(()) => 0, Err(_) => -1,
    }
}

#[unsafe(no_mangle)]
pub extern "C" fn casper_element_type(
    handle: u64, text: *const u8, text_len: u32, delay_ms: u64,
) -> i32 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_element_props(handle: u64, out_len: *mut u64) -> *mut u8 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_element_find_all(
    handle: u64, query_json: *const u8, query_len: u32, out_len: *mut u64,
) -> *mut u8 { /* ... */ }

// --- Input (stateless) ---

#[unsafe(no_mangle)]
pub extern "C" fn casper_keyboard_type(text: *const u8, text_len: u32, delay_ms: u64) -> i32 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_keyboard_hotkey(keys: *const u8, keys_len: u32, hold_ms: u64) -> i32 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_mouse_click(x: f64, y: f64, button: u8, count: u32) -> i32 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_mouse_move(x: f64, y: f64) -> i32 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_mouse_drag(
    from_x: f64, from_y: f64, to_x: f64, to_y: f64, steps: u32, delay_ms: u64,
) -> i32 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_mouse_scroll(dx: i32, dy: i32) -> i32 { /* ... */ }

// --- Screen, Clipboard, Permissions (stateless) ---

#[unsafe(no_mangle)]
pub extern "C" fn casper_screen_capture(display_id: u32, out_len: *mut u64) -> *mut u8 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_clipboard_read(out_len: *mut u64) -> *mut u8 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_clipboard_write(text: *const u8, text_len: u32) -> i32 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_permissions_check(out_len: *mut u64) -> *mut u8 { /* ... */ }

// --- AppleScript ---

#[unsafe(no_mangle)]
pub extern "C" fn casper_script_tell(
    app: *const u8, app_len: u32,
    command: *const u8, command_len: u32,
    launch_if_needed: u8,
    out_len: *mut u64,
) -> *mut u8 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_script_eval(
    source: *const u8, source_len: u32, out_len: *mut u64,
) -> *mut u8 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_script_can_script(app: *const u8, app_len: u32) -> u8 { /* ... */ }

// --- Snapshots ---

#[unsafe(no_mangle)]
pub extern "C" fn casper_window_snapshot(
    handle: u64, max_depth: u32, web_only: u8, include_bounds: u8,
    out_len: *mut u64,
) -> *mut u8 { /* ... */ }

// --- ElementQuery ---

#[derive(serde::Deserialize)]
struct ElementQuery {
    role: Option<String>,
    title: Option<String>,
    #[serde(rename = "titleContains")]
    title_contains: Option<String>,
    label: Option<String>,
    value: Option<String>,
    identifier: Option<String>,
    enabled: Option<bool>,
}

impl ElementQuery {
    fn matches(&self, elem: &crate::ax::AXElement) -> bool {
        if let Some(ref role) = self.role {
            if elem.role().as_deref() != Some(role.as_str()) { return false; }
        }
        if let Some(ref title) = self.title {
            if elem.title().as_deref() != Some(title.as_str()) { return false; }
        }
        if let Some(ref contains) = self.title_contains {
            match elem.title() {
                Some(t) if t.contains(contains.as_str()) => {}
                _ => return false,
            }
        }
        if let Some(ref label) = self.label {
            if elem.label().as_deref() != Some(label.as_str()) { return false; }
        }
        // ... value, identifier, enabled
        true
    }
}
```

---

## TypeScript: Entity Classes

### FFI symbols

```typescript
// casper/ffi/symbols.ts
const lib = Deno.dlopen("./target/release/libcasper.dylib", {
  casper_free_buffer: { parameters: ["pointer", "u64"], result: "void" },
  casper_release: { parameters: ["u64"], result: "void" },
  casper_release_all: { parameters: [], result: "void" },

  casper_app_all: { parameters: ["pointer"], result: "pointer" },
  casper_app_frontmost: { parameters: ["pointer"], result: "pointer" },
  casper_app_launch: { parameters: ["buffer", "u32", "pointer"], result: "pointer" },
  casper_app_find: { parameters: ["buffer", "u32", "pointer"], result: "pointer" },
  casper_app_activate: { parameters: ["u64"], result: "i32" },
  casper_app_quit: { parameters: ["u64", "u8"], result: "i32" },
  casper_app_windows: { parameters: ["u64", "pointer"], result: "pointer" },

  casper_window_focus: { parameters: ["u64"], result: "i32" },
  casper_window_close: { parameters: ["u64"], result: "i32" },
  casper_window_capture: { parameters: ["u64", "pointer"], result: "pointer" },
  casper_window_find_all: { parameters: ["u64", "buffer", "u32", "pointer"], result: "pointer" },
  casper_window_snapshot: { parameters: ["u64", "u32", "u8", "u8", "pointer"], result: "pointer" },

  casper_element_click: { parameters: ["u64", "u8"], result: "i32" },
  casper_element_type: { parameters: ["u64", "buffer", "u32", "u64"], result: "i32" },
  casper_element_props: { parameters: ["u64", "pointer"], result: "pointer" },
  casper_element_find_all: { parameters: ["u64", "buffer", "u32", "pointer"], result: "pointer" },

  casper_keyboard_type: { parameters: ["buffer", "u32", "u64"], result: "i32" },
  casper_keyboard_hotkey: { parameters: ["buffer", "u32", "u64"], result: "i32" },
  casper_mouse_click: { parameters: ["f64", "f64", "u8", "u32"], result: "i32" },
  casper_mouse_move: { parameters: ["f64", "f64"], result: "i32" },
  casper_mouse_drag: { parameters: ["f64", "f64", "f64", "f64", "u32", "u64"], result: "i32" },
  casper_mouse_scroll: { parameters: ["i32", "i32"], result: "i32" },
  casper_screen_capture: { parameters: ["u32", "pointer"], result: "pointer" },
  casper_clipboard_read: { parameters: ["pointer"], result: "pointer" },
  casper_clipboard_write: { parameters: ["buffer", "u32"], result: "i32" },
  casper_permissions_check: { parameters: ["pointer"], result: "pointer" },
  casper_script_tell: { parameters: ["buffer", "u32", "buffer", "u32", "u8", "pointer"], result: "pointer" },
  casper_script_eval: { parameters: ["buffer", "u32", "pointer"], result: "pointer" },
  casper_script_can_script: { parameters: ["buffer", "u32"], result: "u8" },
});

export default lib;
export const sym = lib.symbols;
```

### Handle base class

```typescript
// casper/ffi/handles.ts
import { sym } from "./symbols.ts";

export class Handle implements Disposable {
  readonly _handle: number;
  #released = false;

  constructor(handle: number) { this._handle = handle; }

  dispose(): void {
    if (!this.#released) {
      sym.casper_release(BigInt(this._handle));
      this.#released = true;
    }
  }

  [Symbol.dispose](): void { this.dispose(); }
}
```

### App

```typescript
// casper/entities/app.ts
import { Handle } from "../ffi/handles.ts";

export class App extends Handle {
  readonly pid: number;
  readonly bundleId: string;
  readonly name: string;

  private constructor(data: { handle: number; pid: number; bundleId: string; name: string }) {
    super(data.handle);
    this.pid = data.pid;
    this.bundleId = data.bundleId ?? "";
    this.name = data.name ?? "";
  }

  static async all(): Promise<App[]> {
    return (callJson<any[]>((out) => sym.casper_app_all(out)) ?? []).map((d) => new App(d));
  }

  static async frontmost(): Promise<App> {
    const data = callJson<any>((out) => sym.casper_app_frontmost(out));
    if (!data) throw new Error("No frontmost application");
    return new App(data);
  }

  static async find(query: string): Promise<App> {
    const [buf, len] = encodeStr(query);
    const data = callJson<any>((out) => sym.casper_app_find(buf, len, out));
    if (!data) throw new Error(`App not found: ${query}`);
    return new App(data);
  }

  static async launch(bundleId: string): Promise<App> {
    const [buf, len] = encodeStr(bundleId);
    const data = callJson<any>((out) => sym.casper_app_launch(buf, len, out));
    if (!data) throw new Error(`Failed to launch: ${bundleId}`);
    return new App(data);
  }

  async activate(): Promise<void> { assertOk(sym.casper_app_activate(BigInt(this._handle)), "activate"); }
  async quit(force = false): Promise<void> { assertOk(sym.casper_app_quit(BigInt(this._handle), force ? 1 : 0), "quit"); }

  async windows(): Promise<Window[]> {
    return (callJson<any[]>((out) => sym.casper_app_windows(BigInt(this._handle), out)) ?? [])
      .map((d) => new Window(d, this));
  }

  async focusedWindow(): Promise<Window> { return (await this.windows())[0]; }
}
```

### Window

```typescript
// casper/entities/window.ts
import { Handle } from "../ffi/handles.ts";
import { Element, type ElementQuery } from "./element.ts";

export class Window extends Handle {
  readonly title: string;
  readonly app: App;
  readonly bounds: Rect;

  constructor(data: { handle: number; title: string; bounds: Rect }, app: App) {
    super(data.handle);
    this.title = data.title ?? "";
    this.app = app;
    this.bounds = data.bounds;
  }

  async focus(): Promise<void> { assertOk(sym.casper_window_focus(BigInt(this._handle)), "focus"); }
  async close(): Promise<void> { assertOk(sym.casper_window_close(BigInt(this._handle)), "close"); }

  async capture(): Promise<Uint8Array> {
    const data = callBytes((out) => sym.casper_window_capture(BigInt(this._handle), out));
    if (!data) throw new Error("Window capture failed");
    return data;
  }

  async findAll(query: ElementQuery): Promise<Element[]> {
    const [buf, len] = encodeStr(JSON.stringify(query));
    return (callJson<any[]>((out) => sym.casper_window_find_all(BigInt(this._handle), buf, len, out)) ?? [])
      .map((d) => new Element(d));
  }

  async find(query: ElementQuery): Promise<Element | null> {
    return (await this.findAll(query))[0] ?? null;
  }

  async waitFor(query: ElementQuery, timeoutMs = 5000): Promise<Element> {
    const start = Date.now();
    while (Date.now() - start < timeoutMs) {
      const el = await this.find(query);
      if (el) return el;
      await new Promise((r) => setTimeout(r, 200));
    }
    throw new Error(`Timed out waiting for element: ${JSON.stringify(query)}`);
  }
}
```

### Element

```typescript
// casper/entities/element.ts
import { Handle } from "../ffi/handles.ts";

export interface ElementQuery {
  role?: string;
  title?: string;
  titleContains?: string;
  label?: string;
  value?: string;
  identifier?: string;
  enabled?: boolean;
}

export class Element extends Handle {
  readonly role: string | null;
  readonly title: string | null;
  readonly label: string | null;
  readonly value: string | null;
  readonly bounds: Rect;

  constructor(data: { handle: number; role: string; title: string; label: string; value: string; bounds: Rect }) {
    super(data.handle);
    this.role = data.role;
    this.title = data.title;
    this.label = data.label;
    this.value = data.value;
    this.bounds = data.bounds;
  }

  async refresh(): Promise<{ role: string; title: string; label: string; value: string; bounds: Rect }> {
    return callJson((out) => sym.casper_element_props(BigInt(this._handle), out))!;
  }

  async click(): Promise<void> { assertOk(sym.casper_element_click(BigInt(this._handle), 0), "click"); }
  async doubleClick(): Promise<void> { assertOk(sym.casper_element_click(BigInt(this._handle), 1), "doubleClick"); }
  async rightClick(): Promise<void> { assertOk(sym.casper_element_click(BigInt(this._handle), 2), "rightClick"); }

  async type(text: string, delayMs = 50): Promise<void> {
    const [buf, len] = encodeStr(text);
    assertOk(sym.casper_element_type(BigInt(this._handle), buf, len, BigInt(delayMs)), "type");
  }

  async findAll(query: ElementQuery): Promise<Element[]> {
    const [buf, len] = encodeStr(JSON.stringify(query));
    return (callJson<any[]>((out) => sym.casper_element_find_all(BigInt(this._handle), buf, len, out)) ?? [])
      .map((d) => new Element(d));
  }

  async find(query: ElementQuery): Promise<Element | null> {
    return (await this.findAll(query))[0] ?? null;
  }
}
```

### Keyboard, Mouse, Screen, Clipboard

```typescript
// casper/entities/keyboard.ts
export const Keyboard = {
  async type(text: string, delayMs = 50): Promise<void> {
    const [buf, len] = encodeStr(text);
    assertOk(sym.casper_keyboard_type(buf, len, BigInt(delayMs)), "Keyboard.type");
  },
  async hotkey(keys: string, holdMs = 0): Promise<void> {
    const [buf, len] = encodeStr(keys);
    assertOk(sym.casper_keyboard_hotkey(buf, len, BigInt(holdMs)), "Keyboard.hotkey");
  },
  async press(key: string): Promise<void> { await this.hotkey(key); },
};

// casper/entities/mouse.ts
export const Mouse = {
  async click(point: Point, button: "left" | "right" = "left", count = 1): Promise<void> {
    assertOk(sym.casper_mouse_click(point.x, point.y, button === "right" ? 1 : 0, count), "Mouse.click");
  },
  async move(point: Point): Promise<void> { assertOk(sym.casper_mouse_move(point.x, point.y), "Mouse.move"); },
  async drag(from: Point, to: Point, steps = 20, stepDelayMs = 10): Promise<void> {
    assertOk(sym.casper_mouse_drag(from.x, from.y, to.x, to.y, steps, BigInt(stepDelayMs)), "Mouse.drag");
  },
  async scroll(direction: "up" | "down" | "left" | "right", amount = 3): Promise<void> {
    const [dx, dy] = { up: [0, amount], down: [0, -amount], left: [amount, 0], right: [-amount, 0] }[direction];
    assertOk(sym.casper_mouse_scroll(dx, dy), "Mouse.scroll");
  },
};

// casper/entities/screen.ts
export const Screen = {
  async capture(displayId = 0): Promise<Uint8Array> {
    const data = callBytes((out) => sym.casper_screen_capture(displayId, out));
    if (!data) throw new Error("Screen capture failed");
    return data;
  },
};

// casper/entities/clipboard.ts
export const Clipboard = {
  read(): string | null {
    const data = callBytes((out) => sym.casper_clipboard_read(out));
    return data ? new TextDecoder().decode(data) : null;
  },
  write(text: string): void {
    const [buf, len] = encodeStr(text);
    assertOk(sym.casper_clipboard_write(buf, len), "Clipboard.write");
  },
  clear(): void { this.write(""); },
};
```

### Public API

```typescript
// casper/mod.ts
export { App } from "./entities/app.ts";
export { Window } from "./entities/window.ts";
export { Element, type ElementQuery } from "./entities/element.ts";
export { Keyboard } from "./entities/keyboard.ts";
export { Mouse } from "./entities/mouse.ts";
export { Screen } from "./entities/screen.ts";
export { Clipboard } from "./entities/clipboard.ts";
export { Browser } from "./entities/browser.ts";
export { Mail } from "./entities/mail.ts";
export { MusicPlayer } from "./entities/music-player.ts";
export type { Point, Rect, Size } from "./types.ts";

import lib from "./ffi/symbols.ts";
export function shutdown(): void { lib.symbols.casper_release_all(); lib.close(); }
```

---

## App-Specific Entities

App subclasses wrap Script/AX internals in typed methods. The pattern:

1. Extend `App` — inherit handle, windows, focus, snapshot
2. Add domain methods — typed, async, no AppleScript visible to callers
3. Keep app knowledge private — bundle IDs, verbs, queries are implementation details

`Script.tell()` is like `eval()` in JavaScript: powerful but untyped. App
entities are the typed wrapper — callers get autocomplete, type checking, and
no raw AppleScript strings.

### Browser

```typescript
// casper/entities/browser.ts
import { App } from "./app.ts";

interface Tab {
  title: string;
  url: string;
  index: number;
}

export class Browser extends App {
  /** Open a browser by name, or the system default. */
  static async open(name?: string): Promise<Browser> {
    const bundleId = name
      ? (await App.find(name)).bundleId
      : "com.apple.Safari";  // TODO: read LSHandlers for default
    const app = await App.launch(bundleId);
    return new Browser(app);
  }

  /** Navigate the active tab to a URL. */
  async navigate(url: string): Promise<void> {
    await Script.tell(this.name, `set URL of front document to "${url}"`);
  }

  /** Get the URL of the active tab. */
  async url(): Promise<string> {
    return (await Script.tell(this.name, "return URL of front document")) ?? "";
  }

  /** List all tabs. */
  async tabs(): Promise<Tab[]> {
    const raw = await Script.eval(`
      tell application "${this.name}"
        set output to {}
        repeat with t in tabs of front window
          set end of output to {name of t, URL of t}
        end repeat
        return output
      end tell
    `);
    // parse into Tab[]
    return this.parseTabs(raw);
  }

  /** Open a new tab, optionally navigating to a URL. */
  async newTab(url?: string): Promise<void> {
    if (url) {
      await Script.tell(this.name, `make new document with properties {URL:"${url}"}`);
    } else {
      await Keyboard.hotkey("cmd+t");
    }
  }

  async activeTab(): Promise<Tab> {
    const result = await Script.eval(`
      tell application "${this.name}"
        return {name of front document, URL of front document}
      end tell
    `);
    return this.parseTab(result, 0);
  }

  async search(query: string): Promise<void> {
    await this.activate();
    await Keyboard.hotkey("cmd+l");
    await Keyboard.type(query);
    await Keyboard.press("return");
  }

  // -- private helpers for parsing Script output --
  private parseTabs(raw: unknown): Tab[] { /* ... */ }
  private parseTab(raw: unknown, index: number): Tab { /* ... */ }
}
```

### Mail

```typescript
// casper/entities/mail.ts
import { App } from "./app.ts";

interface Message {
  id: string;
  subject: string;
  sender: string;
  date: Date;
  isRead: boolean;
  snippet: string;
}

export class Mail extends App {
  /** Open the default mail client. */
  static async open(): Promise<Mail> {
    const app = await App.launch("com.apple.mail");
    return new Mail(app);
  }

  /** Fetch inbox messages, optionally filtered. */
  async inbox(opts?: { unread?: boolean; limit?: number }): Promise<Message[]> {
    const limit = opts?.limit ?? 20;
    const filter = opts?.unread ? "whose read status is false" : "";
    const raw = await Script.eval(`
      tell application "Mail"
        set msgs to (messages of inbox ${filter})
        set output to {}
        repeat with i from 1 to (minimum of {${limit}, count of msgs})
          set m to item i of msgs
          set end of output to {id of m, subject of m, sender of m, ¬
            date sent of m, read status of m}
        end repeat
        return output
      end tell
    `);
    return this.parseMessages(raw);
  }

  /** Compose and send a new email. */
  async compose(to: string, subject: string, body: string): Promise<void> {
    await Script.eval(`
      tell application "Mail"
        set newMsg to make new outgoing message with properties ¬
          {subject:"${subject}", content:"${body}", visible:true}
        tell newMsg
          make new to recipient at end of to recipients ¬
            with properties {address:"${to}"}
        end tell
        send newMsg
      end tell
    `);
  }

  /** Reply to a message. */
  async reply(message: Message, body: string): Promise<void> {
    await Script.eval(`
      tell application "Mail"
        set m to first message of inbox whose id is "${message.id}"
        set r to reply m with opening window
        set content of r to "${body}"
        send r
      end tell
    `);
  }

  /** List mail accounts. */
  async accounts(): Promise<string[]> {
    const raw = await Script.tell("Mail", "return name of every account");
    return (raw ?? "").split(", ");
  }

  private parseMessages(raw: unknown): Message[] { /* ... */ }
}
```

### MusicPlayer

```typescript
// casper/entities/music-player.ts
import { App } from "./app.ts";

interface Track {
  name: string;
  artist: string;
  album: string;
  duration: number;
  playing: boolean;
}

const KNOWN_PLAYERS: Record<string, string> = {
  "Spotify": "com.spotify.client",
  "Music": "com.apple.Music",
};

export class MusicPlayer extends App {
  /** Open a music player by name (defaults to Spotify). */
  static async open(name = "Spotify"): Promise<MusicPlayer> {
    const bundleId = KNOWN_PLAYERS[name] ?? (await App.find(name)).bundleId;
    const app = await App.launch(bundleId);
    return new MusicPlayer(app);
  }

  /** What's currently playing? */
  async nowPlaying(): Promise<Track> {
    const raw = await Script.eval(`
      tell application "${this.name}"
        return {name of current track, artist of current track, ¬
                album of current track, duration of current track, player state}
      end tell
    `);
    return this.parseTrack(raw);
  }

  /** Play a track (by URI or just resume). */
  async play(uri?: string): Promise<void> {
    if (uri) {
      await Script.tell(this.name, `play track "${uri}"`);
    } else {
      await Script.tell(this.name, "play");
    }
  }

  async pause(): Promise<void> {
    await Script.tell(this.name, "pause");
  }

  async next(): Promise<void> {
    await Script.tell(this.name, "next track");
  }

  async previous(): Promise<void> {
    await Script.tell(this.name, "previous track");
  }

  /** Open search and type a query. Falls back to AX + keyboard. */
  async search(query: string): Promise<void> {
    await this.activate();
    await Keyboard.hotkey("cmd+k");
    const win = await this.focusedWindow();
    const field = await win.waitFor({ role: "AXTextField" }, 3000);
    await field.type(query);
    await Keyboard.press("return");
  }

  /** Get or set volume (0-100). */
  async volume(level?: number): Promise<number> {
    if (level !== undefined) {
      await Script.tell(this.name, `set sound volume to ${level}`);
      return level;
    }
    const raw = await Script.tell(this.name, "return sound volume");
    return parseInt(raw ?? "0", 10);
  }

  private parseTrack(raw: unknown): Track { /* ... */ }
}
```

### The pattern

Every app-specific entity follows the same shape:

```typescript
class SomeApp extends App {
  // 1. Static factory — knows the bundle ID
  static async open(): Promise<SomeApp> {
    return new SomeApp(await App.launch("com.example.app"));
  }

  // 2. Domain methods — typed params, typed returns
  async doThing(param: string): Promise<Result> {
    // 3. Implementation picks the best tool:
    //    - Script.tell() for scriptable apps
    //    - AX (this.focusedWindow().find(...)) for UI
    //    - fetch() for apps with local APIs
    //    - Keyboard.hotkey() for shortcuts
  }
}
```

Callers never see AppleScript. If an app is non-scriptable, the subclass
uses AX internally — the typed API stays the same.

---

## Snapshots

An agent controlling a GUI needs to see what's on screen, then act on what
it sees. Screenshots are expensive (~1,500 tokens per image) and imprecise.
Full AX dumps are overwhelming (5k+ tokens). Snapshots are the middle ground.

`Window.snapshot()` walks the AX tree and produces compact text with numbered
refs. Each ref maps to a live `Element` handle, so the agent can act on what
it saw without re-querying.

### Text format

```
window "Spotify" [ref=1]
  group "Now Playing Bar"
    image "Album Art" [ref=2]
    statictext "Bohemian Rhapsody"
    statictext "Queen"
    button "Previous" [ref=3]
    button "Pause" [ref=4]
    button "Next" [ref=5]
    slider "Volume" value=75 [ref=6]
  group "Main View"
    heading "Made For You" level=1
    list "Playlist Grid"
      cell "Daily Mix 1" [ref=7]
      cell "Release Radar" [ref=8]
```

Rules:
- One line per meaningful element, indented by tree depth
- `[ref=N]` only on actionable elements (buttons, links, text fields)
- ~200-500 tokens for a typical window

### The Snapshot entity

```typescript
export class Snapshot implements Disposable {
  readonly text: string;
  readonly refs: Map<number, Element>;

  async click(ref: number): Promise<void> {
    const el = this.refs.get(ref);
    if (!el) throw new Error(`Unknown ref: ${ref}`);
    await el.click();
  }

  async type(ref: number, text: string): Promise<void> {
    const el = this.refs.get(ref);
    if (!el) throw new Error(`Unknown ref: ${ref}`);
    await el.type(text);
  }

  get(ref: number): Element {
    const el = this.refs.get(ref);
    if (!el) throw new Error(`Unknown ref: ${ref}`);
    return el;
  }

  dispose(): void {
    for (const el of this.refs.values()) el.dispose();
    this.refs.clear();
  }

  [Symbol.dispose](): void { this.dispose(); }
}
```

### Usage

```typescript
const app = await App.frontmost();
const win = await app.focusedWindow();

// Snapshot → send to LLM → act on ref
using snap = await win.snapshot();
console.log(snap.text);  // send to LLM
await snap.click(5);     // LLM said "click ref 5"
```

---

## AppleScript & Script

AX and AppleScript are complementary:

| | AX | AppleScript |
|---|---|---|
| **Gives you** | UI structure (buttons, positions) | Semantic verbs (`play track`, `set volume`) |
| **Works on** | Every GUI app | Only apps with `.sdef` dictionaries |
| **Speed** | Must walk element tree | Direct command dispatch |
| **Best for** | Clicking elements, reading layout | App-specific actions, state queries |

### The Script entity — an escape hatch

`Script` is to Casper what `eval()` is to JavaScript: powerful, but untyped.
App-specific entity subclasses exist so that callers don't need Script directly.
Use Script as a last resort for one-off automation of apps without a typed entity.

```typescript
export const Script = {
  async tell(appName: string, command: string, opts?: { launchIfNeeded?: boolean }): Promise<string | null> {
    // → casper_script_tell("Spotify", "play track \"spotify:track:xxx\"", launch)
  },
  async eval(source: string): Promise<Record<string, unknown> | string | null> {
    // → casper_script_eval(source)
  },
  async canScript(appName: string): Promise<boolean> {
    // → casper_script_can_script("Spotify")
  },
};
```

### When to use what

```
Has a typed entity? (Browser, Mail, MusicPlayer)
  → Use the entity                             ← typed, autocomplete, no raw strings
  → browser.navigate(url)                      ← NOT Script.tell("Safari", "...")

Unknown scriptable app?
  → Script.tell() as escape hatch              ← still better than AX for verbs

Non-scriptable app (Electron, random GUI)?
  → AX: find → click → type                   ← always works

App with an HTTP API (Gmail, Obsidian)?
  → Just use fetch()                           ← it's TypeScript
```

The entity is always preferred over Script. Script is always preferred over
raw AX tree walking for verb-like actions. AX is for UI structure.

---

## Usage Examples

### 1. Open Hacker News in Safari

```typescript
import { Browser } from "./casper/mod.ts";

const safari = await Browser.open("Safari");
await safari.navigate("https://news.ycombinator.com");

// Want to interact with the page? Use AX through the entity:
const win = await safari.focusedWindow();
const links = await win.findAll({ role: "AXLink" });
const topStory = links[0];
if (topStory) await topStory.click();
```

No `Script.tell("Safari", ...)`. `Browser.navigate()` handles it internally.

### 2. Play top song on Spotify

```typescript
import { MusicPlayer } from "./casper/mod.ts";

const spotify = await MusicPlayer.open("Spotify");
await spotify.search("top hits 2026");

// Check what's playing
const track = await spotify.nowPlaying();
console.log(`Now playing: ${track.name} by ${track.artist}`);

// Control playback
await spotify.next();
await spotify.volume(80);
```

No `Script.eval("tell application \"Spotify\" ...")`. `MusicPlayer` wraps the
AppleScript verbs (play, pause, next track) and keyboard shortcuts (cmd+k for
search) internally. Callers get typed methods.

### 3. Read and respond to email

```typescript
import { Mail } from "./casper/mod.ts";

const mail = await Mail.open();

// Get unread messages — typed return, not raw AppleScript
const unread = await mail.inbox({ unread: true, limit: 10 });

for (const msg of unread) {
  console.log(`From: ${msg.sender} — ${msg.subject}`);

  // Decide which need a response (could be LLM-driven)
  if (msg.subject.includes("urgent")) {
    await mail.reply(msg, `Thanks for flagging this. I'll look into it.`);
  }
}

// Compose a new message
await mail.compose(
  "team@example.com",
  "Weekly update",
  "Here's the summary for this week..."
);
```

No 30-line `Script.eval()` with raw AppleScript. `Mail.inbox()` returns
typed `Message[]`. `Mail.reply()` takes a `Message` and a `string`.
The AppleScript is an implementation detail inside the entity.

### 4. Snapshot-driven agent (generic approach)

For apps without a typed entity, snapshots + AX still work:

```typescript
import { App } from "./casper/mod.ts";

const app = await App.frontmost();
const win = await app.focusedWindow();

using snap = await win.snapshot();
console.log(snap.text);  // send to LLM
await snap.click(3);     // LLM picks a ref
```

This is the fallback for unknown apps. For well-known apps, use the typed entity.

---

## Handle Flow

```
1. App.find("Safari")
   → casper_app_find("Safari")
   → Rust: NSRunningApplication → AXUIElement → handle=1
   → TS: new App({ handle: 1, pid: 12345, name: "Safari" })

2. app.windows()
   → casper_app_windows(handle=1)
   → Rust: handle 1 → app.ax.windows() → handles 2, 3
   → TS: [new Window(data, app)]

3. window.find({ role: "AXButton", title: "Submit" })
   → casper_window_find_all(handle=2, query)
   → Rust: handle 2 → find_all(query) → handle 4
   → TS: new Element(data)

4. element.click()
   → casper_element_click(handle=4)
   → Rust: handle 4 → frame() → CURRENT position → CGEventPost

5. element.dispose()
   → casper_release(handle=4) → drops AXUIElement
```

---

## File Layout

```
casper/
├── deno.json
├── Cargo.toml
├── build.rs
├── src/                          # Rust
│   ├── lib.rs
│   ├── ffi.rs                    # extern "C" — casper_* functions
│   ├── handles.rs                # handle table
│   ├── input.rs                  # CGEvent
│   ├── ax.rs                     # AXUIElement
│   ├── capture.rs                # screen/window capture
│   ├── apps.rs                   # NSWorkspace
│   ├── clipboard.rs              # NSPasteboard
│   ├── permissions.rs            # TCC checks
│   └── scripting.rs              # NSAppleScript / OSAScript
└── deno/                         # TypeScript
    ├── mod.ts                    # public API re-exports
    ├── types.ts                  # Point, Rect, Size
    ├── ffi/
    │   ├── symbols.ts            # Deno.dlopen + symbol defs
    │   ├── handles.ts            # Handle base class
    │   └── helpers.ts            # pointer/buffer utils
    ├── entities/
    │   ├── app.ts                # base App (handle-based)
    │   ├── browser.ts            # Browser extends App
    │   ├── mail.ts               # Mail extends App
    │   ├── music-player.ts       # MusicPlayer extends App
    │   ├── window.ts
    │   ├── element.ts
    │   ├── keyboard.ts
    │   ├── mouse.ts
    │   ├── screen.ts
    │   ├── clipboard.ts
    │   ├── script.ts             # escape hatch — raw AppleScript
    │   └── snapshot.ts
    └── test/
        ├── browser_test.ts
        ├── mail_test.ts
        └── music_test.ts
```

The `apps/` directory from earlier drafts is gone. App knowledge (bundle IDs,
AppleScript verbs, keyboard shortcuts) lives inside the entity subclasses as
private implementation details, not as exported plain objects.
