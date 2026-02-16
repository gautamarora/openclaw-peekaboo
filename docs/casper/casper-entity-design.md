# Casper: Entity-Based Mac Automation

Casper is an entity-oriented TypeScript API for macOS desktop automation,
backed by Rust FFI to native macOS frameworks. Instead of flat function
calls (`click(x, y)`, `listWindows(pid)`), everything is expressed as
`Entity.action()`.

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

## Entity Hierarchy

```
Casper (root)
├── Screen
│   ├── .capture() → Uint8Array
│   ├── .captureArea(rect) → Uint8Array
│   └── .displays() → Display[]
│
├── App
│   ├── .launch(bundleId) → App          (static)
│   ├── .frontmost() → App               (static)
│   ├── .all() → App[]                   (static)
│   ├── .find(name | bundleId) → App     (static)
│   │
│   ├── .pid → number
│   ├── .name → string
│   ├── .bundleId → string
│   ├── .activate()
│   ├── .quit(force?)
│   ├── .hide()
│   ├── .unhide()
│   ├── .windows() → Window[]
│   ├── .focusedWindow() → Window
│   ├── .menu(path) → MenuItem
│   ├── .menus() → MenuItem[]
│   └── .isRunning → boolean
│
├── Window
│   ├── .focus()
│   ├── .close()
│   ├── .minimize()
│   ├── .maximize()
│   ├── .move(point)
│   ├── .resize(size)
│   ├── .bounds() → Rect
│   ├── .capture() → Uint8Array
│   ├── .app → App
│   ├── .title → string
│   ├── .id → number
│   ├── .findAll(query) → Element[]
│   ├── .find(query) → Element | null
│   ├── .waitFor(query, timeout?) → Element
│   ├── .findInPage(query) → Element[]
│   ├── .waitForInPage(query, timeout?) → Element
│   └── .snapshot(opts?) → Snapshot
│
├── Element (AX UI element)
│   ├── .role → string
│   ├── .title → string | null
│   ├── .label → string | null
│   ├── .value → string | null
│   ├── .bounds → Rect
│   ├── .isEnabled → boolean
│   │
│   ├── .click()
│   ├── .doubleClick()
│   ├── .rightClick()
│   ├── .focus()
│   ├── .type(text)
│   ├── .clear()
│   ├── .press(action)          → AXPress, AXConfirm, etc.
│   ├── .scrollTo()
│   │
│   ├── .parent() → Element
│   ├── .children() → Element[]
│   ├── .findAll(query) → Element[]
│   └── .find(query) → Element | null
│
├── Browser (extends App)
│   ├── .open(bundleId?) → Browser       (static, defaults to default browser)
│   ├── .tabs() → Tab[]
│   ├── .activeTab() → Tab
│   └── .newTab(url?) → Tab
│
├── Tab (browser tab)
│   ├── .url → string
│   ├── .title → string
│   ├── .navigate(url)
│   ├── .reload()
│   ├── .close()
│   ├── .activate()
│   └── .window → Window
│
├── Finder (extends App)
│   ├── .open(path?) → Finder            (static)
│   ├── .selectedFiles() → File[]
│   ├── .currentFolder() → string
│   ├── .navigate(path)
│   └── .reveal(path) → File
│
├── File
│   ├── .path → string
│   ├── .name → string
│   ├── .open()
│   ├── .openWith(appBundleId)
│   ├── .reveal()                         → show in Finder
│   ├── .trash()
│   ├── .copyTo(dest)
│   └── .moveTo(dest)
│
├── Keyboard
│   ├── .type(text, opts?)
│   ├── .hotkey(keys)
│   ├── .press(key)
│   ├── .keyDown(key)
│   └── .keyUp(key)
│
├── Mouse
│   ├── .click(point, opts?)
│   ├── .doubleClick(point)
│   ├── .rightClick(point)
│   ├── .move(point)
│   ├── .drag(from, to, opts?)
│   ├── .scroll(direction, amount?)
│   └── .position() → Point
│
├── Clipboard
│   ├── .read() → string
│   ├── .write(text)
│   ├── .readImage() → Uint8Array | null
│   └── .clear()
│
├── Dialog (active system dialog)
│   ├── .find(opts?) → Dialog | null      (static)
│   ├── .title → string
│   ├── .buttons() → Element[]
│   ├── .textFields() → Element[]
│   ├── .clickButton(name)
│   ├── .enterText(text, field?)
│   └── .dismiss()
│
├── Menu
│   ├── .click(path)                      → "File > Save As..."
│   └── .items() → MenuItem[]
│
├── Snapshot (AX tree → text + handles)
│   ├── .text → string
│   ├── .refs → Map<number, Element>
│   ├── .click(ref) → void
│   ├── .type(ref, text) → void
│   ├── .get(ref) → Element
│   └── .dispose()
│
├── Permissions
│   ├── .check() → PermissionsStatus
│   ├── .accessibility → boolean
│   └── .screenRecording → boolean
│
└── Script (AppleScript / OSA)
    ├── .tell(app, command, opts?) → string  (static)
    ├── .eval(source) → Record              (static)
    └── .canScript(app) → boolean           (static)
```

## Design Principles

### 1. Entities hold handles, not data

Each entity instance holds a **handle** — an opaque numeric ID that maps
to a Rust-side object (an `AXUIElement`, a PID, a window ID, etc.). The
handle stays valid until explicitly released.

```typescript
// App holds a PID
const safari = await App.find("Safari");
safari.pid;        // 12345

// Window holds a CGWindowID + AX element handle
const win = (await safari.windows())[0];
win.id;            // 9001 (CGWindowID)
win._handle;       // 42 (Rust handle table index)

// Element holds an AX element handle
const btn = await win.find({ role: "AXButton", title: "OK" });
btn._handle;       // 43 (Rust handle table index)
await btn.click(); // calls Rust with handle 43
```

### 2. Queries, not raw coordinates

Elements are found by **query objects**, not screen coordinates. The query
matches against AX attributes:

```typescript
interface ElementQuery {
  role?: string;          // "AXButton", "AXTextField", "AXLink"
  title?: string;         // exact match
  titleContains?: string; // substring
  label?: string;         // AXDescription
  value?: string;         // AXValue
  identifier?: string;    // AXIdentifier
  enabled?: boolean;      // filter by enabled state
}

// Find all enabled buttons
const buttons = await win.findAll({ role: "AXButton", enabled: true });

// Find a specific text field
const email = await win.find({ role: "AXTextField", label: "Email" });

// Wait for an element to appear (polls AX tree)
const spinner = await win.waitFor({ role: "AXBusyIndicator" }, 5000);
```

### 3. Actions are async

All actions that touch macOS APIs are `async`. This keeps the Deno event
loop responsive and allows using `nonblocking: true` in `Deno.dlopen` for
operations that might be slow (AX tree walks, screen capture).

### 4. Using blocks for disposable contexts

Entities that hold handles should be disposable. Using `Deno.Disposable`
(`using` keyword) or explicit `.dispose()`:

```typescript
{
  using app = await App.launch("com.apple.TextEdit");
  const win = (await app.windows())[0];
  const field = await win.find({ role: "AXTextArea" });
  await field.type("Hello from Casper");
  await Keyboard.hotkey("cmd+s");
} // app handle released, AX element handles freed
```

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  Deno — TypeScript                                           │
│                                                              │
│  ┌─────────────────────────────────────────┐                 │
│  │  Entity Layer (casper/)                 │                 │
│  │                                         │                 │
│  │  App  Window  Element  Browser  Finder  │                 │
│  │  Mouse  Keyboard  Screen  Clipboard     │                 │
│  │  Dialog  File  Tab  Menu                │                 │
│  └──────────────┬──────────────────────────┘                 │
│                 │ calls                                       │
│  ┌──────────────▼──────────────────────────┐                 │
│  │  FFI Bridge (casper/ffi/)               │                 │
│  │                                         │                 │
│  │  ffi.ts       — Deno.dlopen symbols     │                 │
│  │  handles.ts   — handle lifecycle mgmt   │                 │
│  │  helpers.ts   — pointer/buffer utils    │                 │
│  └──────────────┬──────────────────────────┘                 │
│                 │ Deno.dlopen                                 │
└─────────────────┼────────────────────────────────────────────┘
                  │
┌─────────────────▼────────────────────────────────────────────┐
│  Rust — libcasper.dylib                                      │
│                                                              │
│  ┌─────────────────────────────────────────┐                 │
│  │  FFI Surface (ffi.rs)                   │                 │
│  │                                         │                 │
│  │  casper_* extern "C" functions          │                 │
│  │  Handle-based dispatch                  │                 │
│  └──────────────┬──────────────────────────┘                 │
│                 │                                             │
│  ┌──────────────▼──────────────────────────┐                 │
│  │  Handle Table (handles.rs)              │                 │
│  │                                         │                 │
│  │  HashMap<u64, HandleEntry>              │                 │
│  │  Tracks: AXElements, PIDs, WindowIDs    │                 │
│  │  Ref-counted, thread-safe               │                 │
│  └──────────────┬──────────────────────────┘                 │
│                 │                                             │
│  ┌──────────────▼──────────────────────────┐                 │
│  │  Core Modules                           │                 │
│  │                                         │                 │
│  │  ax.rs   input.rs   capture.rs          │                 │
│  │  apps.rs  clipboard.rs  permissions.rs  │                 │
│  └─────────────────────────────────────────┘                 │
│                 │                                             │
│                 ▼                                             │
│  macOS: ApplicationServices, CoreGraphics, AppKit            │
└──────────────────────────────────────────────────────────────┘
```

---

## Rust: Handle Table

The central design change from the flat FFI approach. The Rust side maintains
a table of live objects that Deno holds references to.

```rust
// handles.rs

use std::collections::HashMap;
use std::sync::Mutex;
use std::sync::atomic::{AtomicU64, Ordering};

static NEXT_HANDLE: AtomicU64 = AtomicU64::new(1);
static HANDLE_TABLE: Mutex<Option<HashMap<u64, HandleEntry>>> = Mutex::new(None);

