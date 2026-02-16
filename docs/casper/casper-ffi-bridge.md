# Casper FFI Bridge: TypeScript ↔ Rust via Deno

> **Foundation layer.** This document covers the FFI plumbing that Casper's
> entity model (see `casper-tech-design.md`) is built on — how TypeScript
> running in Deno calls into a Rust `cdylib` that wraps macOS frameworks.
> Read this first to understand the boundary; read the tech design for the
> higher-level `App` / `Window` / `Element` API that sits on top.

This document walks through building a Mac automation bridge where:

- **TypeScript (Deno)** is the agent orchestrator — it runs the LLM loop,
  decides what to do, and calls into native code for every macOS interaction.
- **Rust** is the native layer — a compiled `cdylib` that links against macOS
  frameworks (Accessibility, CoreGraphics, AppKit) and exposes a C ABI.
- **Deno.dlopen** is the glue — Deno's built-in FFI mechanism loads the Rust
  `.dylib` at runtime and maps its C functions directly into TypeScript.

## What is FFI?

**Foreign Function Interface (FFI)** lets one language call functions written
in another. The key constraint: both sides must agree on a calling convention,
and the C ABI is the universal one. Any language that can produce or consume
C-compatible functions can participate.

In Casper's stack this looks like:

```
TypeScript  ──Deno.dlopen──▶  C ABI  ◀──extern "C"──  Rust
  (agent)                   (contract)              (native)
```

- **Rust** compiles to a `.dylib` (dynamic library) with `extern "C"` entry
  points. The `#[no_mangle]` attribute preserves function names so the
  dynamic linker can find them.
- **Deno** calls `Deno.dlopen(path, symbolDefs)` to load that `.dylib` and
  bind each C symbol to a callable TypeScript function.
- **No serialization for primitives** — `f64`, `i32`, `bool` etc. map directly
  between JS and C. For complex data (strings, JSON, byte arrays), Rust
  allocates a buffer and returns a pointer that Deno reads and then frees.

This is a **single-process** model: the Deno runtime and the Rust `.dylib`
share an address space. macOS TCC permissions (Accessibility, Screen Recording)
granted to the Deno process apply to the Rust code automatically.

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  Deno Process                                                │
│                                                              │
│  agent.ts  ─── orchestrator / LLM loop                      │
│    ↓                                                         │
│  mac-bridge.ts  ─── typed wrappers around FFI symbols        │
│    ↓                                                         │
│  Deno.dlopen("libmacbridge.dylib", symbols)                  │
│    ↓  (C ABI function calls — no serialization)              │
│                                                              │
│  Rust cdylib: libmacbridge.dylib                             │
│    ├── ffi.rs       ─── extern "C" entry points for Deno     │
│    ├── input.rs     ─── CGEvent (mouse, keyboard, scroll)    │
│    ├── ax.rs        ─── AXUIElement (accessibility tree)     │
│    ├── capture.rs   ─── CGWindowList / ScreenCaptureKit      │
│    ├── apps.rs      ─── NSWorkspace / NSRunningApplication   │
│    ├── windows.rs   ─── AX window attrs + CGWindowList       │
│    ├── clipboard.rs ─── NSPasteboard                         │
│    ├── permissions.rs── TCC permission checks                │
│    └── screen.rs    ─── NSScreen display enumeration         │
└──────────────────────────────────────────────────────────────┘
         ↓ linked at build time ↓
┌──────────────────────────────────────────────────────────────┐
│  macOS Frameworks                                            │
│                                                              │
│  ApplicationServices.framework                               │
│    ├── AXUIElement*  (Accessibility)                         │
│    └── CGEvent*      (Quartz Event Services)                 │
│  CoreGraphics.framework                                      │
│    ├── CGWindowList* (window enumeration + legacy capture)   │
│    ├── CGDisplay*    (display info + legacy capture)         │
│    └── CGImage*      (image handling)                        │
│  AppKit.framework                                            │
│    ├── NSWorkspace    (app launch, running apps)             │
│    ├── NSRunningApplication (app control)                    │
│    ├── NSScreen       (display enumeration)                  │
│    ├── NSPasteboard   (clipboard)                            │
│    └── NSEvent        (mouse location)                       │
│  CoreFoundation.framework                                    │
│    └── CFRunLoop (needed for AX callbacks)                   │
└──────────────────────────────────────────────────────────────┘
```

### Why Deno + Deno.dlopen

- **Built-in FFI** — `Deno.dlopen` is part of the standard runtime. Point it
  at any `.dylib` that exports C functions and get callable TypeScript
  symbols immediately.
- **Built-in TypeScript** — Deno runs `.ts` files directly with top-level
  await, a permissions model, and a standard library. The agent loop is
  plain TypeScript with zero build step.
- **Minimal Rust surface** — the Rust side only needs `extern "C"` +
  `#[no_mangle]` and `cargo build`. No codegen, no binding macros, no
  extra crates for the FFI layer itself.
- **Single process** — the Deno runtime and the Rust `.dylib` share an address
  space. TCC grants (Accessibility, Screen Recording) apply to the single
  Deno process.
- **Zero serialization for primitives** — `Deno.dlopen` maps C types directly
  to JS: `f64`, `i32`, `u32`, `bool`, `pointer`, `buffer`. A call like
  `click(500, 300)` is a direct C function invocation with no encoding.

### How Deno FFI Works

```typescript
// Load the compiled .dylib
const lib = Deno.dlopen("./target/release/libmacbridge.dylib", {
  // Each symbol: { parameters: [...types], result: type }
  mac_click: { parameters: ["f64", "f64", "u8", "u32"], result: "i32" },
  mac_hotkey: { parameters: ["buffer", "u32", "u64"], result: "i32" },
  mac_capture_screen: { parameters: ["pointer"], result: "pointer" },
  mac_free_buffer: { parameters: ["pointer", "u64"], result: "void" },
});

// Call it — Deno passes args directly to the C function
lib.symbols.mac_click(500.0, 300.0, 0, 1); // left click at (500, 300)
```

Deno FFI supports these C-to-JS type mappings:

| FFI type | C type | Rust type | JS type |
|---|---|---|---|
| `"i32"` | `int32_t` | `i32` | `number` |
| `"u32"` | `uint32_t` | `u32` | `number` |
| `"i64"` | `int64_t` | `i64` | `number \| bigint` |
| `"u64"` | `uint64_t` | `u64` | `number \| bigint` |
| `"f32"` | `float` | `f32` | `number` |
| `"f64"` | `double` | `f64` | `number` |
| `"u8"` | `uint8_t` | `u8` | `number` |
| `"bool"` | `bool` | `bool` | `boolean` |
| `"pointer"` | `void*` | `*mut T` | `Deno.PointerObject \| null` |
| `"buffer"` | `void*` | `*const u8` | `Uint8Array` (param only) |
| `"void"` | `void` | `()` | `undefined` |

**Key constraint**: `Deno.dlopen` can only pass/return C primitives and
pointers. For complex data (strings, structs, arrays), the pattern is:
1. Rust allocates and returns a pointer + length
2. Deno reads via `Deno.UnsafePointerView`
3. Deno calls a `free` function to release the Rust allocation

### TCC Permissions

Your Deno process needs these macOS permissions (granted to the terminal
emulator or to a Tauri/.app wrapper):

| Permission | Needed for | System Preferences path |
|---|---|---|
| **Accessibility** | AXUIElement, CGEvent input | Privacy & Security → Accessibility |
| **Screen Recording** | CGWindowListCreateImage, SCScreenshotManager | Privacy & Security → Screen Recording |
| **Automation** (optional) | NSAppleScript, app control via AppleScript | Privacy & Security → Automation |

If you run Deno from Terminal.app or iTerm, the **terminal** needs the grants.
If you wrap Deno in a Tauri app, the **.app bundle** gets the grants.

---

## Part 1: Rust Crate Structure

```
mac-bridge/
├── Cargo.toml
├── build.rs               # link macOS frameworks
├── src/
│   ├── lib.rs             # module declarations
│   ├── ffi.rs             # all extern "C" exports (Deno calls these)
│   ├── ffi_types.rs       # FFI-safe types and buffer helpers
│   ├── input.rs           # CGEvent mouse, keyboard, scroll
│   ├── ax.rs              # AXUIElement accessibility tree
│   ├── capture.rs         # screen capture (CGWindowList)
│   ├── apps.rs            # NSWorkspace / NSRunningApplication
│   ├── clipboard.rs       # NSPasteboard
│   └── permissions.rs     # TCC checks
└── deno/
    ├── deno.json           # Deno project config
    ├── mod.ts              # main entry — Deno.dlopen + symbol defs
    ├── mac_bridge.ts       # high-level typed API
    └── agent.ts            # example agent loop
```

### Cargo.toml

```toml
[package]
name = "mac-bridge"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]    # produces libmacbridge.dylib

[dependencies]
core-foundation = "0.10"
core-graphics = "0.24"
objc2 = "0.6"
objc2-app-kit = { version = "0.3", features = [
    "NSWorkspace", "NSRunningApplication", "NSScreen",
    "NSPasteboard", "NSEvent"
] }
objc2-foundation = { version = "0.3", features = [
    "NSString", "NSArray", "NSDictionary", "NSURL",
    "NSProcessInfo"
] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "2"
png = "0.17"
libc = "0.2"
```

### build.rs — Link macOS Frameworks

```rust
fn main() {
    println!("cargo:rustc-link-lib=framework=ApplicationServices");
    println!("cargo:rustc-link-lib=framework=CoreGraphics");
    println!("cargo:rustc-link-lib=framework=AppKit");
    println!("cargo:rustc-link-lib=framework=CoreFoundation");
}
```

---

## Part 2: FFI Boundary Layer (ffi.rs)

This is the critical file. It's the contract between Deno and Rust. Every
function is `extern "C"` with `#[no_mangle]` and only uses C-safe types.

For complex returns (JSON strings, PNG bytes), the pattern is:
- Rust serializes to JSON or encodes to PNG
- Writes the data into a heap-allocated buffer
- Returns a pointer + writes the length to an out-parameter
- Deno reads the data, then calls `mac_free_buffer` to release it