pub enum HandleEntry {
    /// An AXUIElement reference (for windows, elements, menus)
    AXElement(crate::ax::AXElement),

    /// An application (PID + AXUIElement for the app)
    App {
        pid: i32,
        bundle_id: String,
        name: String,
        ax: crate::ax::AXElement,
    },

    /// A window (CGWindowID + AX element + owning app PID)
    Window {
        window_id: u32,
        pid: i32,
        ax: crate::ax::AXElement,
    },
}

fn table() -> std::sync::MutexGuard<'static, Option<HashMap<u64, HandleEntry>>> {
    let mut guard = HANDLE_TABLE.lock().unwrap();
    if guard.is_none() {
        *guard = Some(HashMap::new());
    }
    guard
}

/// Insert a new entry, return its handle ID.
pub fn insert(entry: HandleEntry) -> u64 {
    let id = NEXT_HANDLE.fetch_add(1, Ordering::Relaxed);
    table().as_mut().unwrap().insert(id, entry);
    id
}

/// Get a reference to an entry by handle.
pub fn get(handle: u64) -> Option<std::sync::MutexGuard<'static, Option<HashMap<u64, HandleEntry>>>> {
    let guard = table();
    if guard.as_ref().unwrap().contains_key(&handle) {
        Some(guard)
    } else {
        None
    }
}

/// Remove and drop an entry.
pub fn release(handle: u64) {
    table().as_mut().unwrap().remove(&handle);
}

/// Release all handles (cleanup).
pub fn release_all() {
    table().as_mut().unwrap().clear();
}
```

### Why handles matter

Without handles, every operation requires re-querying the AX tree:

```typescript
// BAD: re-walks the tree every call
const buttons = await mac_find_elements_by_role(pid, "AXButton", 10);
// Returns JSON snapshots — the elements are gone. You get data, not references.
// To click, you need coordinates. If the UI shifted, coordinates are stale.
```

With handles, the Rust side holds a live `AXUIElement` reference:

```typescript
// GOOD: holds a reference
const btn = await win.find({ role: "AXButton", title: "OK" });
// btn._handle = 43, pointing at a live AXUIElement in Rust

// Can query its *current* position at click time
await btn.click();
// Rust: reads btn's current frame from AX, clicks center of it
// Works even if the UI moved since the query
```

---

## Rust: Handle-Based FFI Surface

```rust
// ffi.rs — Casper's extern "C" surface, handle-oriented

use crate::handles;
use std::ptr;

// ================================================================
// Buffer helpers (same as before)
// ================================================================

#[unsafe(no_mangle)]
pub extern "C" fn casper_free_buffer(ptr: *mut u8, len: u64) { /* ... */ }

fn vec_to_ffi(data: Vec<u8>, out_len: *mut u64) -> *mut u8 { /* ... */ }
fn json_to_ffi<T: serde::Serialize>(value: &T, out_len: *mut u64) -> *mut u8 { /* ... */ }
unsafe fn str_from_buf(ptr: *const u8, len: u32) -> &'static str { /* ... */ }

// ================================================================
// Lifecycle
// ================================================================

/// Release a handle. Frees the Rust-side object.
#[unsafe(no_mangle)]
pub extern "C" fn casper_release(handle: u64) {
    handles::release(handle);
}

/// Release all handles. Call on shutdown.
#[unsafe(no_mangle)]
pub extern "C" fn casper_release_all() {
    handles::release_all();
}

// ================================================================
// App
// ================================================================

/// List all running GUI apps. Returns JSON array, each with a `handle` field.
/// Each app is inserted into the handle table.
#[unsafe(no_mangle)]
pub extern "C" fn casper_app_all(out_len: *mut u64) -> *mut u8 {
    let apps = crate::apps::list_applications();
    let result: Vec<serde_json::Value> = apps.into_iter().map(|app| {
        let ax = crate::ax::application(app.pid);
        ax.set_timeout(10.0);
        let handle = handles::insert(handles::HandleEntry::App {
            pid: app.pid,
            bundle_id: app.bundle_id.clone().unwrap_or_default(),
            name: app.name.clone().unwrap_or_default(),
            ax,
        });
        serde_json::json!({
            "handle": handle,
            "pid": app.pid,
            "bundleId": app.bundle_id,
            "name": app.name,
            "isActive": app.is_active,
            "isHidden": app.is_hidden,
        })
    }).collect();
    json_to_ffi(&result, out_len)
}

/// Get the frontmost app. Returns JSON with handle.
#[unsafe(no_mangle)]
pub extern "C" fn casper_app_frontmost(out_len: *mut u64) -> *mut u8 {
    match crate::apps::frontmost_application() {
        Some(app) => {
            let ax = crate::ax::application(app.pid);
            ax.set_timeout(10.0);
            let handle = handles::insert(handles::HandleEntry::App {
                pid: app.pid,
                bundle_id: app.bundle_id.clone().unwrap_or_default(),
                name: app.name.clone().unwrap_or_default(),
                ax,
            });
            let value = serde_json::json!({
                "handle": handle,
                "pid": app.pid,
                "bundleId": app.bundle_id,
                "name": app.name,
            });
            json_to_ffi(&value, out_len)
        }
        None => { unsafe { *out_len = 0; } ptr::null_mut() }
    }
}

/// Launch an app by bundle ID. Returns JSON with handle.
#[unsafe(no_mangle)]
pub extern "C" fn casper_app_launch(
    bundle_id: *const u8, bundle_id_len: u32,
    out_len: *mut u64,
) -> *mut u8 {
    let id = unsafe { str_from_buf(bundle_id, bundle_id_len) };
    if crate::apps::launch_app(id).is_err() {
        unsafe { *out_len = 0; }
        return ptr::null_mut();
    }
    // Wait briefly for launch, then query
    std::thread::sleep(std::time::Duration::from_millis(500));
    casper_app_find(bundle_id, bundle_id_len, out_len)
}

/// Find a running app by bundle ID or name. Returns JSON with handle.
#[unsafe(no_mangle)]
pub extern "C" fn casper_app_find(
    query: *const u8, query_len: u32,
    out_len: *mut u64,
) -> *mut u8 {
    let q = unsafe { str_from_buf(query, query_len) };
    let apps = crate::apps::list_applications();
    let found = apps.into_iter().find(|a| {
        a.bundle_id.as_deref() == Some(q)
            || a.name.as_deref().map(|n| n.eq_ignore_ascii_case(q)).unwrap_or(false)
    });
    match found {
        Some(app) => {
            let ax = crate::ax::application(app.pid);
            ax.set_timeout(10.0);
            let handle = handles::insert(handles::HandleEntry::App {
                pid: app.pid,
                bundle_id: app.bundle_id.clone().unwrap_or_default(),
                name: app.name.clone().unwrap_or_default(),
                ax,
            });
            let value = serde_json::json!({
                "handle": handle,
                "pid": app.pid,
                "bundleId": app.bundle_id,
                "name": app.name,
            });
            json_to_ffi(&value, out_len)
        }
        None => { unsafe { *out_len = 0; } ptr::null_mut() }
    }
}

/// Activate an app by handle. Returns 0 on success.
#[unsafe(no_mangle)]
pub extern "C" fn casper_app_activate(handle: u64) -> i32 {
    let table = handles::table();
    let map = table.as_ref().unwrap();
    match map.get(&handle) {
        Some(handles::HandleEntry::App { bundle_id, .. }) => {
            match crate::apps::activate_app(bundle_id) {
                Ok(()) => 0,
                Err(_) => -1,
            }
        }
        _ => -1,
    }
}

/// Quit an app by handle. force: 0=graceful, 1=force.
#[unsafe(no_mangle)]
pub extern "C" fn casper_app_quit(handle: u64, force: u8) -> i32 {
    let table = handles::table();
    let map = table.as_ref().unwrap();
    match map.get(&handle) {
        Some(handles::HandleEntry::App { bundle_id, .. }) => {
            match crate::apps::quit_app(bundle_id, force != 0) {
                Ok(_) => { drop(table); handles::release(handle); 0 }
                Err(_) => -1,
            }
        }
        _ => -1,
    }
}

/// Get windows for an app handle. Returns JSON array with window handles.
#[unsafe(no_mangle)]
pub extern "C" fn casper_app_windows(handle: u64, out_len: *mut u64) -> *mut u8 {
    let table = handles::table();
    let map = table.as_ref().unwrap();
    match map.get(&handle) {
        Some(handles::HandleEntry::App { pid, ax, .. }) => {
            let windows = ax.windows();
            let result: Vec<serde_json::Value> = windows.into_iter().map(|w| {
                let title = w.title();
                let (x, y, width, height) = w.frame().unwrap_or((0.0, 0.0, 0.0, 0.0));
                let win_handle = handles::insert(handles::HandleEntry::Window {
                    window_id: 0, // resolved via AXWindowResolver if needed
                    pid: *pid,
                    ax: w,
                });
                serde_json::json!({
                    "handle": win_handle,
                    "title": title,
                    "bounds": { "x": x, "y": y, "width": width, "height": height },
                })
            }).collect();
            json_to_ffi(&result, out_len)
        }
        _ => { unsafe { *out_len = 0; } ptr::null_mut() }
    }
}

// ================================================================
// Window
// ================================================================

/// Focus a window by handle.
#[unsafe(no_mangle)]
pub extern "C" fn casper_window_focus(handle: u64) -> i32 {
    let table = handles::table();
    let map = table.as_ref().unwrap();
    match map.get(&handle) {
        Some(handles::HandleEntry::Window { ax, .. }) => {
            match ax.perform_action("AXRaise") {
                Ok(()) => 0,
                Err(_) => -1,
            }
        }
        _ => -1,
    }
}