```rust
// ffi.rs — all functions Deno.dlopen sees

use std::ffi::{CStr, CString};
use std::os::raw::c_char;
use std::ptr;

// =================================================================
// Buffer helpers — how Rust returns variable-length data to Deno
// =================================================================

/// Allocate a byte buffer on the Rust heap. Returns pointer.
/// Deno reads `len` bytes via UnsafePointerView, then calls mac_free_buffer.
#[repr(C)]
pub struct FfiBuffer {
    pub ptr: *mut u8,
    pub len: u64,
}

/// Free a buffer previously returned by a mac_* function.
#[unsafe(no_mangle)]
pub extern "C" fn mac_free_buffer(ptr: *mut u8, len: u64) {
    if ptr.is_null() { return; }
    unsafe {
        let _ = Vec::from_raw_parts(ptr, len as usize, len as usize);
        // Vec drops, memory freed
    }
}

/// Helper: box a Vec<u8> into a pointer+length pair.
/// Writes length to *out_len, returns pointer.
fn vec_to_ffi(data: Vec<u8>, out_len: *mut u64) -> *mut u8 {
    let len = data.len();
    let ptr = data.leak().as_mut_ptr();
    if !out_len.is_null() {
        unsafe { *out_len = len as u64; }
    }
    ptr
}

/// Helper: box a JSON-serializable value into a C string pointer.
fn json_to_ffi<T: serde::Serialize>(value: &T, out_len: *mut u64) -> *mut u8 {
    match serde_json::to_vec(value) {
        Ok(bytes) => vec_to_ffi(bytes, out_len),
        Err(_) => {
            if !out_len.is_null() { unsafe { *out_len = 0; } }
            ptr::null_mut()
        }
    }
}

/// Helper: read a C string pointer from Deno's buffer type.
unsafe fn cstr_from_ptr(ptr: *const u8, len: u32) -> &'static str {
    if ptr.is_null() || len == 0 { return ""; }
    let slice = unsafe { std::slice::from_raw_parts(ptr, len as usize) };
    std::str::from_utf8(slice).unwrap_or("")
}

// =================================================================
// Permissions
// =================================================================

/// Check TCC permissions. Returns JSON: {"accessibility":bool,"screenRecording":bool}
#[unsafe(no_mangle)]
pub extern "C" fn mac_check_permissions(out_len: *mut u64) -> *mut u8 {
    let status = crate::permissions::check_permissions();
    json_to_ffi(&status, out_len)
}

/// Returns 1 if accessibility is trusted, 0 otherwise.
#[unsafe(no_mangle)]
pub extern "C" fn mac_is_accessibility_trusted() -> u8 {
    crate::ax::is_trusted() as u8
}

// =================================================================
// Input — mouse, keyboard, scroll
// =================================================================

/// Click at (x, y). button: 0=left, 1=right. Returns 0 on success.
#[unsafe(no_mangle)]
pub extern "C" fn mac_click(x: f64, y: f64, button: u8, count: u32) -> i32 {
    let btn = if button == 1 {
        crate::input::MouseButton::Right
    } else {
        crate::input::MouseButton::Left
    };
    match crate::input::click(x, y, btn, count) {
        Ok(()) => 0,
        Err(_) => -1,
    }
}

/// Move mouse to (x, y). Returns 0 on success.
#[unsafe(no_mangle)]
pub extern "C" fn mac_mouse_move(x: f64, y: f64) -> i32 {
    match crate::input::mouse_move(x, y) {
        Ok(()) => 0,
        Err(_) => -1,
    }
}

/// Press a hotkey combination. `keys` is a UTF-8 buffer like "cmd+shift+s".
/// `keys_len` is the byte length. Returns 0 on success.
#[unsafe(no_mangle)]
pub extern "C" fn mac_hotkey(
    keys: *const u8, keys_len: u32,
    hold_duration_ms: u64,
) -> i32 {
    let keys_str = unsafe { cstr_from_ptr(keys, keys_len) };
    match crate::input::hotkey(keys_str, hold_duration_ms) {
        Ok(()) => 0,
        Err(_) => -1,
    }
}

/// Type text character by character. `text` is UTF-8 buffer.
/// Returns 0 on success.
#[unsafe(no_mangle)]
pub extern "C" fn mac_type_text(
    text: *const u8, text_len: u32,
    delay_ms: u64,
) -> i32 {
    let text_str = unsafe { cstr_from_ptr(text, text_len) };
    match crate::input::type_text(text_str, delay_ms) {
        Ok(()) => 0,
        Err(_) => -1,
    }
}

/// Scroll. delta_y positive = scroll up, negative = scroll down.
/// Returns 0 on success.
#[unsafe(no_mangle)]
pub extern "C" fn mac_scroll(delta_x: i32, delta_y: i32) -> i32 {
    match crate::input::scroll(delta_x, delta_y) {
        Ok(()) => 0,
        Err(_) => -1,
    }
}

/// Drag from (x0,y0) to (x1,y1) with interpolation steps.
/// Returns 0 on success.
#[unsafe(no_mangle)]
pub extern "C" fn mac_drag(
    from_x: f64, from_y: f64,
    to_x: f64, to_y: f64,
    steps: u32, step_delay_ms: u64,
) -> i32 {
    match crate::input::drag(from_x, from_y, to_x, to_y, steps, step_delay_ms) {
        Ok(()) => 0,
        Err(_) => -1,
    }
}

// =================================================================
// Screen Capture
// =================================================================

/// Capture the main display as PNG. Returns pointer to PNG bytes.
/// Writes byte count to *out_len. Caller must call mac_free_buffer.
#[unsafe(no_mangle)]
pub extern "C" fn mac_capture_screen(display_id: u32, out_len: *mut u64) -> *mut u8 {
    match crate::capture::capture_display(display_id) {
        Ok(png_bytes) => vec_to_ffi(png_bytes, out_len),
        Err(_) => {
            if !out_len.is_null() { unsafe { *out_len = 0; } }
            ptr::null_mut()
        }
    }
}

/// Capture a specific window by CGWindowID. Returns PNG bytes pointer.
/// Caller must call mac_free_buffer.
#[unsafe(no_mangle)]
pub extern "C" fn mac_capture_window(window_id: u32, out_len: *mut u64) -> *mut u8 {
    match crate::capture::capture_window(window_id) {
        Ok(png_bytes) => vec_to_ffi(png_bytes, out_len),
        Err(_) => {
            if !out_len.is_null() { unsafe { *out_len = 0; } }
            ptr::null_mut()
        }
    }
}

// =================================================================
// Applications
// =================================================================

/// List running GUI applications. Returns JSON array pointer.
/// Caller must call mac_free_buffer.
#[unsafe(no_mangle)]
pub extern "C" fn mac_list_applications(out_len: *mut u64) -> *mut u8 {
    let apps = crate::apps::list_applications();
    json_to_ffi(&apps, out_len)
}

/// Get the frontmost application. Returns JSON pointer.
/// Caller must call mac_free_buffer.
#[unsafe(no_mangle)]
pub extern "C" fn mac_frontmost_application(out_len: *mut u64) -> *mut u8 {
    match crate::apps::frontmost_application() {
        Some(app) => json_to_ffi(&app, out_len),
        None => {
            if !out_len.is_null() { unsafe { *out_len = 0; } }
            ptr::null_mut()
        }
    }
}

/// Activate an application by bundle ID. Returns 0 on success.
#[unsafe(no_mangle)]
pub extern "C" fn mac_activate_app(
    bundle_id: *const u8, bundle_id_len: u32,
) -> i32 {
    let id = unsafe { cstr_from_ptr(bundle_id, bundle_id_len) };
    match crate::apps::activate_app(id) {
        Ok(()) => 0,
        Err(_) => -1,
    }
}

/// Launch an application by bundle ID. Returns 0 on success.
#[unsafe(no_mangle)]
pub extern "C" fn mac_launch_app(
    bundle_id: *const u8, bundle_id_len: u32,
) -> i32 {
    let id = unsafe { cstr_from_ptr(bundle_id, bundle_id_len) };
    match crate::apps::launch_app(id) {
        Ok(()) => 0,
        Err(_) => -1,
    }
}

/// Quit an application. force: 0=graceful, 1=force. Returns 0 on success.
#[unsafe(no_mangle)]
pub extern "C" fn mac_quit_app(
    bundle_id: *const u8, bundle_id_len: u32,
    force: u8,
) -> i32 {
    let id = unsafe { cstr_from_ptr(bundle_id, bundle_id_len) };
    match crate::apps::quit_app(id, force != 0) {
        Ok(_) => 0,
        Err(_) => -1,
    }
}

// =================================================================
// Clipboard
// =================================================================

/// Read clipboard text. Returns UTF-8 pointer.
/// Caller must call mac_free_buffer.
#[unsafe(no_mangle)]
pub extern "C" fn mac_read_clipboard(out_len: *mut u64) -> *mut u8 {
    match crate::clipboard::read_clipboard() {
        Some(text) => {
            let bytes = text.into_bytes();
            vec_to_ffi(bytes, out_len)
        }
        None => {
            if !out_len.is_null() { unsafe { *out_len = 0; } }
            ptr::null_mut()
        }
    }
}

/// Write text to clipboard. Returns 0 on success.
#[unsafe(no_mangle)]
pub extern "C" fn mac_write_clipboard(
    text: *const u8, text_len: u32,
) -> i32 {
    let text_str = unsafe { cstr_from_ptr(text, text_len) };
    crate::clipboard::write_clipboard(text_str);
    0
}

// =================================================================
// Accessibility — element queries
// =================================================================

/// Get AX windows for a PID. Returns JSON array pointer.
/// Caller must call mac_free_buffer.
#[unsafe(no_mangle)]
pub extern "C" fn mac_get_windows(pid: i32, out_len: *mut u64) -> *mut u8 {
    let app = crate::ax::application(pid);
    app.set_timeout(10.0);
    let windows: Vec<serde_json::Value> = app.windows().iter().map(|w| {
        let (x, y, width, height) = w.frame().unwrap_or((0.0, 0.0, 0.0, 0.0));
        serde_json::json!({
            "role": w.role(),
            "title": w.title(),
            "label": w.label(),
            "value": w.value(),
            "x": x, "y": y, "width": width, "height": height,
        })
    }).collect();
    json_to_ffi(&windows, out_len)
}

/// Find all AX elements matching a role under the frontmost app.
/// Returns JSON array pointer. Caller must call mac_free_buffer.
#[unsafe(no_mangle)]
pub extern "C" fn mac_find_elements_by_role(
    pid: i32,
    role: *const u8, role_len: u32,
    max_depth: u32,
    out_len: *mut u64,
) -> *mut u8 {
    let role_str = unsafe { cstr_from_ptr(role, role_len) };
    let app = crate::ax::application(pid);
    app.set_timeout(10.0);

    let matches = app.find_all(max_depth as usize, &|elem| {
        elem.role().as_deref() == Some(role_str)
    });

    let elements: Vec<serde_json::Value> = matches.iter().map(|e| {
        let (x, y, w, h) = e.frame().unwrap_or((0.0, 0.0, 0.0, 0.0));
        serde_json::json!({
            "role": e.role(),
            "title": e.title(),
            "label": e.label(),
            "value": e.value(),
            "x": x, "y": y, "width": w, "height": h,
        })
    }).collect();
    json_to_ffi(&elements, out_len)
}

/// Get the element at screen coordinates. Returns JSON pointer.
/// Caller must call mac_free_buffer.
#[unsafe(no_mangle)]
pub extern "C" fn mac_element_at_position(
    pid: i32, x: f32, y: f32,
    out_len: *mut u64,
) -> *mut u8 {
    let app = crate::ax::application(pid);
    match app.element_at_position(x, y) {
        Some(elem) => {
            let (ex, ey, ew, eh) = elem.frame().unwrap_or((0.0, 0.0, 0.0, 0.0));
            let value = serde_json::json!({
                "role": elem.role(),
                "title": elem.title(),
                "label": elem.label(),
                "value": elem.value(),
                "x": ex, "y": ey, "width": ew, "height": eh,
            });
            json_to_ffi(&value, out_len)
        }
        None => {
            if !out_len.is_null() { unsafe { *out_len = 0; } }
            ptr::null_mut()
        }
    }
}

/// Perform an AX action on the element at (x, y) for a given app.
/// action is UTF-8 buffer (e.g. "AXPress"). Returns 0 on success.
#[unsafe(no_mangle)]
pub extern "C" fn mac_perform_action_at(
    pid: i32, x: f32, y: f32,
    action: *const u8, action_len: u32,
) -> i32 {
    let action_str = unsafe { cstr_from_ptr(action, action_len) };
    let app = crate::ax::application(pid);
    match app.element_at_position(x, y) {
        Some(elem) => match elem.perform_action(action_str) {
            Ok(()) => 0,
            Err(_) => -1,
        },
        None => -1,
    }
}
```