/// Close a window by handle.
#[unsafe(no_mangle)]
pub extern "C" fn casper_window_close(handle: u64) -> i32 {
    // Press the close button via AX
    let table = handles::table();
    let map = table.as_ref().unwrap();
    match map.get(&handle) {
        Some(handles::HandleEntry::Window { ax, .. }) => {
            // AXCloseButton attribute → perform AXPress
            if let Some(btn) = ax.ax_attr_element("AXCloseButton") {
                match btn.perform_action("AXPress") {
                    Ok(()) => 0,
                    Err(_) => -1,
                }
            } else { -1 }
        }
        _ => -1,
    }
}

/// Capture a window as PNG by handle. Returns bytes pointer.
#[unsafe(no_mangle)]
pub extern "C" fn casper_window_capture(handle: u64, out_len: *mut u64) -> *mut u8 {
    let table = handles::table();
    let map = table.as_ref().unwrap();
    match map.get(&handle) {
        Some(handles::HandleEntry::Window { window_id, .. }) if *window_id != 0 => {
            match crate::capture::capture_window(*window_id) {
                Ok(png) => vec_to_ffi(png, out_len),
                Err(_) => { unsafe { *out_len = 0; } ptr::null_mut() }
            }
        }
        _ => { unsafe { *out_len = 0; } ptr::null_mut() }
    }
}

/// Find elements within a window. query_json is a JSON ElementQuery.
/// Returns JSON array of elements with handles.
#[unsafe(no_mangle)]
pub extern "C" fn casper_window_find_all(
    handle: u64,
    query_json: *const u8, query_len: u32,
    out_len: *mut u64,
) -> *mut u8 {
    let query_str = unsafe { str_from_buf(query_json, query_len) };
    let query: ElementQuery = match serde_json::from_str(query_str) {
        Ok(q) => q,
        Err(_) => { unsafe { *out_len = 0; } return ptr::null_mut(); }
    };

    let table = handles::table();
    let map = table.as_ref().unwrap();
    let ax = match map.get(&handle) {
        Some(handles::HandleEntry::Window { ax, .. }) => ax,
        _ => { unsafe { *out_len = 0; } return ptr::null_mut(); }
    };

    let matches = ax.find_all(12, &|elem| query.matches(elem));

    let result: Vec<serde_json::Value> = matches.into_iter().map(|e| {
        let (x, y, w, h) = e.frame().unwrap_or((0.0, 0.0, 0.0, 0.0));
        let role = e.role();
        let title = e.title();
        let label = e.label();
        let value = e.value();
        let elem_handle = handles::insert(handles::HandleEntry::AXElement(e));
        serde_json::json!({
            "handle": elem_handle,
            "role": role,
            "title": title,
            "label": label,
            "value": value,
            "bounds": { "x": x, "y": y, "width": w, "height": h },
        })
    }).collect();
    json_to_ffi(&result, out_len)
}

// ================================================================
// Element
// ================================================================

/// Click an element by handle. Reads current position from AX.
#[unsafe(no_mangle)]
pub extern "C" fn casper_element_click(handle: u64, click_type: u8) -> i32 {
    let table = handles::table();
    let map = table.as_ref().unwrap();
    let ax = match map.get(&handle) {
        Some(handles::HandleEntry::AXElement(ax)) => ax,
        _ => return -1,
    };

    // Read the element's current frame from AX (not cached)
    let (x, y, w, h) = match ax.frame() {
        Some(f) => f,
        None => return -1,
    };
    let center_x = x + w / 2.0;
    let center_y = y + h / 2.0;

    drop(table); // release lock before input

    let button = crate::input::MouseButton::Left;
    let count = match click_type {
        1 => 2, // double
        2 => { // right click
            return match crate::input::click(center_x, center_y,
                crate::input::MouseButton::Right, 1) {
                Ok(()) => 0, Err(_) => -1,
            };
        }
        _ => 1, // single
    };
    match crate::input::click(center_x, center_y, button, count) {
        Ok(()) => 0,
        Err(_) => -1,
    }
}

/// Type text into an element. Focuses it first via AXFocused.
#[unsafe(no_mangle)]
pub extern "C" fn casper_element_type(
    handle: u64,
    text: *const u8, text_len: u32,
    delay_ms: u64,
) -> i32 {
    let text_str = unsafe { str_from_buf(text, text_len) };

    // Focus the element first
    {
        let table = handles::table();
        let map = table.as_ref().unwrap();
        match map.get(&handle) {
            Some(handles::HandleEntry::AXElement(ax)) => {
                let _ = ax.perform_action("AXFocus");
            }
            _ => return -1,
        }
    }

    // Brief pause for focus to take effect
    std::thread::sleep(std::time::Duration::from_millis(50));

    match crate::input::type_text(text_str, delay_ms) {
        Ok(()) => 0,
        Err(_) => -1,
    }
}

/// Get current properties of an element. Returns JSON.
#[unsafe(no_mangle)]
pub extern "C" fn casper_element_props(handle: u64, out_len: *mut u64) -> *mut u8 {
    let table = handles::table();
    let map = table.as_ref().unwrap();
    match map.get(&handle) {
        Some(handles::HandleEntry::AXElement(ax)) => {
            let (x, y, w, h) = ax.frame().unwrap_or((0.0, 0.0, 0.0, 0.0));
            let value = serde_json::json!({
                "role": ax.role(),
                "title": ax.title(),
                "label": ax.label(),
                "value": ax.value(),
                "bounds": { "x": x, "y": y, "width": w, "height": h },
            });
            json_to_ffi(&value, out_len)
        }
        _ => { unsafe { *out_len = 0; } ptr::null_mut() }
    }
}

/// Find children of an element matching a query. Returns JSON with handles.
#[unsafe(no_mangle)]
pub extern "C" fn casper_element_find_all(
    handle: u64,
    query_json: *const u8, query_len: u32,
    out_len: *mut u64,
) -> *mut u8 {
    // Same pattern as casper_window_find_all but scoped to element's subtree
    // ...
    todo!()
}

// ================================================================
// Input (stateless — no handles)
// ================================================================

#[unsafe(no_mangle)]
pub extern "C" fn casper_keyboard_type(
    text: *const u8, text_len: u32, delay_ms: u64,
) -> i32 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_keyboard_hotkey(
    keys: *const u8, keys_len: u32, hold_ms: u64,
) -> i32 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_mouse_click(
    x: f64, y: f64, button: u8, count: u32,
) -> i32 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_mouse_move(x: f64, y: f64) -> i32 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_mouse_drag(
    from_x: f64, from_y: f64, to_x: f64, to_y: f64,
    steps: u32, delay_ms: u64,
) -> i32 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_mouse_scroll(dx: i32, dy: i32) -> i32 { /* ... */ }

// ================================================================
// Screen capture (stateless)
// ================================================================

#[unsafe(no_mangle)]
pub extern "C" fn casper_screen_capture(
    display_id: u32, out_len: *mut u64,
) -> *mut u8 { /* ... */ }

// ================================================================
// Clipboard (stateless)
// ================================================================

#[unsafe(no_mangle)]
pub extern "C" fn casper_clipboard_read(out_len: *mut u64) -> *mut u8 { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn casper_clipboard_write(
    text: *const u8, text_len: u32,
) -> i32 { /* ... */ }

// ================================================================
// Permissions (stateless)
// ================================================================

#[unsafe(no_mangle)]
pub extern "C" fn casper_permissions_check(out_len: *mut u64) -> *mut u8 { /* ... */ }

// ================================================================
// ElementQuery — used by find operations
// ================================================================

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
        if let Some(ref value) = self.value {
            if elem.value().as_deref() != Some(value.as_str()) { return false; }
        }
        // identifier, enabled checks...
        true
    }
}
```

---

## TypeScript: Entity Classes

### casper/ffi/symbols.ts — Deno.dlopen

```typescript
const LIB_PATH = new URL(
  "../../target/release/libcasper.dylib",
  import.meta.url,
);

const lib = Deno.dlopen(LIB_PATH, {
  // Lifecycle
  casper_free_buffer: { parameters: ["pointer", "u64"], result: "void" },
  casper_release: { parameters: ["u64"], result: "void" },
  casper_release_all: { parameters: [], result: "void" },

  // App
  casper_app_all: { parameters: ["pointer"], result: "pointer" },
  casper_app_frontmost: { parameters: ["pointer"], result: "pointer" },
  casper_app_launch: { parameters: ["buffer", "u32", "pointer"], result: "pointer" },
  casper_app_find: { parameters: ["buffer", "u32", "pointer"], result: "pointer" },
  casper_app_activate: { parameters: ["u64"], result: "i32" },
  casper_app_quit: { parameters: ["u64", "u8"], result: "i32" },
  casper_app_windows: { parameters: ["u64", "pointer"], result: "pointer" },

  // Window
  casper_window_focus: { parameters: ["u64"], result: "i32" },
  casper_window_close: { parameters: ["u64"], result: "i32" },
  casper_window_capture: { parameters: ["u64", "pointer"], result: "pointer" },
  casper_window_find_all: { parameters: ["u64", "buffer", "u32", "pointer"], result: "pointer" },

  // Element
  casper_element_click: { parameters: ["u64", "u8"], result: "i32" },
  casper_element_type: { parameters: ["u64", "buffer", "u32", "u64"], result: "i32" },
  casper_element_props: { parameters: ["u64", "pointer"], result: "pointer" },
  casper_element_find_all: { parameters: ["u64", "buffer", "u32", "pointer"], result: "pointer" },

  // Keyboard
  casper_keyboard_type: { parameters: ["buffer", "u32", "u64"], result: "i32" },
  casper_keyboard_hotkey: { parameters: ["buffer", "u32", "u64"], result: "i32" },

  // Mouse
  casper_mouse_click: { parameters: ["f64", "f64", "u8", "u32"], result: "i32" },
  casper_mouse_move: { parameters: ["f64", "f64"], result: "i32" },
  casper_mouse_drag: { parameters: ["f64", "f64", "f64", "f64", "u32", "u64"], result: "i32" },
  casper_mouse_scroll: { parameters: ["i32", "i32"], result: "i32" },

  // Screen
  casper_screen_capture: { parameters: ["u32", "pointer"], result: "pointer" },

  // Clipboard
  casper_clipboard_read: { parameters: ["pointer"], result: "pointer" },
  casper_clipboard_write: { parameters: ["buffer", "u32"], result: "i32" },

  // Permissions
  casper_permissions_check: { parameters: ["pointer"], result: "pointer" },
});