---

## Part 3: Rust Core Modules

Each core module wraps a specific set of macOS APIs. They are pure Rust —
they know nothing about Deno or FFI. The `ffi.rs` layer above is the only
file that uses `extern "C"`.

- **input.rs** — `CGEventCreateMouseEvent`, `CGEventCreateKeyboardEvent`,
  `CGEventCreateScrollWheelEvent`, `CGEventPost` via the `core-graphics` crate
- **ax.rs** — `AXUIElementCreateApplication`, `AXUIElementCopyAttributeValue`,
  `AXUIElementPerformAction`, `AXValueCreate` via raw FFI declarations
- **capture.rs** — `CGDisplay::image()`, `CGWindowListCreateImage` via the
  `core-graphics` crate (ScreenCaptureKit via `objc2` later)
- **apps.rs** — `NSWorkspace`, `NSRunningApplication` via `objc2-app-kit`
- **clipboard.rs** — `NSPasteboard` via `objc2-app-kit`
- **permissions.rs** — `CGPreflightScreenCaptureAccess`, `AXIsProcessTrusted`

This separation keeps the native logic testable in Rust independently of the
TypeScript consumer.

---

## Part 4: Deno FFI Client

### deno/mod.ts — Symbol Definitions and dlopen

```typescript
// mod.ts — Load the Rust dylib and define all symbols

const LIB_PATH = new URL(
  "../target/release/libmacbridge.dylib",
  import.meta.url,
);

const lib = Deno.dlopen(LIB_PATH, {
  // --- Buffer management ---
  mac_free_buffer: {
    parameters: ["pointer", "u64"],
    result: "void",
  },

  // --- Permissions ---
  mac_check_permissions: {
    parameters: ["pointer"],  // out_len: *mut u64
    result: "pointer",
  },
  mac_is_accessibility_trusted: {
    parameters: [],
    result: "u8",
  },

  // --- Input ---
  mac_click: {
    parameters: ["f64", "f64", "u8", "u32"],
    result: "i32",
  },
  mac_mouse_move: {
    parameters: ["f64", "f64"],
    result: "i32",
  },
  mac_hotkey: {
    parameters: ["buffer", "u32", "u64"],
    result: "i32",
  },
  mac_type_text: {
    parameters: ["buffer", "u32", "u64"],
    result: "i32",
  },
  mac_scroll: {
    parameters: ["i32", "i32"],
    result: "i32",
  },
  mac_drag: {
    parameters: ["f64", "f64", "f64", "f64", "u32", "u64"],
    result: "i32",
  },

  // --- Capture ---
  mac_capture_screen: {
    parameters: ["u32", "pointer"],  // display_id, out_len
    result: "pointer",
  },
  mac_capture_window: {
    parameters: ["u32", "pointer"],  // window_id, out_len
    result: "pointer",
  },

  // --- Applications ---
  mac_list_applications: {
    parameters: ["pointer"],  // out_len
    result: "pointer",
  },
  mac_frontmost_application: {
    parameters: ["pointer"],  // out_len
    result: "pointer",
  },
  mac_activate_app: {
    parameters: ["buffer", "u32"],
    result: "i32",
  },
  mac_launch_app: {
    parameters: ["buffer", "u32"],
    result: "i32",
  },
  mac_quit_app: {
    parameters: ["buffer", "u32", "u8"],
    result: "i32",
  },

  // --- Clipboard ---
  mac_read_clipboard: {
    parameters: ["pointer"],  // out_len
    result: "pointer",
  },
  mac_write_clipboard: {
    parameters: ["buffer", "u32"],
    result: "i32",
  },

  // --- Accessibility ---
  mac_get_windows: {
    parameters: ["i32", "pointer"],  // pid, out_len
    result: "pointer",
  },
  mac_find_elements_by_role: {
    parameters: ["i32", "buffer", "u32", "u32", "pointer"],
    result: "pointer",
  },
  mac_element_at_position: {
    parameters: ["i32", "f32", "f32", "pointer"],
    result: "pointer",
  },
  mac_perform_action_at: {
    parameters: ["i32", "f32", "f32", "buffer", "u32"],
    result: "i32",
  },
});

export default lib;
export const symbols = lib.symbols;
```