export default lib;
export const sym = lib.symbols;
```

### casper/ffi/handles.ts — Handle Lifecycle

```typescript
import { sym } from "./symbols.ts";

/** Base class for any entity that holds a Rust handle. */
export class Handle implements Disposable {
  readonly _handle: number;
  #released = false;

  constructor(handle: number) {
    this._handle = handle;
  }

  /** Release the Rust-side object. */
  dispose(): void {
    if (!this.#released) {
      sym.casper_release(BigInt(this._handle));
      this.#released = true;
    }
  }

  /** For `using` keyword support. */
  [Symbol.dispose](): void {
    this.dispose();
  }
}
```

### casper/entities/app.ts

```typescript
import { sym } from "../ffi/symbols.ts";
import { callJson, encodeStr, assertOk } from "../ffi/helpers.ts";
import { Handle } from "../ffi/handles.ts";
import { Window } from "./window.ts";

interface AppData {
  handle: number;
  pid: number;
  bundleId: string | null;
  name: string | null;
  isActive?: boolean;
  isHidden?: boolean;
}

export class App extends Handle {
  readonly pid: number;
  readonly bundleId: string;
  readonly name: string;

  private constructor(data: AppData) {
    super(data.handle);
    this.pid = data.pid;
    this.bundleId = data.bundleId ?? "";
    this.name = data.name ?? "";
  }

  // --- Static factories ---

  static async all(): Promise<App[]> {
    const apps = callJson<AppData[]>((out) => sym.casper_app_all(out));
    return (apps ?? []).map((d) => new App(d));
  }

  static async frontmost(): Promise<App> {
    const data = callJson<AppData>((out) => sym.casper_app_frontmost(out));
    if (!data) throw new Error("No frontmost application");
    return new App(data);
  }

  static async find(query: string): Promise<App> {
    const [buf, len] = encodeStr(query);
    const data = callJson<AppData>((out) =>
      sym.casper_app_find(buf, len, out)
    );
    if (!data) throw new Error(`App not found: ${query}`);
    return new App(data);
  }

  static async launch(bundleId: string): Promise<App> {
    const [buf, len] = encodeStr(bundleId);
    const data = callJson<AppData>((out) =>
      sym.casper_app_launch(buf, len, out)
    );
    if (!data) throw new Error(`Failed to launch: ${bundleId}`);
    return new App(data);
  }

  // --- Instance methods ---

  async activate(): Promise<void> {
    assertOk(sym.casper_app_activate(BigInt(this._handle)), "activate");
  }

  async quit(force = false): Promise<void> {
    assertOk(
      sym.casper_app_quit(BigInt(this._handle), force ? 1 : 0),
      "quit",
    );
  }

  async hide(): Promise<void> {
    // ... delegate to FFI
  }

  async windows(): Promise<Window[]> {
    const wins = callJson<Window.Data[]>((out) =>
      sym.casper_app_windows(BigInt(this._handle), out)
    );
    return (wins ?? []).map((d) => new Window(d, this));
  }

  async focusedWindow(): Promise<Window> {
    const wins = await this.windows();
    return wins[0]; // first window is typically the focused one
  }
}
```

### casper/entities/window.ts

```typescript
import { sym } from "../ffi/symbols.ts";
import { callJson, callBytes, encodeStr, assertOk } from "../ffi/helpers.ts";
import { Handle } from "../ffi/handles.ts";
import { Element, type ElementQuery } from "./element.ts";
import type { App } from "./app.ts";
import type { Rect } from "../types.ts";

export class Window extends Handle {
  readonly title: string;
  readonly app: App;
  readonly bounds: Rect;

  /** @internal */
  constructor(
    data: { handle: number; title: string | null; bounds: Rect },
    app: App,
  ) {
    super(data.handle);
    this.title = data.title ?? "";
    this.app = app;
    this.bounds = data.bounds;
  }

  async focus(): Promise<void> {
    assertOk(sym.casper_window_focus(BigInt(this._handle)), "focus");
  }

  async close(): Promise<void> {
    assertOk(sym.casper_window_close(BigInt(this._handle)), "close");
  }

  async capture(): Promise<Uint8Array> {
    const data = callBytes((out) =>
      sym.casper_window_capture(BigInt(this._handle), out)
    );
    if (!data) throw new Error("Window capture failed");
    return data;
  }

  async findAll(query: ElementQuery): Promise<Element[]> {
    const [buf, len] = encodeStr(JSON.stringify(query));
    const elements = callJson<Element.Data[]>((out) =>
      sym.casper_window_find_all(BigInt(this._handle), buf, len, out)
    );
    return (elements ?? []).map((d) => new Element(d));
  }

  async find(query: ElementQuery): Promise<Element | null> {
    const all = await this.findAll(query);
    return all[0] ?? null;
  }