### deno/ffi_helpers.ts — Pointer/Buffer Utilities

```typescript
// ffi_helpers.ts — helpers for reading Rust-allocated data from Deno

import lib from "./mod.ts";

const encoder = new TextEncoder();
const decoder = new TextDecoder();

/**
 * Encode a string to a Uint8Array for passing as FFI "buffer" parameter.
 * Returns [buffer, length] for the two-arg pattern.
 */
export function encodeStr(s: string): [Uint8Array, number] {
  const buf = encoder.encode(s);
  return [buf, buf.byteLength];
}

/**
 * Allocate a u64 out-parameter for Rust to write a length into.
 * Returns [BigUint64Array, pointer-to-it].
 */
export function outLen(): [BigUint64Array, Deno.PointerValue] {
  const buf = new BigUint64Array(1);
  return [buf, Deno.UnsafePointer.of(buf)!];
}

/**
 * Read bytes from a Rust-allocated pointer, then free the Rust buffer.
 * Returns a Uint8Array copy owned by JS.
 */
export function readAndFreeBuffer(
  ptr: Deno.PointerValue,
  lenBuf: BigUint64Array,
): Uint8Array | null {
  if (ptr === null) return null;
  const len = Number(lenBuf[0]);
  if (len === 0) return null;

  const view = new Deno.UnsafePointerView(ptr);
  const copy = new Uint8Array(len);
  view.copyInto(copy);

  // Free the Rust allocation
  lib.symbols.mac_free_buffer(ptr, lenBuf[0]);

  return copy;
}

/**
 * Call a Rust function that returns JSON via pointer+length.
 * Parses and returns the typed result.
 */
export function callJson<T>(
  fn: (outLenPtr: Deno.PointerValue) => Deno.PointerValue,
): T | null {
  const [lenBuf, lenPtr] = outLen();
  const ptr = fn(lenPtr);
  const bytes = readAndFreeBuffer(ptr, lenBuf);
  if (!bytes) return null;
  return JSON.parse(decoder.decode(bytes)) as T;
}

/**
 * Call a Rust function that returns raw bytes via pointer+length.
 * Returns the bytes as a Uint8Array.
 */
export function callBytes(
  fn: (outLenPtr: Deno.PointerValue) => Deno.PointerValue,
): Uint8Array | null {
  const [lenBuf, lenPtr] = outLen();
  const ptr = fn(lenPtr);
  return readAndFreeBuffer(ptr, lenBuf);
}

/**
 * Assert a C function returned 0 (success).
 */
export function assertOk(result: number, op: string): void {
  if (result !== 0) {
    throw new Error(`${op} failed (error code: ${result})`);
  }
}
```