  /** Wait for an element matching the query to appear. Polls with timeout. */
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

### casper/entities/element.ts

```typescript
import { sym } from "../ffi/symbols.ts";
import { callJson, encodeStr, assertOk } from "../ffi/helpers.ts";
import { Handle } from "../ffi/handles.ts";
import type { Rect } from "../types.ts";

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

  /** @internal */
  constructor(data: Element.Data) {
    super(data.handle);
    this.role = data.role;
    this.title = data.title;
    this.label = data.label;
    this.value = data.value;
    this.bounds = data.bounds;
  }

  /** Re-read current properties from the live AX element. */
  async refresh(): Promise<Element.Props> {
    return callJson<Element.Props>((out) =>
      sym.casper_element_props(BigInt(this._handle), out)
    )!;
  }

  async click(): Promise<void> {
    assertOk(sym.casper_element_click(BigInt(this._handle), 0), "click");
  }

  async doubleClick(): Promise<void> {
    assertOk(sym.casper_element_click(BigInt(this._handle), 1), "doubleClick");
  }

  async rightClick(): Promise<void> {
    assertOk(sym.casper_element_click(BigInt(this._handle), 2), "rightClick");
  }

  async type(text: string, delayMs = 50): Promise<void> {
    const [buf, len] = encodeStr(text);
    assertOk(
      sym.casper_element_type(BigInt(this._handle), buf, len, BigInt(delayMs)),
      "type",
    );
  }

  async children(): Promise<Element[]> {
    const [buf, len] = encodeStr("{}"); // empty query = all children
    const elements = callJson<Element.Data[]>((out) =>
      sym.casper_element_find_all(BigInt(this._handle), buf, len, out)
    );
    return (elements ?? []).map((d) => new Element(d));
  }

  async findAll(query: ElementQuery): Promise<Element[]> {
    const [buf, len] = encodeStr(JSON.stringify(query));
    const elements = callJson<Element.Data[]>((out) =>
      sym.casper_element_find_all(BigInt(this._handle), buf, len, out)
    );
    return (elements ?? []).map((d) => new Element(d));
  }

  async find(query: ElementQuery): Promise<Element | null> {
    const all = await this.findAll(query);
    return all[0] ?? null;
  }

  toString(): string {
    return `Element(${this.role} "${this.title ?? this.label ?? ""}")`;
  }
}

export namespace Element {
  export interface Data {
    handle: number;
    role: string | null;
    title: string | null;
    label: string | null;
    value: string | null;
    bounds: Rect;
  }
  export interface Props {
    role: string | null;
    title: string | null;
    label: string | null;
    value: string | null;
    bounds: Rect;
  }
}
```

### casper/entities/keyboard.ts

```typescript
import { sym } from "../ffi/symbols.ts";
import { encodeStr, assertOk } from "../ffi/helpers.ts";

/** Global keyboard — no handle, stateless. */
export const Keyboard = {
  async type(text: string, delayMs = 50): Promise<void> {
    const [buf, len] = encodeStr(text);
    assertOk(
      sym.casper_keyboard_type(buf, len, BigInt(delayMs)),
      "Keyboard.type",
    );
  },

  async hotkey(keys: string, holdMs = 0): Promise<void> {
    const [buf, len] = encodeStr(keys);
    assertOk(
      sym.casper_keyboard_hotkey(buf, len, BigInt(holdMs)),
      "Keyboard.hotkey",
    );
  },

  async press(key: string): Promise<void> {
    await this.hotkey(key);
  },
};
```

### casper/entities/mouse.ts

```typescript
import { sym } from "../ffi/symbols.ts";
import { assertOk } from "../ffi/helpers.ts";
import type { Point } from "../types.ts";

/** Global mouse — no handle, stateless. */
export const Mouse = {
  async click(point: Point, button: "left" | "right" = "left", count = 1): Promise<void> {
    const btn = button === "right" ? 1 : 0;
    assertOk(sym.casper_mouse_click(point.x, point.y, btn, count), "Mouse.click");
  },

  async doubleClick(point: Point): Promise<void> {
    await this.click(point, "left", 2);
  },

  async rightClick(point: Point): Promise<void> {
    await this.click(point, "right");
  },

  async move(point: Point): Promise<void> {
    assertOk(sym.casper_mouse_move(point.x, point.y), "Mouse.move");
  },

  async drag(from: Point, to: Point, steps = 20, stepDelayMs = 10): Promise<void> {
    assertOk(
      sym.casper_mouse_drag(from.x, from.y, to.x, to.y, steps, BigInt(stepDelayMs)),
      "Mouse.drag",
    );
  },

  async scroll(direction: "up" | "down" | "left" | "right", amount = 3): Promise<void> {
    const [dx, dy] = {
      up: [0, amount],
      down: [0, -amount],
      left: [amount, 0],
      right: [-amount, 0],
    }[direction];
    assertOk(sym.casper_mouse_scroll(dx, dy), "Mouse.scroll");
  },
};
```

### casper/entities/screen.ts

```typescript
import { sym } from "../ffi/symbols.ts";
import { callBytes } from "../ffi/helpers.ts";

export const Screen = {
  async capture(displayId = 0): Promise<Uint8Array> {
    const data = callBytes((out) => sym.casper_screen_capture(displayId, out));
    if (!data) throw new Error("Screen capture failed");
    return data;
  },
};
```

### casper/entities/clipboard.ts

```typescript
import { sym } from "../ffi/symbols.ts";
import { callBytes, encodeStr, assertOk } from "../ffi/helpers.ts";

export const Clipboard = {
  read(): string | null {
    const data = callBytes((out) => sym.casper_clipboard_read(out));
    if (!data) return null;
    return new TextDecoder().decode(data);
  },

  write(text: string): void {
    const [buf, len] = encodeStr(text);
    assertOk(sym.casper_clipboard_write(buf, len), "Clipboard.write");
  },

  clear(): void {
    this.write("");
  },
};
```

### casper/mod.ts — Public API

```typescript
// Casper — entity-based Mac automation for Deno

export { App } from "./entities/app.ts";
export { Window } from "./entities/window.ts";
export { Element, type ElementQuery } from "./entities/element.ts";
export { Keyboard } from "./entities/keyboard.ts";
export { Mouse } from "./entities/mouse.ts";
export { Screen } from "./entities/screen.ts";
export { Clipboard } from "./entities/clipboard.ts";
export type { Point, Rect, Size } from "./types.ts";

import lib from "./ffi/symbols.ts";

/** Shut down Casper. Releases all handles and closes the dylib. */
export function shutdown(): void {
  lib.symbols.casper_release_all();
  lib.close();
}
```

### casper/types.ts

```typescript
export interface Point { x: number; y: number }
export interface Size { width: number; height: number }
export interface Rect { x: number; y: number; width: number; height: number }
```

---

## Usage: Agent Loop

```typescript
// agent.ts
import { App, Window, Screen, Keyboard, Mouse, Clipboard, shutdown } from "./casper/mod.ts";

// Check what's running
const apps = await App.all();
console.log(`${apps.length} apps running`);

// Find Safari, or launch it
let safari: App;
try {
  safari = await App.find("Safari");
} catch {
  safari = await App.launch("com.apple.Safari");
}
await safari.activate();

// Get the main window
const win = await safari.focusedWindow();
console.log(`Window: "${win.title}"`);

// Screenshot the window
const png = await win.capture();
await Deno.writeFile("/tmp/safari.png", png);

// Find the URL bar and type into it
const urlBar = await win.find({ role: "AXTextField", identifier: "WEB_BROWSER_ADDRESS_AND_SEARCH_FIELD" });
if (urlBar) {
  await urlBar.click();
  await Keyboard.hotkey("cmd+a");
  await Keyboard.type("https://example.com");
  await Keyboard.press("return");
}

// Wait for page to load, then find links
await new Promise(r => setTimeout(r, 2000));
const links = await win.findAll({ role: "AXLink" });
console.log(`Found ${links.length} links`);

for (const link of links.slice(0, 3)) {
  console.log(`  ${link.title} → (${link.bounds.x}, ${link.bounds.y})`);
}

// Click the first link
if (links[0]) {
  await links[0].click();
}

// Cleanup: release all AX handles
apps.forEach(a => a.dispose());
shutdown();
```

---

## Usage: File Operations via Finder

```typescript
import { App, Keyboard } from "./casper/mod.ts";

const finder = await App.find("Finder");
await finder.activate();

const win = await finder.focusedWindow();

// Navigate using Go → Go to Folder
await Keyboard.hotkey("cmd+shift+g");
const dialog = await win.waitFor({ role: "AXSheet" }, 3000);
const pathField = await dialog.find({ role: "AXTextField" });
await pathField!.type("/Users/me/Documents");
await Keyboard.press("return");

// Select a file
const files = await win.findAll({ role: "AXRow" });
if (files[0]) {
  await files[0].click();
  // Open it
  await Keyboard.hotkey("cmd+o");
}
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
│   └── permissions.rs            # TCC checks
└── deno/                         # TypeScript
    ├── mod.ts                    # public API re-exports
    ├── types.ts                  # Point, Rect, Size
    ├── ffi/
    │   ├── symbols.ts            # Deno.dlopen + symbol defs
    │   ├── handles.ts            # Handle base class
    │   └── helpers.ts            # pointer/buffer utils
    └── entities/
        ├── app.ts                # App entity
        ├── window.ts             # Window entity
        ├── element.ts            # Element entity (AX)
        ├── keyboard.ts           # Keyboard singleton
        ├── mouse.ts              # Mouse singleton
        ├── screen.ts             # Screen singleton
        ├── clipboard.ts          # Clipboard singleton
        ├── browser.ts            # Browser extends App
        ├── tab.ts                # Tab entity
        ├── finder.ts             # Finder extends App
        ├── file.ts               # File entity
        └── dialog.ts             # Dialog entity
```

---

## Entity Classification

| Entity | Handle type | Rust state | Stateless? |
|---|---|---|---|
| **App** | PID + AXUIElement | `HandleEntry::App` | No |
| **Window** | CGWindowID + AXUIElement | `HandleEntry::Window` | No |
| **Element** | AXUIElement | `HandleEntry::AXElement` | No |
| **Tab** | AXUIElement (tab element) | `HandleEntry::AXElement` | No |
| **Dialog** | AXUIElement (sheet/dialog) | `HandleEntry::AXElement` | No |
| **Keyboard** | — | — | Yes |
| **Mouse** | — | — | Yes |
| **Screen** | — | — | Yes |
| **Clipboard** | — | — | Yes |
| **File** | path string | (no Rust state) | TS-only |
| **Finder** | Inherits from App | `HandleEntry::App` | No |
| **Browser** | Inherits from App | `HandleEntry::App` | No |
| **Script** | — | — | Yes |
| **Snapshot** | ref→handle map | (TS-only, holds Element refs) | No |

---

## How Handles Flow

```
1. App.find("Safari")
   → casper_app_find("Safari")
   → Rust: finds NSRunningApplication, creates AXUIElement
   → Rust: inserts HandleEntry::App { pid, ax, ... } → handle=1
   → Returns JSON: { "handle": 1, "pid": 12345, "name": "Safari" }
   → TS: new App({ handle: 1, pid: 12345, name: "Safari" })

2. app.windows()
   → casper_app_windows(handle=1)
   → Rust: reads handle 1 → app.ax.windows()
   → Rust: inserts HandleEntry::Window for each → handles 2, 3
   → Returns JSON: [{ "handle": 2, "title": "Google" }, ...]
   → TS: [new Window({ handle: 2, title: "Google" }, app)]

3. window.find({ role: "AXButton", title: "Submit" })
   → casper_window_find_all(handle=2, '{"role":"AXButton","title":"Submit"}')
   → Rust: reads handle 2 → window.ax.find_all(query)
   → Rust: inserts HandleEntry::AXElement for match → handle 4
   → Returns JSON: [{ "handle": 4, "role": "AXButton", "title": "Submit", ... }]
   → TS: new Element({ handle: 4, ... })

4. element.click()
   → casper_element_click(handle=4)
   → Rust: reads handle 4 → element.frame() → gets CURRENT position
   → Rust: CGEventPost(click at center of element)
   → Returns 0

5. element.dispose()
   → casper_release(handle=4)
   → Rust: removes handle 4, drops AXUIElement
```

---

## AppleScript as a Complementary Control Plane

> Inspired by [spotify-cli-mac](https://github.com/ersel/spotify-cli-mac), which
> uses AppleScript for all local playback control and the Spotify Web API for
> search/discovery — choosing the fastest, most reliable mechanism per operation.

### Why both AX and AppleScript?

Accessibility APIs and AppleScript scripting dictionaries are **complementary,
not competing**:

| | Accessibility (AX) | AppleScript |
|---|---|---|
| **What it gives you** | UI structure — buttons, text fields, their positions | Semantic verbs — `play track`, `set volume`, `make new document` |
| **Works on** | Every app with a GUI (even unsigned) | Only apps that ship a `.sdef` scripting dictionary |
| **Reliability** | Depends on AX tree quality (Electron can be messy) | Very stable for apps that support it |
| **Speed** | Must walk the element tree | Direct command dispatch, no tree walking |
| **Best for** | UI discovery, clicking arbitrary elements, reading layout | App-specific actions, state queries, batch operations |

Many macOS apps expose rich scripting dictionaries: Spotify, Music, Safari,
Finder, Mail, Messages, Calendar, Reminders, Terminal, Keynote, Pages, Numbers,
etc. For these apps, AppleScript gives **semantic actions** that are 10x more
reliable than groping through AX elements.

### The Script entity

`Script` is a stateless entity (no handles) that executes AppleScript via the
Rust layer, which calls `NSAppleScript` or `OSAScript`:

```typescript
export const Script = {
  /**
   * Execute a command inside `tell application "appName"`.
   *
   * @param appName - Application name (e.g. "Spotify", "Safari")
   * @param command - AppleScript command(s) to run inside the tell block
   * @param opts.launchIfNeeded - Launch the app first if not running (default: false)
   * @returns The string result from AppleScript, or null if no return value
   */
  async tell(appName: string, command: string, opts?: { launchIfNeeded?: boolean }): Promise<string | null> {
    // → casper_script_tell("Spotify", "play track \"spotify:track:xxx\"", launch)
    // Rust: wraps in `tell application "Spotify" to <command>`, executes via NSAppleScript
  },

  /**
   * Execute a full AppleScript source string. Returns the result parsed as
   * a JSON-compatible record when possible.
   */
  async eval(source: string): Promise<Record<string, unknown> | string | null> {
    // → casper_script_eval(source)
    // Rust: NSAppleScript executeAndReturnError
  },

  /**
   * Check whether an application has a scripting dictionary (.sdef).
   */
  async canScript(appName: string): Promise<boolean> {
    // → casper_script_can_script("Spotify")
    // Rust: check for .sdef in app bundle
  },
};
```

### Rust FFI surface

```rust
/// Execute `tell application "<app>" to <command>`.
/// Returns the AppleScript result string (or null pointer on failure).
#[unsafe(no_mangle)]
pub extern "C" fn casper_script_tell(
    app: *const u8, app_len: u32,
    command: *const u8, command_len: u32,
    launch_if_needed: u8,
    out_len: *mut u64,
) -> *mut u8 {
    let app_name = unsafe { str_from_buf(app, app_len) };
    let cmd = unsafe { str_from_buf(command, command_len) };

    if launch_if_needed != 0 {
        // Check NSWorkspace for running app, launch if absent
        let _ = crate::apps::launch_app_by_name(app_name);
        std::thread::sleep(std::time::Duration::from_millis(500));
    }

    let script = format!("tell application \"{}\" to {}", app_name, cmd);
    match crate::scripting::execute(&script) {
        Ok(result) => match result {
            Some(s) => vec_to_ffi(s.into_bytes(), out_len),
            None => { unsafe { *out_len = 0; } ptr::null_mut() }
        },
        Err(_) => { unsafe { *out_len = 0; } ptr::null_mut() }
    }
}

/// Execute a full AppleScript source. Returns JSON-encoded result.
#[unsafe(no_mangle)]
pub extern "C" fn casper_script_eval(
    source: *const u8, source_len: u32,
    out_len: *mut u64,
) -> *mut u8 { /* ... */ }

/// Check if an app has a scripting dictionary.
#[unsafe(no_mangle)]
pub extern "C" fn casper_script_can_script(
    app: *const u8, app_len: u32,
) -> u8 { /* 1 = yes, 0 = no */ }
```

### When to use Script vs AX

```
Agent decides to control Spotify:

  1. Is Spotify scriptable?
     → Script.canScript("Spotify") → true

  2. Use Script for semantic actions:
     → Script.tell("Spotify", "play track \"spotify:track:xxx\"")
     → Script.tell("Spotify", "set sound volume to 75")
     → Script.tell("Spotify", "next track")

  3. Use AX only when you need UI structure:
     → e.g., reading the layout of the "Now Playing" view for a screenshot
     → or clicking a custom UI element that has no scripting verb

Agent decides to control a random Electron app:

  1. Is it scriptable?
     → Script.canScript("SomeElectronApp") → false

  2. Fall back to AX entirely:
     → App.find("SomeElectronApp") → windows → findAll → click
```

---

## App Profiles

App profiles are bundled metadata about well-known applications — their
scripting verbs, AX landmarks, and keyboard shortcuts. They let agents skip
exploratory AX tree walking for common operations.

```typescript
// casper/profiles/types.ts
export interface AppProfile {
  bundleId: string;
  name: string;
  scriptable: boolean;

  /** AppleScript verb templates. Keyed by action name. */
  verbs?: Record<string, string | ((...args: unknown[]) => string)>;

  /** Well-known keyboard shortcuts. */
  shortcuts?: Record<string, string>;

  /** AppleScript expressions that return state. */
  queries?: Record<string, string>;

  /** AX element landmarks — known identifiers/roles for key UI elements. */
  landmarks?: Record<string, ElementQuery>;
}
```

### Example: Spotify

```typescript
// casper/profiles/spotify.ts
import type { AppProfile } from "./types.ts";

export const spotify: AppProfile = {
  bundleId: "com.spotify.client",
  name: "Spotify",
  scriptable: true,

  verbs: {
    play: (uri?: string) => uri ? `play track "${uri}"` : "play",
    pause: "pause",
    toggle: "playpause",
    next: "next track",
    previous: "previous track",
    volume: (n: number) => `set sound volume to ${n}`,
    seek: (seconds: number) => `set player position to ${seconds}`,
    shuffle: "set shuffling to not shuffling",
    repeat: "set repeating to not repeating",
  },

  shortcuts: {
    search: "cmd+k",
    preferences: "cmd+,",
    newPlaylist: "cmd+n",
  },

  queries: {
    nowPlaying: `
      set a to artist of current track
      set t to name of current track
      set al to album of current track
      set d to duration of current track
      set p to player position
      set s to player state as string
      set u to spotify url of current track
      set art to artwork url of current track
      return {artist:a, track:t, album:al, duration:d, position:p, state:s, url:u, artwork:art}
    `,
    isPlaying: "return player state is playing",
    currentVolume: "return sound volume",
    isShuffling: "return shuffling",
    isRepeating: "return repeating",
  },

  landmarks: {
    searchField: { role: "AXTextField", identifier: "search-input" },
    playButton: { role: "AXButton", label: "Play" },
    nowPlayingBar: { role: "AXGroup", identifier: "now-playing-bar" },
  },
};
```

### Example: Safari

```typescript
// casper/profiles/safari.ts
import type { AppProfile } from "./types.ts";

export const safari: AppProfile = {
  bundleId: "com.apple.Safari",
  name: "Safari",
  scriptable: true,

  verbs: {
    openUrl: (url: string) => `open location "${url}"`,
    currentUrl: "return URL of current tab of front window",
    currentTitle: "return name of current tab of front window",
    newTab: (url?: string) => url
      ? `tell front window to set current tab to (make new tab with properties {URL:"${url}"})`
      : "tell front window to make new tab",
    closeTab: "close current tab of front window",
    listTabs: `
      set tabList to {}
      tell front window
        repeat with t in tabs
          set end of tabList to {name of t, URL of t}
        end repeat
      end tell
      return tabList
    `,
  },

  shortcuts: {
    addressBar: "cmd+l",
    newTab: "cmd+t",
    closeTab: "cmd+w",
    reload: "cmd+r",
    back: "cmd+[",
    forward: "cmd+]",
  },

  landmarks: {
    addressBar: { role: "AXTextField", identifier: "WEB_BROWSER_ADDRESS_AND_SEARCH_FIELD" },
    webContent: { role: "AXWebArea" },
  },
};
```

### Example: Music

```typescript
// casper/profiles/music.ts
import type { AppProfile } from "./types.ts";

export const music: AppProfile = {
  bundleId: "com.apple.Music",
  name: "Music",
  scriptable: true,

  verbs: {
    play: "play",
    pause: "pause",
    toggle: "playpause",
    next: "next track",
    previous: "back track",
    volume: (n: number) => `set sound volume to ${n}`,
    search: (query: string) => `search playlist "Library" for "${query}"`,
  },

  queries: {
    nowPlaying: `
      set a to artist of current track
      set t to name of current track
      set al to album of current track
      set d to duration of current track
      set p to player position
      return {artist:a, track:t, album:al, duration:d, position:p}
    `,
  },
};
```

### Profile registry

```typescript
// casper/profiles/mod.ts
import { spotify } from "./spotify.ts";
import { safari } from "./safari.ts";
import { music } from "./music.ts";
import type { AppProfile } from "./types.ts";

const profiles = new Map<string, AppProfile>();

function register(profile: AppProfile): void {
  profiles.set(profile.bundleId, profile);
  profiles.set(profile.name.toLowerCase(), profile);
}

register(spotify);
register(safari);
register(music);

/** Look up a profile by bundle ID or app name. */
export function getProfile(appOrBundleId: string): AppProfile | undefined {
  return profiles.get(appOrBundleId) ?? profiles.get(appOrBundleId.toLowerCase());
}
```

Profiles are **optional hints**, not requirements. If an agent has a profile,
it can take the fast path. Without one, it falls back to AX exploration.

---

## Hybrid Control: Script + Web API + AX

Real-world automation often combines multiple control planes. Casper provides
the local planes (AX and Script); external API calls come from outside (e.g.,
Tachikoma providers or direct `fetch` calls in agent code).

### Pattern: search remotely, play locally

```typescript
// Agent-level code: Tachikoma handles the Spotify Web API
const results = await tachikoma.call("spotify-search", { query: "Bohemian Rhapsody" });
const track = results[0]; // { uri: "spotify:track:4uLU6hMCjMI75M1A2tKUQC", name: "..." }

// Casper handles local playback — no UI fumbling needed
await Script.tell("Spotify", `play track "${track.uri}"`, { launchIfNeeded: true });
```

This is **dramatically more reliable** than automating the Spotify search UI:

| Approach | Steps | Failure modes |
|---|---|---|
| AX-only | Launch → find search field → type → wait → find result row → click | AX tree mismatch, wrong result clicked, timing issues |
| Script-only | `tell Spotify to play track "uri"` | Need to know the URI already |
| Hybrid (Web API + Script) | API search → `tell Spotify to play track "uri"` | Network for search, but local play is rock-solid |

### Pattern: read state via Script, act via AX

```typescript
// Fast state read via AppleScript
const state = await Script.eval(`
  tell application "Spotify"
    return {playerState:player state as string, track:name of current track}
  end tell
`);

if (state.playerState === "paused") {
  // Need to click a specific UI element? Use AX.
  const spotify = await App.find("Spotify");
  const win = await spotify.focusedWindow();
  const customBtn = await win.find({ role: "AXButton", identifier: "some-custom-ui" });
  await customBtn?.click();
}
```

### Pattern: script with fallback to AX

```typescript
async function playTrack(uri: string): Promise<void> {
  if (await Script.canScript("Spotify")) {
    // Fast path: AppleScript
    await Script.tell("Spotify", `play track "${uri}"`, { launchIfNeeded: true });
  } else {
    // Fallback: UI automation
    const spotify = await App.launch("com.spotify.client");
    await spotify.activate();
    const win = await spotify.focusedWindow();
    await Keyboard.hotkey("cmd+k");
    const search = await win.waitFor({ role: "AXTextField" }, 3000);
    await search.type(uri);
    await Keyboard.press("return");
  }
}
```

---

## Web Content Extensions

Browser-hosted apps (Twitter/X, Gmail, Slack web) present challenges that
native apps don't: deep AX trees, dynamic content, and elements behind an
`AXWebArea` boundary.

### Window.findInPage() — crossing the AXWebArea boundary

Browser windows have an AX structure like:

```
AXWindow → AXSplitGroup → AXGroup → AXWebArea → (all page content here)
```

Standard `findAll()` searches the full subtree, but `findInPage()` makes
intent explicit and could apply web-specific optimizations:

```typescript
export class Window extends Handle {
  /** Find elements within the web content area (crosses AXWebArea boundary). */
  async findInPage(query: ElementQuery): Promise<Element[]> {
    const webArea = await this.find({ role: "AXWebArea" });
    if (!webArea) return [];
    return webArea.findAll(query);
  }

  /** Wait for an element within web content to appear. */
  async waitForInPage(query: ElementQuery, timeoutMs = 10000): Promise<Element> {
    const start = Date.now();
    while (Date.now() - start < timeoutMs) {
      const results = await this.findInPage(query);
      if (results[0]) return results[0];
      await new Promise(r => setTimeout(r, 300));
    }
    throw new Error(`Timed out waiting for web element: ${JSON.stringify(query)}`);
  }
}
```

### Extended ElementQuery for web content

Web content elements often lack clean titles but have DOM-specific AX
attributes. The query interface should support these:

```typescript
export interface ElementQuery {
  // Existing — works for all AX elements
  role?: string;
  title?: string;
  titleContains?: string;
  label?: string;
  value?: string;
  identifier?: string;
  enabled?: boolean;

  // New — especially useful for web content and Electron apps
  description?: string;           // AXDescription (exact)
  descriptionContains?: string;   // AXDescription (substring)
  valueContains?: string;         // AXValue (substring)
  roleDescription?: string;       // e.g. "link", "button", "heading", "article"
  url?: string;                   // AXLink URL attribute (exact)
  urlContains?: string;           // AXLink URL attribute (substring)
  domId?: string;                 // AXDOMIdentifier
  domClass?: string;              // matches within AXDOMClassList
}
```

### Usage: opening a tweet

```typescript
const safari = await Browser.findOrLaunch("com.apple.Safari");

// Use Script for navigation (faster than AX for scriptable browsers)
await Script.tell("Safari", 'open location "https://x.com/user/status/123"');

const win = await safari.focusedWindow();

// Wait for the tweet to render in web content
const tweet = await win.waitForInPage(
  { role: "AXGroup", roleDescription: "article" },
  10000,
);

// Interact with elements inside the tweet
const likeBtn = await tweet.find({ role: "AXButton", label: "Like" });
if (likeBtn) await likeBtn.click();

// Or find a specific link within the page
const link = await win.findInPage({
  role: "AXLink",
  urlContains: "/status/123",
});
```

### Rust-side ElementQuery additions

```rust
#[derive(serde::Deserialize)]
struct ElementQuery {
    // existing fields...
    role: Option<String>,
    title: Option<String>,
    #[serde(rename = "titleContains")]
    title_contains: Option<String>,
    label: Option<String>,
    value: Option<String>,
    identifier: Option<String>,
    enabled: Option<bool>,

    // new fields for web content
    description: Option<String>,
    #[serde(rename = "descriptionContains")]
    description_contains: Option<String>,
    #[serde(rename = "valueContains")]
    value_contains: Option<String>,
    #[serde(rename = "roleDescription")]
    role_description: Option<String>,
    url: Option<String>,
    #[serde(rename = "urlContains")]
    url_contains: Option<String>,
    #[serde(rename = "domId")]
    dom_id: Option<String>,
    #[serde(rename = "domClass")]
    dom_class: Option<String>,
}

impl ElementQuery {
    fn matches(&self, elem: &crate::ax::AXElement) -> bool {
        // ... existing checks ...

        if let Some(ref desc) = self.description {
            if elem.description().as_deref() != Some(desc.as_str()) { return false; }
        }
        if let Some(ref contains) = self.description_contains {
            match elem.description() {
                Some(d) if d.contains(contains.as_str()) => {}
                _ => return false,
            }
        }
        if let Some(ref contains) = self.value_contains {
            match elem.value() {
                Some(v) if v.contains(contains.as_str()) => {}
                _ => return false,
            }
        }
        if let Some(ref rd) = self.role_description {
            if elem.role_description().as_deref() != Some(rd.as_str()) { return false; }
        }
        if let Some(ref url) = self.url {
            if elem.ax_attr_string("AXURL").as_deref() != Some(url.as_str()) { return false; }
        }
        if let Some(ref contains) = self.url_contains {
            match elem.ax_attr_string("AXURL") {
                Some(u) if u.contains(contains.as_str()) => {}
                _ => return false,
            }
        }
        if let Some(ref dom_id) = self.dom_id {
            if elem.ax_attr_string("AXDOMIdentifier").as_deref() != Some(dom_id.as_str()) { return false; }
        }
        if let Some(ref dom_class) = self.dom_class {
            match elem.ax_attr_string("AXDOMClassList") {
                Some(classes) if classes.contains(dom_class.as_str()) => {}
                _ => return false,
            }
        }
        true
    }
}
```

---

## Semantic Snapshots

> Inspired by [OpenClaw](https://github.com/openclaw/openclaw)'s browser tool,
> which parses the accessibility tree into structured text with element
> references instead of sending screenshots to the LLM — ~100x cheaper in
> tokens and more precise than pixel-coordinate guessing.

### The problem

An agent controlling a GUI needs to **see** what's on screen, then **act** on
what it sees. There are three approaches:

| Approach | Token cost | Precision | Handles UI changes? |
|---|---|---|---|
| **Screenshot** | ~1,500 tokens per image (base64) | Coordinate guessing from pixels | No — stale coordinates |
| **Full AX dump** | Hundreds of elements, 5k+ tokens | Exact, but overwhelming | No — data snapshot |
| **Semantic snapshot** | Compact text, ~200-500 tokens | Exact ref→handle mapping | Yes — refs hold live handles |

Semantic snapshots are the sweet spot: compact enough for LLM context, precise
enough for direct action, and backed by live handles so refs track UI changes.

### The Snapshot entity

`Window.snapshot()` walks the AX tree and produces a `Snapshot` — a text
representation with numbered refs, plus a map from ref numbers to live
`Element` handles:

```typescript
interface SnapshotOpts {
  /** Maximum tree depth to walk (default: 10) */
  maxDepth?: number;
  /** Only snapshot web content (AXWebArea subtree) */
  webContentOnly?: boolean;
  /** Include element bounds in output (default: false) */
  includeBounds?: boolean;
  /** Roles to skip (e.g. ["AXGroup", "AXGenericElement"] to reduce noise) */
  skipRoles?: string[];
}

export class Snapshot implements Disposable {
  /** Compact text representation of the AX tree. */
  readonly text: string;

  /** Map from ref number to live Element handle. */
  readonly refs: Map<number, Element>;

  /** Click an element by its ref number. */
  async click(ref: number): Promise<void> {
    const el = this.refs.get(ref);
    if (!el) throw new Error(`Unknown ref: ${ref}`);
    await el.click();
  }

  /** Type text into an element by its ref number. */
  async type(ref: number, text: string): Promise<void> {
    const el = this.refs.get(ref);
    if (!el) throw new Error(`Unknown ref: ${ref}`);
    await el.type(text);
  }

  /** Get the Element handle for a ref. */
  get(ref: number): Element {
    const el = this.refs.get(ref);
    if (!el) throw new Error(`Unknown ref: ${ref}`);
    return el;
  }

  /** Release all Element handles held by this snapshot. */
  dispose(): void {
    for (const el of this.refs.values()) {
      el.dispose();
    }
    this.refs.clear();
  }

  [Symbol.dispose](): void {
    this.dispose();
  }
}
```

### Text format

The snapshot text is a compact, indented tree with one line per meaningful
element. Each actionable element gets a `[ref=N]` tag:

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
      cell "Daily Mix 2" [ref=8]
      cell "Release Radar" [ref=9]
    scrollbar vertical [ref=10]
```

Formatting rules:
- Indent = nesting depth (2 spaces per level)
- Skip structural-only groups that add no information
- Include `value=` for sliders, text fields, checkboxes
- Include `level=` for headings
- Only assign refs to actionable elements (buttons, links, text fields,
  sliders, cells, checkboxes, tabs)
- Non-actionable text (statictext, headings) shown inline without refs
- Roles are lowercased without the "AX" prefix for readability

### Rust FFI surface

```rust
/// Generate a semantic snapshot of a window's AX tree.
/// Returns JSON: { "text": "...", "refs": { "1": handle, "2": handle, ... } }
#[unsafe(no_mangle)]
pub extern "C" fn casper_window_snapshot(
    handle: u64,
    max_depth: u32,
    web_content_only: u8,
    include_bounds: u8,
    skip_roles_json: *const u8, skip_roles_len: u32,
    out_len: *mut u64,
) -> *mut u8 {
    let table = handles::table();
    let map = table.as_ref().unwrap();
    let ax = match map.get(&handle) {
        Some(handles::HandleEntry::Window { ax, .. }) => ax,
        _ => { unsafe { *out_len = 0; } return ptr::null_mut(); }
    };

    let root = if web_content_only != 0 {
        // Find the first AXWebArea descendant
        match ax.find_first(10, &|e| e.role().as_deref() == Some("AXWebArea")) {
            Some(web_area) => web_area,
            None => { unsafe { *out_len = 0; } return ptr::null_mut(); }
        }
    } else {
        ax.clone()
    };

    let mut text = String::new();
    let mut refs: HashMap<u32, u64> = HashMap::new();
    let mut next_ref: u32 = 1;

    fn walk(
        elem: &crate::ax::AXElement,
        depth: u32, max_depth: u32,
        text: &mut String, refs: &mut HashMap<u32, u64>,
        next_ref: &mut u32,
        include_bounds: bool,
    ) {
        if depth > max_depth { return; }

        let role = elem.role().unwrap_or_default();
        let display_role = role.strip_prefix("AX").unwrap_or(&role).to_lowercase();
        let title = elem.title();
        let value = elem.value();
        let indent = "  ".repeat(depth as usize);

        let is_actionable = matches!(
            role.as_str(),
            "AXButton" | "AXLink" | "AXTextField" | "AXTextArea"
            | "AXSlider" | "AXCheckBox" | "AXRadioButton"
            | "AXPopUpButton" | "AXComboBox" | "AXCell"
            | "AXTab" | "AXMenuItem" | "AXImage"
            | "AXIncrementor" | "AXDisclosureTriangle"
        );

        // Build the line
        let mut line = format!("{}{}", indent, display_role);
        if let Some(ref t) = title {
            if !t.is_empty() { line.push_str(&format!(" \"{}\"", t)); }
        }
        if let Some(ref v) = value {
            if !v.is_empty() { line.push_str(&format!(" value={}", v)); }
        }

        if is_actionable {
            let ref_id = *next_ref;
            *next_ref += 1;
            let elem_handle = handles::insert(handles::HandleEntry::AXElement(elem.clone()));
            refs.insert(ref_id, elem_handle);
            line.push_str(&format!(" [ref={}]", ref_id));
        }

        if include_bounds {
            if let Some((x, y, w, h)) = elem.frame() {
                line.push_str(&format!(" @({:.0},{:.0} {:.0}x{:.0})", x, y, w, h));
            }
        }

        text.push_str(&line);
        text.push('\n');

        // Recurse into children
        for child in elem.children() {
            walk(&child, depth + 1, max_depth, text, refs, next_ref, include_bounds);
        }
    }

    walk(&root, 0, max_depth, &mut text, &mut refs, &mut next_ref, include_bounds != 0);

    let result = serde_json::json!({
        "text": text,
        "refs": refs,
    });
    json_to_ffi(&result, out_len)
}
```

### Usage in an agent loop

```typescript
import { App, Keyboard, shutdown } from "./casper/mod.ts";

const spotify = await App.find("Spotify");
await spotify.activate();
const win = await spotify.focusedWindow();

// Take a semantic snapshot — much cheaper than a screenshot
{
  using snap = await win.snapshot();

  // Send snap.text to the LLM as context (~300 tokens vs ~1,500 for an image):
  //
  // window "Spotify" [ref=1]
  //   group "Now Playing Bar"
  //     statictext "Bohemian Rhapsody"
  //     statictext "Queen"
  //     button "Pause" [ref=4]
  //     button "Next" [ref=5]
  //     slider "Volume" value=75 [ref=6]

  // LLM responds: "click ref=5 to skip to next track"
  await snap.click(5);

  // Or get the element for more complex interaction:
  const volumeSlider = snap.get(6);
  const props = await volumeSlider.refresh();
  console.log(`Volume: ${props.value}`);

} // snap.dispose() called — all ref handles released
```

### Snapshots vs screenshots

Snapshots don't replace screenshots — they complement them. An agent might:

1. Take a **snapshot** for structured understanding (~300 tokens, actionable)
2. Take a **screenshot** for visual verification (~1,500 tokens, read-only)
3. Use snapshot refs to **act** without coordinate math

```typescript
// Snapshot for decision-making
const snap = await win.snapshot();
// → Send snap.text to LLM: "I see a Login button [ref=3] and a Sign Up link [ref=4]"

// Screenshot for visual confirmation (optional)
const png = await win.capture();
// → Send to multimodal LLM: "Verify the page looks correct"

// Act using snapshot refs (no coordinates needed)
await snap.click(3);  // Click "Login"
```

### Web content snapshots

For browser-hosted apps, use `webContentOnly` to skip browser chrome and focus
on the page:

```typescript
const safari = await App.find("Safari");
const win = await safari.focusedWindow();

// Snapshot only the web page content
const snap = await win.snapshot({ webContentOnly: true });
// →
// heading "Twitter / X" level=1
//   group article [ref=1]
//     link "@elonmusk" [ref=2]
//     statictext "This is a tweet"
//     button "Reply" [ref=3]
//     button "Retweet" [ref=4]
//     button "Like" [ref=5]
//   group article [ref=6]
//     ...

await snap.click(5); // Like the tweet
```

---

## Comparison: Casper vs OpenClaw

> [OpenClaw](https://github.com/openclaw/openclaw) (formerly Clawdbot, by Peter
> Steinberger, 40k+ GitHub stars) is an AI agent gateway that bridges messaging
> apps to LLMs with the ability to act on your machine. OpenClaw actually uses
> Peekaboo as its macOS GUI automation layer — Casper would replace/upgrade the
> automation substrate that OpenClaw sits on top of.

### Architecture comparison

| | **OpenClaw** | **Casper** |
|---|---|---|
| **What it is** | AI agent gateway / orchestrator | Native automation engine |
| **Core language** | Node.js / TypeScript | Rust (engine) + TypeScript (API) |
| **How it controls Mac** | Delegates to Peekaboo CLI, AppleScript MCP, CDP | Direct AX handles, AppleScript, CGEvent — all in-process |
| **Automation model** | CLI subprocess calls (`peekaboo see`, `peekaboo click`) | Entity methods (`element.click()`, `win.find()`) |
| **State model** | Stateless — every command re-queries the world | Stateful handles — hold live references to AX elements |
| **Type safety** | Skills are markdown instructions for the LLM | Full TypeScript types — `App`, `Window`, `Element`, `ElementQuery` |
| **Performance** | Process spawn per action + JSON parsing | FFI call per action, handles avoid re-walking AX tree |
| **Browser control** | CDP + Semantic Snapshots (ARIA tree → text) | AX tree via `findInPage()` + semantic snapshots |
| **Knowledge system** | 53 bundled + 5,700 community skills (SKILL.md) | App Profiles (typed, bundled) |

### What we borrowed from OpenClaw

**1. Semantic Snapshots** — OpenClaw's browser tool parses the ARIA
accessibility tree into compact text with element references (`button "Sign In"
[ref=1]`) instead of sending screenshots. The agent says "click ref=1" — exact,
cheap, no coordinate guessing. We adopted this as `Window.snapshot()`, with the
key difference that Casper's refs map to **live Element handles** (not stale
data), so they track UI changes.

**2. Three-tier lazy loading** — OpenClaw avoids prompt bloat by loading skill
knowledge progressively: name + description first (~30 tokens), then full
instructions, then deep reference files. Our App Profiles follow the same
pattern:

```typescript
export interface AppProfile {
  // Tier 1: always loaded (tiny) — name, bundleId, description, scriptable
  // Tier 2: loaded when agent targets this app — verbs, shortcuts, landmarks
  // Tier 3: loaded on demand — referenceUrl, exampleFlows
}
```

**3. Permission brokering awareness** — OpenClaw solves macOS TCC
(Transparency, Consent, and Control) by having a signed GUI app own all
permissions, brokering requests over a Unix socket. Casper should account for
running in contexts without TCC grants by supporting PeekabooBridge fallback.

**4. `exec` escape hatch** — Sometimes `osascript -e '...'` or `open -a
Spotify` is the right call. A lightweight `System.exec()` escape hatch
prevents over-engineering entity wrappers for one-off operations.

### What makes Casper different

**1. Live handles vs stateless CLI calls** — OpenClaw shells out to `peekaboo
click --on B3`, spawning a process that walks the AX tree, finds the element,
clicks, and exits. Every action starts from scratch. Casper's `element.click()`
reads the live AXUIElement's current frame from a Rust handle table — the
element was found once and held. This matters for multi-step interactions where
UI shifts between actions.

**2. Composable typed API vs interpreted markdown** — OpenClaw skills are prose
instructions the LLM follows ("to search Spotify, run `peekaboo click --on
<ref>`"). Casper is typed code that catches errors at write time, enables IDE
completion, and composes with standard TypeScript control flow.

**3. In-process FFI vs subprocess IPC** — OpenClaw shells out for every
automation action — process spawn overhead, JSON serialization, stdout parsing.
Casper calls Rust functions directly through Deno's FFI — microsecond overhead,
zero serialization for handle-based operations.

**4. Multi-plane coherence** — OpenClaw bolts on AppleScript via a separate MCP
server (`macos-automator-mcp`). Casper integrates Script, AX, and
keyboard/mouse as peer entities in a single API — the agent doesn't need to
know which MCP server to call.

### Summary: what to borrow, what to keep

| Borrow from OpenClaw | Keep in Casper |
|---|---|
| Semantic Snapshots — AX tree → text with refs | Live handles — snapshot refs map to real Element handles |
| Three-tier lazy loading for profiles | Typed profiles — TypeScript interfaces, not markdown |
| Permission brokering awareness for daemon contexts | In-process FFI — no subprocess overhead |
| `exec` escape hatch for shell commands | Multi-plane coherence — Script, AX, Input as peers |
| Skill ecosystem idea — community-contributed profiles | Type-first composability — entities compose with TypeScript |

---

## File Layout (updated)

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
    │   ├── app.ts                # App entity
    │   ├── window.ts             # Window entity
    │   ├── element.ts            # Element entity (AX)
    │   ├── keyboard.ts           # Keyboard singleton
    │   ├── mouse.ts              # Mouse singleton
    │   ├── screen.ts             # Screen singleton
    │   ├── clipboard.ts          # Clipboard singleton
    │   ├── browser.ts            # Browser extends App
    │   ├── tab.ts                # Tab entity
    │   ├── finder.ts             # Finder extends App
    │   ├── file.ts               # File entity
    │   ├── dialog.ts             # Dialog entity
    │   ├── script.ts             # Script singleton (AppleScript)
    │   └── snapshot.ts           # Snapshot entity (AX tree → text + refs)
    └── profiles/
        ├── types.ts              # AppProfile interface
        ├── mod.ts                # Profile registry
        ├── spotify.ts            # Spotify profile
        ├── safari.ts             # Safari profile
        └── music.ts              # Music profile
```