### deno/mac_bridge.ts — High-Level Typed API

```typescript
// mac_bridge.ts — typed wrappers for the Deno FFI bridge

import { symbols } from "./mod.ts";
import { encodeStr, callJson, callBytes, assertOk } from "./ffi_helpers.ts";

// ---- Types ----

export interface Point { x: number; y: number }
export interface Rect  { x: number; y: number; width: number; height: number }

export interface AppInfo {
  pid: number;
  bundle_id: string | null;
  name: string | null;
  is_active: boolean;
  is_hidden: boolean;
}

export interface ElementInfo {
  role: string | null;
  title: string | null;
  label: string | null;
  value: string | null;
  x: number;
  y: number;
  width: number;
  height: number;
}

export interface Permissions {
  accessibility: boolean;
  screen_recording: boolean;
}

// ---- Permissions ----

export function checkPermissions(): Permissions {
  return callJson<Permissions>((outLen) =>
    symbols.mac_check_permissions(outLen)
  )!;
}

export function isAccessibilityTrusted(): boolean {
  return symbols.mac_is_accessibility_trusted() !== 0;
}

// ---- Input ----

export function click(
  point: Point,
  button: "left" | "right" = "left",
  count = 1,
): void {
  const btn = button === "right" ? 1 : 0;
  assertOk(symbols.mac_click(point.x, point.y, btn, count), "click");
}

export function doubleClick(point: Point): void {
  click(point, "left", 2);
}

export function rightClick(point: Point): void {
  click(point, "right", 1);
}

export function mouseMove(point: Point): void {
  assertOk(symbols.mac_mouse_move(point.x, point.y), "mouseMove");
}

export function hotkey(keys: string, holdDurationMs = 0): void {
  const [buf, len] = encodeStr(keys);
  assertOk(
    symbols.mac_hotkey(buf, len, BigInt(holdDurationMs)),
    "hotkey",
  );
}

export function typeText(text: string, delayMs = 50): void {
  const [buf, len] = encodeStr(text);
  assertOk(
    symbols.mac_type_text(buf, len, BigInt(delayMs)),
    "typeText",
  );
}

export function scroll(deltaX: number, deltaY: number): void {
  assertOk(symbols.mac_scroll(deltaX, deltaY), "scroll");
}

export function scrollDown(amount = 3): void {
  scroll(0, -amount);
}

export function scrollUp(amount = 3): void {
  scroll(0, amount);
}

export function drag(
  from: Point,
  to: Point,
  steps = 20,
  stepDelayMs = 10,
): void {
  assertOk(
    symbols.mac_drag(
      from.x, from.y, to.x, to.y, steps, BigInt(stepDelayMs),
    ),
    "drag",
  );
}

// ---- Screen Capture ----

/** Capture the main display. Returns PNG bytes. */
export function captureScreen(displayId = 0): Uint8Array {
  const data = callBytes((outLen) =>
    symbols.mac_capture_screen(displayId, outLen)
  );
  if (!data) throw new Error("Screen capture failed");
  return data;
}

/** Capture a window by its CGWindowID. Returns PNG bytes. */
export function captureWindow(windowId: number): Uint8Array {
  const data = callBytes((outLen) =>
    symbols.mac_capture_window(windowId, outLen)
  );
  if (!data) throw new Error("Window capture failed");
  return data;
}

// ---- Applications ----

export function listApplications(): AppInfo[] {
  return callJson<AppInfo[]>((outLen) =>
    symbols.mac_list_applications(outLen)
  ) ?? [];
}

export function frontmostApplication(): AppInfo | null {
  return callJson<AppInfo>((outLen) =>
    symbols.mac_frontmost_application(outLen)
  );
}

export function activateApp(bundleId: string): void {
  const [buf, len] = encodeStr(bundleId);
  assertOk(symbols.mac_activate_app(buf, len), "activateApp");
}

export function launchApp(bundleId: string): void {
  const [buf, len] = encodeStr(bundleId);
  assertOk(symbols.mac_launch_app(buf, len), "launchApp");
}

export function quitApp(bundleId: string, force = false): void {
  const [buf, len] = encodeStr(bundleId);
  assertOk(
    symbols.mac_quit_app(buf, len, force ? 1 : 0),
    "quitApp",
  );
}

// ---- Clipboard ----

export function readClipboard(): string | null {
  const data = callBytes((outLen) => symbols.mac_read_clipboard(outLen));
  if (!data) return null;
  return new TextDecoder().decode(data);
}

export function writeClipboard(text: string): void {
  const [buf, len] = encodeStr(text);
  assertOk(symbols.mac_write_clipboard(buf, len), "writeClipboard");
}

// ---- Accessibility ----

/** Get AX windows for a process. */
export function getWindows(pid: number): ElementInfo[] {
  return callJson<ElementInfo[]>((outLen) =>
    symbols.mac_get_windows(pid, outLen)
  ) ?? [];
}

/** Find all elements matching a role (e.g. "AXButton") in an app. */
export function findElementsByRole(
  pid: number,
  role: string,
  maxDepth = 10,
): ElementInfo[] {
  const [buf, len] = encodeStr(role);
  return callJson<ElementInfo[]>((outLen) =>
    symbols.mac_find_elements_by_role(pid, buf, len, maxDepth, outLen)
  ) ?? [];
}

/** Get the AX element at a screen position. */
export function elementAtPosition(
  pid: number,
  point: Point,
): ElementInfo | null {
  return callJson<ElementInfo>((outLen) =>
    symbols.mac_element_at_position(pid, point.x, point.y, outLen)
  );
}

/** Perform an AX action on the element at a screen position. */
export function performActionAt(
  pid: number,
  point: Point,
  action: string,
): void {
  const [buf, len] = encodeStr(action);
  assertOk(
    symbols.mac_perform_action_at(pid, point.x, point.y, buf, len),
    "performActionAt",
  );
}

// ---- Lifecycle ----

/** Close the dylib. Call when done. */
export function close(): void {
  lib.close();
}
```

---

## Part 5: Agent Usage Example

```typescript
// deno/agent.ts — example agent loop
//
// Run: deno run --allow-ffi --allow-write agent.ts

import {
  checkPermissions,
  captureScreen,
  click,
  typeText,
  hotkey,
  scrollDown,
  listApplications,
  frontmostApplication,
  activateApp,
  getWindows,
  findElementsByRole,
  close,
} from "./mac_bridge.ts";

async function agentLoop() {
  // 1. Check permissions on startup
  const perms = checkPermissions();
  console.log("Permissions:", perms);
  if (!perms.accessibility || !perms.screen_recording) {
    console.error("Missing required permissions. Grant them in System Settings.");
    Deno.exit(1);
  }

  // 2. See what's running
  const apps = listApplications();
  console.log(`${apps.length} running apps`);
  const front = frontmostApplication();
  console.log("Frontmost:", front?.name);

  // 3. Capture the screen
  const screenshot = captureScreen();
  await Deno.writeFile("/tmp/screen.png", screenshot);
  console.log(`Screenshot: ${screenshot.byteLength} bytes → /tmp/screen.png`);

  // 4. Find all buttons in the frontmost app
  if (front) {
    const buttons = findElementsByRole(front.pid, "AXButton", 8);
    console.log(`Found ${buttons.length} buttons:`);
    for (const btn of buttons.slice(0, 5)) {
      console.log(`  "${btn.title ?? btn.label}" at (${btn.x}, ${btn.y})`);
    }
  }

  // 5. Interact
  // (Your LLM decides what to do based on the screenshot)
  // click({ x: 500, y: 300 });
  // typeText("hello world");
  // hotkey("cmd+s");

  // 6. Cleanup
  close();
}

agentLoop();
```

### deno.json

```json
{
  "tasks": {
    "build": "cargo build --release",
    "agent": "deno run --allow-ffi --allow-write --allow-read deno/agent.ts",
    "dev": "cargo build && deno run --allow-ffi --allow-write --allow-read deno/agent.ts"
  },
  "compilerOptions": {
    "strict": true
  }
}
```

### Build and Run

```bash
# Build the Rust dylib
cargo build --release
# Produces: target/release/libmacbridge.dylib

# Run the agent
deno run --allow-ffi --allow-write --allow-read deno/agent.ts

# Or use the task shortcut
deno task agent
```

---

## Part 6: How Data Flows Across the FFI Boundary

### Simple calls (primitives only)

```
TS: click({ x: 500, y: 300 })
  → mac_bridge.ts calls symbols.mac_click(500.0, 300.0, 0, 1)
  → Deno passes f64, f64, u8, u32 directly to the C function
  → Rust: mac_click(500.0, 300.0, 0, 1) → input::click() → CGEventPost
  → Returns i32 (0 = success)
  → TS: assertOk checks return code
```

No allocation, no serialization. A single function call through the C ABI.

### String parameters

```
TS: hotkey("cmd+shift+s")
  → encodeStr("cmd+shift+s") → [Uint8Array(11), 11]
  → symbols.mac_hotkey(buffer, 11, 0n)
  → Deno passes Uint8Array as "buffer" type (pointer to JS typed array)
  → Rust: reads UTF-8 slice from pointer+length
  → Returns i32
```

The `"buffer"` FFI type passes a `Uint8Array` as a pointer directly to Rust.
No copy. The Rust side reads it as `*const u8` with a length parameter.

### Complex returns (JSON via pointer)

```
TS: listApplications()
  → callJson(outLen => symbols.mac_list_applications(outLen))
  → Allocates BigUint64Array(1) for length out-param
  → symbols.mac_list_applications(pointerToLenBuf)
  → Rust: serializes Vec<AppInfo> to JSON bytes
  → Leaks the Vec to get a stable pointer, writes length to out_len
  → Returns *mut u8 pointer
  → Deno: UnsafePointerView.copyInto(new Uint8Array(len))
  → Deno: symbols.mac_free_buffer(ptr, len) to release Rust memory
  → JSON.parse(decoder.decode(bytes)) → AppInfo[]
```

### Binary returns (PNG via pointer)

```
TS: captureScreen()
  → callBytes(outLen => symbols.mac_capture_screen(0, outLen))
  → Same pointer+length pattern as JSON
  → Returns raw PNG bytes as Uint8Array
  → No JSON parsing needed
```

---

## FFI Trade-offs and Design Choices

**What you get** from the Deno + Rust FFI approach:

- A single `cargo build --release` produces the `.dylib`. The TypeScript side
  loads it with one `Deno.dlopen` call. The build chain is two tools: `cargo`
  and `deno`.
- Primitive calls (click, scroll, hotkey) have zero serialization overhead —
  they're direct C function invocations from the JS runtime.
- The Rust core modules are plain library code with no framework-specific
  binding macros. They can be tested independently with `cargo test`.

**What you manage manually**:

- String and buffer passing — TypeScript must encode strings to `Uint8Array`
  and pass a length alongside. The `ffi_helpers.ts` module encapsulates this.
- Complex returns — Rust allocates a buffer (JSON or PNG bytes), returns a
  pointer + length, and TypeScript must call `mac_free_buffer` after reading.
  Again, `callJson` and `callBytes` in `ffi_helpers.ts` handle the pattern.
- Symbol definitions — each C function must be declared in both `ffi.rs`
  (Rust) and `mod.ts` (TypeScript). These must stay in sync manually.

---

## Incremental Build Order

1. **Start with input.rs + ffi.rs + permissions.rs** — mouse, keyboard, scroll,
   and permission checks. These are pure C-type calls with no pointer
   management on the return side. You can validate the entire FFI pipeline.

2. **Add capture.rs** — introduces the pointer+length return pattern for PNG
   bytes. Tests the `callBytes` / `mac_free_buffer` round-trip.

3. **Add apps.rs + clipboard.rs** — introduces JSON returns via pointer. Tests
   the `callJson` / `mac_free_buffer` round-trip.

4. **Add ax.rs** — the most complex module. Start with `mac_get_windows` and
   `mac_element_at_position`, then add tree-walking search.

5. **Defer ScreenCaptureKit** — CGWindowList legacy capture works fine. SCK
   requires ObjC async bridging (`block2` + `NSRunLoop`) which is substantially
   harder in Rust.

6. **Defer menu traversal** — deep AX tree walks with timeout handling. Only
   add if your agent needs to interact with menus.

---

## Appendix: Core Module Source

The implementations of `input.rs`, `ax.rs`, `capture.rs`, `apps.rs`,
`clipboard.rs`, and `permissions.rs` contain the actual macOS API calls
(CGEvent, AXUIElement, CGWindowList, NSWorkspace, NSPasteboard, etc.).
They are pure Rust modules with no FFI awareness — the `extern "C"`
boundary in `ffi.rs` is the only file that bridges them to Deno.

These modules will be detailed in a future revision of this document as
the Casper implementation progresses.
