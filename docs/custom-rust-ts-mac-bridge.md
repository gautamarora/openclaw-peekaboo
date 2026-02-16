# Building a TypeScript + Rust Mac Automation Bridge

Design document for building your own IPC bridge to macOS automation services,
replacing Peekaboo's Swift stack with TypeScript (agent orchestrator) and Rust
(native macOS API calls), connected via napi-rs.

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  TypeScript Agent (Node.js)                                  │
│                                                              │
│  agent.ts  ─── orchestrator / LLM loop                      │
│    ↓                                                         │
│  mac-bridge.ts  ─── typed wrapper around napi bindings       │
│    ↓                                                         │
│  @anthropic/mac-bridge (napi-rs)  ─── native Node addon      │
│    ↓                                                         │
│  Rust crate: mac-bridge-core                                 │
│    ├── input.rs      ─── CGEvent (mouse, keyboard, scroll)  │
│    ├── ax.rs         ─── AXUIElement (accessibility tree)    │
│    ├── capture.rs    ─── CGWindowList / ScreenCaptureKit     │
│    ├── apps.rs       ─── NSWorkspace / NSRunningApplication  │
│    ├── windows.rs    ─── AX window attrs + CGWindowList      │
│    ├── clipboard.rs  ─── NSPasteboard                        │
│    ├── permissions.rs─── TCC permission checks               │
│    └── screen.rs     ─── NSScreen display enumeration        │
└──────────────────────────────────────────────────────────────┘
         ↓ calls directly via FFI ↓
┌──────────────────────────────────────────────────────────────┐
│  macOS Frameworks (linked at build time)                     │
│                                                              │
│  ApplicationServices.framework                               │
│    ├── AXUIElement*  (Accessibility)                         │
│    └── CGEvent*      (Quartz Event Services)                 │
│  CoreGraphics.framework                                      │
│    ├── CGWindowList* (window enumeration + legacy capture)   │
│    ├── CGDisplay*    (display info + legacy capture)         │
│    └── CGImage*      (image handling)                        │
│  ScreenCaptureKit.framework  (macOS 13+)                     │
│    ├── SCShareableContent                                    │
│    ├── SCScreenshotManager                                   │
│    └── SCContentFilter / SCStreamConfiguration               │
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

### Why This Architecture

- **No external daemon needed** — your process directly holds TCC permissions
  and calls macOS APIs. No socket protocol, no auth handshake, no Peekaboo
  dependency.
- **napi-rs** gives you synchronous and async Rust functions callable from
  TypeScript with zero-copy where possible. The N-API boundary is a function
  call, not serialization.
- **Single process** — the Node.js runtime, your TS agent, and the Rust native
  code all live in one process. TCC grants (Accessibility, Screen Recording)
  apply to this one process.

### TCC Permissions

Your Node.js process needs these macOS permissions (granted to the terminal
or to your Electron/.app wrapper):

| Permission | Needed for | System Preferences path |
|---|---|---|
| **Accessibility** | AXUIElement, CGEvent input | Privacy & Security → Accessibility |
| **Screen Recording** | SCScreenshotManager, CGWindowListCreateImage | Privacy & Security → Screen Recording |
| **Automation** (optional) | NSAppleScript, app launch/quit via AppleScript | Privacy & Security → Automation |

If you run from Terminal.app or iTerm, the terminal itself needs the grants.
If you package as an .app (Electron, Tauri), the .app bundle gets the grants.

---

## Part 1: Rust Crate Structure

### Cargo workspace

```
mac-bridge/
├── Cargo.toml              # workspace root
├── crates/
│   ├── mac-bridge-core/    # pure Rust, no napi dependency
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── input.rs        # mouse, keyboard, scroll
│   │       ├── ax.rs           # accessibility tree
│   │       ├── capture.rs      # screen capture
│   │       ├── apps.rs         # application management
│   │       ├── windows.rs      # window management
│   │       ├── clipboard.rs    # pasteboard
│   │       ├── permissions.rs  # TCC checks
│   │       ├── screen.rs       # display enumeration
│   │       └── types.rs        # shared types
│   └── mac-bridge-napi/    # napi-rs bindings (thin layer)
│       ├── Cargo.toml
│       └── src/
│           └── lib.rs      # #[napi] exports wrapping core
├── package.json            # npm package config
├── index.d.ts              # generated TypeScript types
└── ts/
    └── mac-bridge.ts       # high-level TS wrapper
```

### Dependencies (Cargo.toml for mac-bridge-core)

```toml
[package]
name = "mac-bridge-core"
version = "0.1.0"
edition = "2021"

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
block2 = "0.6"
thiserror = "2"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
base64 = "0.22"
png = "0.17"

[build-dependencies]
cc = "1"   # for linking .m bridging files if needed
```

### Dependencies (Cargo.toml for mac-bridge-napi)

```toml
[package]
name = "mac-bridge-napi"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
mac-bridge-core = { path = "../mac-bridge-core" }
napi = { version = "3", features = ["async", "napi9", "serde"] }
napi-derive = "3"
serde_json = "1"

[build-dependencies]
napi-build = "2"
```

---

## Part 2: Rust Implementation — macOS API Calls

### 2.1 Input Simulation (input.rs)

This replaces Peekaboo's `ClickService`, `HotkeyService`, `ScrollService`,
`GestureService`, all of which delegate to AXorcist's `InputDriver`.

The underlying C APIs are in `ApplicationServices.framework`:

```rust
// input.rs
use core_graphics::event::{
    CGEvent, CGEventTapLocation, CGEventType, CGMouseButton,
    CGEventFlags, ScrollEventUnit,
};
use core_graphics::event_source::{CGEventSource, CGEventSourceStateID};
use core_graphics::geometry::CGPoint;

/// Create an event source for synthetic events.
fn event_source() -> CGEventSource {
    CGEventSource::new(CGEventSourceStateID::HIDSystemState)
        .expect("failed to create event source")
}

/// Move the mouse cursor to a point.
pub fn mouse_move(x: f64, y: f64) -> Result<(), InputError> {
    let source = event_source();
    let point = CGPoint::new(x, y);
    let event = CGEvent::new_mouse_event(
        source,
        CGEventType::MouseMoved,
        point,
        CGMouseButton::Left, // ignored for move events
    ).map_err(|_| InputError::EventCreation)?;
    event.post(CGEventTapLocation::HID);
    Ok(())
}

/// Click at a point.
pub fn click(x: f64, y: f64, button: MouseButton, count: u32) -> Result<(), InputError> {
    let source = event_source();
    let point = CGPoint::new(x, y);

    let (down_type, up_type, cg_button) = match button {
        MouseButton::Left => (
            CGEventType::LeftMouseDown,
            CGEventType::LeftMouseUp,
            CGMouseButton::Left,
        ),
        MouseButton::Right => (
            CGEventType::RightMouseDown,
            CGEventType::RightMouseUp,
            CGMouseButton::Right,
        ),
    };

    for i in 0..count {
        let down = CGEvent::new_mouse_event(source.clone(), down_type, point, cg_button)
            .map_err(|_| InputError::EventCreation)?;
        let up = CGEvent::new_mouse_event(source.clone(), up_type, point, cg_button)
            .map_err(|_| InputError::EventCreation)?;

        // Set click count for double/triple clicks
        let click_number = (i + 1) as i64;
        down.set_integer_value_field(
            core_graphics::event::EventField::MOUSE_EVENT_CLICK_STATE,
            click_number,
        );
        up.set_integer_value_field(
            core_graphics::event::EventField::MOUSE_EVENT_CLICK_STATE,
            click_number,
        );

        down.post(CGEventTapLocation::HID);
        up.post(CGEventTapLocation::HID);
    }
    Ok(())
}

/// Press a keyboard shortcut. Keys string like "cmd+shift+s".
pub fn hotkey(keys: &str, hold_duration_ms: u64) -> Result<(), InputError> {
    let source = event_source();
    let parts: Vec<&str> = keys.split('+').map(str::trim).collect();

    let mut flags = CGEventFlags::empty();
    let mut key_code: Option<u16> = None;

    for part in &parts {
        match part.to_lowercase().as_str() {
            "cmd" | "command" => flags |= CGEventFlags::CGEventFlagCommand,
            "shift"           => flags |= CGEventFlags::CGEventFlagShift,
            "ctrl" | "control"=> flags |= CGEventFlags::CGEventFlagControl,
            "alt" | "option"  => flags |= CGEventFlags::CGEventFlagAlternate,
            other             => key_code = Some(keycode_for_name(other)?),
        }
    }

    let code = key_code.ok_or(InputError::NoKeySpecified)?;

    // Key down
    let down = CGEvent::new_keyboard_event(source.clone(), code, true)
        .map_err(|_| InputError::EventCreation)?;
    down.set_flags(flags);
    down.post(CGEventTapLocation::HID);

    if hold_duration_ms > 0 {
        std::thread::sleep(std::time::Duration::from_millis(hold_duration_ms));
    }

    // Key up
    let up = CGEvent::new_keyboard_event(source, code, false)
        .map_err(|_| InputError::EventCreation)?;
    up.set_flags(flags);
    up.post(CGEventTapLocation::HID);

    Ok(())
}

/// Type a string character by character using CGEvent keyboard events.
pub fn type_text(text: &str, delay_ms: u64) -> Result<(), InputError> {
    let source = event_source();
    for ch in text.chars() {
        // Use CGEvent's key_from_char or Unicode input
        let event = CGEvent::new_keyboard_event(source.clone(), 0, true)
            .map_err(|_| InputError::EventCreation)?;
        // Set the Unicode string directly on the event
        let chars = [ch as u16];
        unsafe {
            // CGEventKeyboardSetUnicodeString
            core_graphics::sys::CGEventKeyboardSetUnicodeString(
                event.as_concrete_TypeRef(),
                chars.len() as libc::c_ulong,
                chars.as_ptr(),
            );
        }
        event.post(CGEventTapLocation::HID);

        let up = CGEvent::new_keyboard_event(source.clone(), 0, false)
            .map_err(|_| InputError::EventCreation)?;
        up.post(CGEventTapLocation::HID);

        if delay_ms > 0 {
            std::thread::sleep(std::time::Duration::from_millis(delay_ms));
        }
    }
    Ok(())
}

/// Scroll at a point.
pub fn scroll(delta_x: i32, delta_y: i32) -> Result<(), InputError> {
    let source = event_source();
    let event = CGEvent::new_scroll_event(
        source,
        ScrollEventUnit::LINE,
        2,       // axis count
        delta_y, // vertical first
        delta_x, // horizontal second
        0,
    ).map_err(|_| InputError::EventCreation)?;
    event.post(CGEventTapLocation::HID);
    Ok(())
}

/// Drag from one point to another with intermediate steps.
pub fn drag(
    from_x: f64, from_y: f64,
    to_x: f64, to_y: f64,
    steps: u32,
    step_delay_ms: u64,
) -> Result<(), InputError> {
    let source = event_source();
    let from = CGPoint::new(from_x, from_y);

    // Mouse down at start
    let down = CGEvent::new_mouse_event(
        source.clone(), CGEventType::LeftMouseDown, from, CGMouseButton::Left,
    ).map_err(|_| InputError::EventCreation)?;
    down.post(CGEventTapLocation::HID);

    // Interpolated drag steps
    for i in 1..=steps {
        let t = i as f64 / steps as f64;
        let x = from_x + (to_x - from_x) * t;
        let y = from_y + (to_y - from_y) * t;
        let point = CGPoint::new(x, y);

        let drag_event = CGEvent::new_mouse_event(
            source.clone(), CGEventType::LeftMouseDragged, point, CGMouseButton::Left,
        ).map_err(|_| InputError::EventCreation)?;
        drag_event.post(CGEventTapLocation::HID);

        std::thread::sleep(std::time::Duration::from_millis(step_delay_ms));
    }

    // Mouse up at end
    let up = CGEvent::new_mouse_event(
        source, CGEventType::LeftMouseUp, CGPoint::new(to_x, to_y), CGMouseButton::Left,
    ).map_err(|_| InputError::EventCreation)?;
    up.post(CGEventTapLocation::HID);

    Ok(())
}

/// Map key names to macOS virtual key codes.
fn keycode_for_name(name: &str) -> Result<u16, InputError> {
    match name {
        "a" => Ok(0x00), "s" => Ok(0x01), "d" => Ok(0x02), "f" => Ok(0x03),
        "h" => Ok(0x04), "g" => Ok(0x05), "z" => Ok(0x06), "x" => Ok(0x07),
        "c" => Ok(0x08), "v" => Ok(0x09), "b" => Ok(0x0B), "q" => Ok(0x0C),
        "w" => Ok(0x0D), "e" => Ok(0x0E), "r" => Ok(0x0F), "y" => Ok(0x10),
        "t" => Ok(0x11), "1" => Ok(0x12), "2" => Ok(0x13), "3" => Ok(0x14),
        "4" => Ok(0x15), "6" => Ok(0x16), "5" => Ok(0x17), "9" => Ok(0x19),
        "7" => Ok(0x1A), "8" => Ok(0x1C), "0" => Ok(0x1D),
        "return" | "enter" => Ok(0x24),
        "tab"              => Ok(0x30),
        "space"            => Ok(0x31),
        "delete"           => Ok(0x33),
        "escape" | "esc"   => Ok(0x35),
        "left"             => Ok(0x7B),
        "right"            => Ok(0x7C),
        "down"             => Ok(0x7D),
        "up"               => Ok(0x7E),
        "f1"               => Ok(0x7A),
        "f2"               => Ok(0x78),
        "f3"               => Ok(0x63),
        "f4"               => Ok(0x76),
        "f5"               => Ok(0x60),
        // ... extend as needed
        other => Err(InputError::UnknownKey(other.to_string())),
    }
}

#[derive(Debug)]
pub enum MouseButton { Left, Right }

#[derive(Debug, thiserror::Error)]
pub enum InputError {
    #[error("failed to create CGEvent")]
    EventCreation,
    #[error("no key specified in hotkey string")]
    NoKeySpecified,
    #[error("unknown key name: {0}")]
    UnknownKey(String),
}
```

### 2.2 Accessibility Tree (ax.rs)

This replaces Peekaboo's AXorcist dependency. The raw C API is in
`ApplicationServices.framework` (`HIServices` sub-framework).

```rust
// ax.rs
//
// AXUIElement is a C API. Rust calls it via FFI.
// The core-foundation crate provides CFType wrappers;
// we declare the AX functions ourselves.

use core_foundation::base::{CFType, TCFType, CFTypeRef, Boolean};
use core_foundation::string::{CFString, CFStringRef};
use core_foundation::array::{CFArray, CFArrayRef};
use std::ffi::c_void;
use std::ptr;

// --- FFI declarations ---

#[repr(C)]
pub struct __AXUIElement(c_void);
pub type AXUIElementRef = *mut __AXUIElement;

pub type AXError = i32;
pub const kAXErrorSuccess: AXError = 0;
pub const kAXErrorNoValue: AXError = -25212;

extern "C" {
    fn AXUIElementCreateSystemWide() -> AXUIElementRef;
    fn AXUIElementCreateApplication(pid: i32) -> AXUIElementRef;
    fn AXUIElementCopyAttributeValue(
        element: AXUIElementRef,
        attribute: CFStringRef,
        value: *mut CFTypeRef,
    ) -> AXError;
    fn AXUIElementCopyAttributeNames(
        element: AXUIElementRef,
        names: *mut CFArrayRef,
    ) -> AXError;
    fn AXUIElementPerformAction(
        element: AXUIElementRef,
        action: CFStringRef,
    ) -> AXError;
    fn AXUIElementSetAttributeValue(
        element: AXUIElementRef,
        attribute: CFStringRef,
        value: CFTypeRef,
    ) -> AXError;
    fn AXUIElementCopyElementAtPosition(
        application: AXUIElementRef,
        x: f32,
        y: f32,
        element: *mut AXUIElementRef,
    ) -> AXError;
    fn AXUIElementGetPid(
        element: AXUIElementRef,
        pid: *mut i32,
    ) -> AXError;
    fn AXUIElementSetMessagingTimeout(
        element: AXUIElementRef,
        timeout_in_seconds: f32,
    ) -> AXError;
    fn AXIsProcessTrusted() -> Boolean;
    fn AXIsProcessTrustedWithOptions(options: core_foundation::dictionary::CFDictionaryRef) -> Boolean;
}

// --- Safe wrappers ---

/// Check if this process has Accessibility permission.
pub fn is_trusted() -> bool {
    unsafe { AXIsProcessTrusted() != 0 }
}

/// Get the system-wide AXUIElement.
pub fn system_wide() -> AXElement {
    AXElement { raw: unsafe { AXUIElementCreateSystemWide() } }
}

/// Get the AXUIElement for an application by PID.
pub fn application(pid: i32) -> AXElement {
    AXElement { raw: unsafe { AXUIElementCreateApplication(pid) } }
}

/// Wrapper around AXUIElementRef with safe attribute access.
pub struct AXElement {
    raw: AXUIElementRef,
}

// AXUIElementRef is CFType-based and follows CFRetain/CFRelease.
impl Drop for AXElement {
    fn drop(&mut self) {
        if !self.raw.is_null() {
            unsafe { core_foundation::base::CFRelease(self.raw as CFTypeRef) };
        }
    }
}

impl AXElement {
    /// Get a string attribute (e.g. "AXTitle", "AXRole", "AXValue").
    pub fn string_attr(&self, name: &str) -> Option<String> {
        let cf_name = CFString::new(name);
        let mut value: CFTypeRef = ptr::null_mut();
        let err = unsafe {
            AXUIElementCopyAttributeValue(self.raw, cf_name.as_concrete_TypeRef(), &mut value)
        };
        if err != kAXErrorSuccess || value.is_null() {
            return None;
        }
        // Try to cast to CFString
        let cf_str = unsafe { CFString::wrap_under_create_rule(value as CFStringRef) };
        Some(cf_str.to_string())
    }

    /// Get an array attribute (e.g. "AXChildren", "AXWindows").
    pub fn children(&self) -> Vec<AXElement> {
        self.array_attr("AXChildren")
    }

    pub fn windows(&self) -> Vec<AXElement> {
        self.array_attr("AXWindows")
    }

    fn array_attr(&self, name: &str) -> Vec<AXElement> {
        let cf_name = CFString::new(name);
        let mut value: CFTypeRef = ptr::null_mut();
        let err = unsafe {
            AXUIElementCopyAttributeValue(self.raw, cf_name.as_concrete_TypeRef(), &mut value)
        };
        if err != kAXErrorSuccess || value.is_null() {
            return vec![];
        }
        let array = unsafe { CFArray::wrap_under_create_rule(value as CFArrayRef) };
        let mut result = Vec::new();
        for i in 0..array.len() {
            let elem_ref = array.get(i).expect("array index");
            // Retain the element since we're storing it
            unsafe { core_foundation::base::CFRetain(elem_ref as CFTypeRef) };
            result.push(AXElement { raw: elem_ref as AXUIElementRef });
        }
        result
    }

    /// Get the element's role (e.g. "AXButton", "AXTextField").
    pub fn role(&self) -> Option<String> {
        self.string_attr("AXRole")
    }

    /// Get the element's title.
    pub fn title(&self) -> Option<String> {
        self.string_attr("AXTitle")
    }

    /// Get the element's label (AXDescription in AX terms).
    pub fn label(&self) -> Option<String> {
        self.string_attr("AXDescription")
    }

    /// Get the element's value as string.
    pub fn value(&self) -> Option<String> {
        self.string_attr("AXValue")
    }

    /// Get element position as (x, y).
    pub fn position(&self) -> Option<(f64, f64)> {
        self.point_attr("AXPosition")
    }

    /// Get element size as (w, h).
    pub fn size(&self) -> Option<(f64, f64)> {
        self.point_attr("AXSize")
    }

    /// Get element bounds as (x, y, w, h).
    pub fn frame(&self) -> Option<(f64, f64, f64, f64)> {
        let (x, y) = self.position()?;
        let (w, h) = self.size()?;
        Some((x, y, w, h))
    }

    fn point_attr(&self, name: &str) -> Option<(f64, f64)> {
        let cf_name = CFString::new(name);
        let mut value: CFTypeRef = ptr::null_mut();
        let err = unsafe {
            AXUIElementCopyAttributeValue(self.raw, cf_name.as_concrete_TypeRef(), &mut value)
        };
        if err != kAXErrorSuccess || value.is_null() {
            return None;
        }
        // AXValue wraps CGPoint/CGSize. Extract via AXValueGetValue.
        // We need to use the AXValue C API here.
        let mut point = core_graphics::geometry::CGPoint::new(0.0, 0.0);
        let ok = unsafe {
            AXValueGetValue(
                value as AXValueRef,
                kAXValueCGPointType,
                &mut point as *mut _ as *mut c_void,
            )
        };
        unsafe { core_foundation::base::CFRelease(value) };
        if ok != 0 { Some((point.x, point.y)) } else { None }
    }

    /// Perform an action (e.g. "AXPress", "AXConfirm", "AXRaise").
    pub fn perform_action(&self, action: &str) -> Result<(), AXError> {
        let cf_action = CFString::new(action);
        let err = unsafe { AXUIElementPerformAction(self.raw, cf_action.as_concrete_TypeRef()) };
        if err == kAXErrorSuccess { Ok(()) } else { Err(err) }
    }

    /// Set an attribute value (e.g. set AXPosition to move a window).
    pub fn set_position(&self, x: f64, y: f64) -> Result<(), AXError> {
        let point = core_graphics::geometry::CGPoint::new(x, y);
        let ax_value = unsafe {
            AXValueCreate(kAXValueCGPointType, &point as *const _ as *const c_void)
        };
        let cf_name = CFString::new("AXPosition");
        let err = unsafe {
            AXUIElementSetAttributeValue(self.raw, cf_name.as_concrete_TypeRef(), ax_value as CFTypeRef)
        };
        unsafe { core_foundation::base::CFRelease(ax_value as CFTypeRef) };
        if err == kAXErrorSuccess { Ok(()) } else { Err(err) }
    }

    pub fn set_size(&self, w: f64, h: f64) -> Result<(), AXError> {
        let size = core_graphics::geometry::CGSize::new(w, h);
        let ax_value = unsafe {
            AXValueCreate(kAXValueCGSizeType, &size as *const _ as *const c_void)
        };
        let cf_name = CFString::new("AXSize");
        let err = unsafe {
            AXUIElementSetAttributeValue(self.raw, cf_name.as_concrete_TypeRef(), ax_value as CFTypeRef)
        };
        unsafe { core_foundation::base::CFRelease(ax_value as CFTypeRef) };
        if err == kAXErrorSuccess { Ok(()) } else { Err(err) }
    }

    /// Set the messaging timeout for this element.
    pub fn set_timeout(&self, seconds: f32) {
        unsafe { AXUIElementSetMessagingTimeout(self.raw, seconds) };
    }

    /// Get the PID of the process that owns this element.
    pub fn pid(&self) -> Option<i32> {
        let mut pid: i32 = 0;
        let err = unsafe { AXUIElementGetPid(self.raw, &mut pid) };
        if err == kAXErrorSuccess { Some(pid) } else { None }
    }

    /// Get the element at a screen coordinate (for the given app).
    pub fn element_at_position(&self, x: f32, y: f32) -> Option<AXElement> {
        let mut element: AXUIElementRef = ptr::null_mut();
        let err = unsafe {
            AXUIElementCopyElementAtPosition(self.raw, x, y, &mut element)
        };
        if err == kAXErrorSuccess && !element.is_null() {
            Some(AXElement { raw: element })
        } else {
            None
        }
    }

    /// Walk the AX tree and collect elements matching a predicate.
    pub fn find_all<F>(&self, max_depth: usize, predicate: &F) -> Vec<AXElement>
    where
        F: Fn(&AXElement) -> bool,
    {
        let mut results = Vec::new();
        self.walk(max_depth, 0, predicate, &mut results);
        results
    }

    fn walk<F>(&self, max_depth: usize, depth: usize, predicate: &F, results: &mut Vec<AXElement>)
    where
        F: Fn(&AXElement) -> bool,
    {
        if depth > max_depth { return; }
        if predicate(self) {
            // Retain since we're storing a reference
            unsafe { core_foundation::base::CFRetain(self.raw as CFTypeRef) };
            results.push(AXElement { raw: self.raw });
        }
        for child in self.children() {
            child.walk(max_depth, depth + 1, predicate, results);
        }
    }
}

// --- Additional FFI for AXValue ---

type AXValueRef = *const c_void;
const kAXValueCGPointType: u32 = 1;
const kAXValueCGSizeType: u32 = 2;

extern "C" {
    fn AXValueCreate(value_type: u32, value: *const c_void) -> AXValueRef;
    fn AXValueGetValue(value: AXValueRef, value_type: u32, out: *mut c_void) -> Boolean;
}
```

### 2.3 Screen Capture (capture.rs)

Two paths: modern ScreenCaptureKit (macOS 13+) and legacy CGWindowList.

```rust
// capture.rs
use core_graphics::display::{
    CGDisplay, CGWindowListCopyWindowInfo,
    kCGWindowListOptionAll, kCGWindowListExcludeDesktopElements,
    kCGNullWindowID,
};
use core_graphics::image::CGImage;
use core_graphics::geometry::CGRect;

/// Capture the entire main display using the legacy CGDisplay API.
/// Works on all macOS versions, no ScreenCaptureKit needed.
pub fn capture_display(display_id: u32) -> Result<Vec<u8>, CaptureError> {
    let image = CGDisplay::image(display_id)
        .ok_or(CaptureError::DisplayCaptureFailed)?;
    image_to_png(&image)
}

/// Capture a specific window by its CGWindowID using legacy API.
pub fn capture_window(window_id: u32) -> Result<Vec<u8>, CaptureError> {
    let cg_rect = CGRect::null(); // .infinite captures full window
    let image = unsafe {
        // CGWindowListCreateImage(rect, listOption, windowID, imageOption)
        CGImage::from_window_list(
            cg_rect,
            core_graphics::display::kCGWindowListOptionIncludingWindow,
            window_id,
            core_graphics::display::kCGWindowImageBoundsIgnoreFraming
                | core_graphics::display::kCGWindowImageBestResolution,
        )
    }.ok_or(CaptureError::WindowCaptureFailed)?;
    image_to_png(&image)
}

/// List all on-screen windows via CGWindowListCopyWindowInfo.
/// Returns (window_id, pid, title, bounds) tuples.
pub fn list_windows() -> Vec<WindowInfo> {
    let info = unsafe {
        CGWindowListCopyWindowInfo(
            kCGWindowListOptionAll | kCGWindowListExcludeDesktopElements,
            kCGNullWindowID,
        )
    };
    // info is a CFArray of CFDictionary. Parse each entry.
    // Extract: kCGWindowNumber, kCGWindowOwnerPID, kCGWindowName, kCGWindowBounds
    parse_window_list(info)
}

/// Encode a CGImage as PNG bytes.
fn image_to_png(image: &CGImage) -> Result<Vec<u8>, CaptureError> {
    let width = image.width();
    let height = image.height();
    let data = image.data(); // CFData with raw pixels
    let bytes = data.bytes();

    let mut png_bytes = Vec::new();
    let mut encoder = png::Encoder::new(&mut png_bytes, width as u32, height as u32);
    encoder.set_color(png::ColorType::Rgba);
    encoder.set_depth(png::BitDepth::Eight);
    let mut writer = encoder.write_header()
        .map_err(|_| CaptureError::PngEncodeFailed)?;

    // CGImage pixel data may have extra bytes_per_row padding
    let bpr = image.bytes_per_row();
    let expected_bpr = width * 4;
    if bpr == expected_bpr {
        writer.write_image_data(bytes).map_err(|_| CaptureError::PngEncodeFailed)?;
    } else {
        // Strip row padding
        let mut stripped = Vec::with_capacity(height * expected_bpr);
        for row in 0..height {
            let start = row * bpr;
            stripped.extend_from_slice(&bytes[start..start + expected_bpr]);
        }
        writer.write_image_data(&stripped).map_err(|_| CaptureError::PngEncodeFailed)?;
    }

    drop(writer);
    Ok(png_bytes)
}

// For ScreenCaptureKit (macOS 13+), you'd use objc2 to call the
// Objective-C API. This requires an async bridge since SCK is
// callback-based. See section 2.3.1 below.
```

#### 2.3.1 ScreenCaptureKit via objc2 (optional, macOS 13+)

ScreenCaptureKit is an Objective-C framework. Calling it from Rust requires
`objc2` message sends and an active `NSRunLoop`. This is the hardest part
of the Rust bridge. For a first version, the CGWindowList/CGDisplay legacy
APIs above are sufficient and much simpler.

If you need ScreenCaptureKit (better quality, no prompt badge):

```rust
// capture_sck.rs — ScreenCaptureKit via objc2
//
// This is a sketch. SCK requires:
// 1. An async completion handler (block2 crate)
// 2. A running NSRunLoop on the calling thread
// 3. objc2-screen-capture-kit bindings (or hand-rolled)

use objc2::rc::Retained;
use objc2::runtime::AnyObject;
use objc2::{msg_send, msg_send_id, class};
use block2::ConcreteBlock;

/// Capture using SCScreenshotManager (macOS 14+).
pub async fn capture_screen_sck() -> Result<Vec<u8>, CaptureError> {
    // 1. Get shareable content
    //    [SCShareableContent getShareableContentExcludingDesktopWindows:NO
    //                                        onScreenWindowsOnly:YES
    //                                        completionHandler:^(content, error) { ... }]
    //
    // 2. Create filter for the display
    //    [[SCContentFilter alloc] initWithDisplay:display excludingWindows:@[]]
    //
    // 3. Create configuration
    //    SCStreamConfiguration *config = [[SCStreamConfiguration alloc] init];
    //    config.width = display.width;
    //    config.height = display.height;
    //    config.showsCursor = NO;
    //
    // 4. Capture
    //    [SCScreenshotManager captureImageWithFilter:filter
    //                                configuration:config
    //                            completionHandler:^(image, error) { ... }]
    //
    // The completion handlers need to be bridged to Rust futures
    // using tokio::sync::oneshot channels inside block2 blocks.

    todo!("SCK implementation requires objc2 + block2 + runloop")
}
```

### 2.4 Application Management (apps.rs)

```rust
// apps.rs — NSWorkspace and NSRunningApplication via objc2
use objc2_app_kit::{NSWorkspace, NSRunningApplication, NSApplicationActivationPolicy};
use objc2_foundation::{NSString, NSArray};

/// List all running GUI applications.
pub fn list_applications() -> Vec<AppInfo> {
    let workspace = unsafe { NSWorkspace::sharedWorkspace() };
    let apps = unsafe { workspace.runningApplications() };
    let mut result = Vec::new();

    for i in 0..apps.len() {
        let app: &NSRunningApplication = &apps[i];
        let policy = unsafe { app.activationPolicy() };
        // Skip background-only and prohibited apps
        if policy != NSApplicationActivationPolicy::Regular {
            continue;
        }
        result.push(AppInfo {
            pid: unsafe { app.processIdentifier() },
            bundle_id: unsafe { app.bundleIdentifier() }
                .map(|s| s.to_string()),
            name: unsafe { app.localizedName() }
                .map(|s| s.to_string()),
            is_active: unsafe { app.isActive() },
            is_hidden: unsafe { app.isHidden() },
        });
    }
    result
}

/// Get the frontmost application.
pub fn frontmost_application() -> Option<AppInfo> {
    let workspace = unsafe { NSWorkspace::sharedWorkspace() };
    let app = unsafe { workspace.frontmostApplication()? };
    Some(AppInfo {
        pid: unsafe { app.processIdentifier() },
        bundle_id: unsafe { app.bundleIdentifier() }.map(|s| s.to_string()),
        name: unsafe { app.localizedName() }.map(|s| s.to_string()),
        is_active: true,
        is_hidden: false,
    })
}

/// Activate (bring to front) an application by bundle ID.
pub fn activate_app(bundle_id: &str) -> Result<(), AppError> {
    let app = find_running_app(bundle_id)?;
    unsafe { app.activateWithOptions(0x01) }; // NSApplicationActivateIgnoringOtherApps
    Ok(())
}

/// Hide an application.
pub fn hide_app(bundle_id: &str) -> Result<(), AppError> {
    let app = find_running_app(bundle_id)?;
    unsafe { app.hide() };
    Ok(())
}

/// Quit an application (gracefully or force).
pub fn quit_app(bundle_id: &str, force: bool) -> Result<bool, AppError> {
    let app = find_running_app(bundle_id)?;
    let result = if force {
        unsafe { app.forceTerminate() }
    } else {
        unsafe { app.terminate() }
    };
    Ok(result)
}

/// Launch an application by bundle ID.
pub fn launch_app(bundle_id: &str) -> Result<(), AppError> {
    let workspace = unsafe { NSWorkspace::sharedWorkspace() };
    let ns_bundle_id = NSString::from_str(bundle_id);
    let url = unsafe { workspace.URLForApplicationWithBundleIdentifier(&ns_bundle_id) }
        .ok_or(AppError::NotFound(bundle_id.to_string()))?;

    let config = unsafe { objc2_app_kit::NSWorkspaceOpenConfiguration::new() };
    // Launch is async with completion handler; for simplicity, fire-and-forget
    unsafe {
        workspace.openApplicationAtURL_configuration_completionHandler(
            &url, &config, None,
        );
    }
    Ok(())
}

fn find_running_app(bundle_id: &str) -> Result<Retained<NSRunningApplication>, AppError> {
    let workspace = unsafe { NSWorkspace::sharedWorkspace() };
    let apps = unsafe { workspace.runningApplications() };
    for i in 0..apps.len() {
        let app: &NSRunningApplication = &apps[i];
        if let Some(bid) = unsafe { app.bundleIdentifier() } {
            if bid.to_string() == bundle_id {
                return Ok(app.retain());
            }
        }
    }
    Err(AppError::NotFound(bundle_id.to_string()))
}

#[derive(Debug, serde::Serialize)]
pub struct AppInfo {
    pub pid: i32,
    pub bundle_id: Option<String>,
    pub name: Option<String>,
    pub is_active: bool,
    pub is_hidden: bool,
}

#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("application not found: {0}")]
    NotFound(String),
}
```

### 2.5 Clipboard (clipboard.rs)

```rust
// clipboard.rs
use objc2_app_kit::NSPasteboard;
use objc2_foundation::NSString;

pub fn read_clipboard() -> Option<String> {
    let pb = unsafe { NSPasteboard::generalPasteboard() };
    let ns_type = unsafe { NSString::from_str("public.utf8-plain-text") };
    unsafe { pb.stringForType(&ns_type) }.map(|s| s.to_string())
}

pub fn write_clipboard(text: &str) {
    let pb = unsafe { NSPasteboard::generalPasteboard() };
    unsafe { pb.clearContents() };
    let ns_string = NSString::from_str(text);
    let ns_type = unsafe { NSString::from_str("public.utf8-plain-text") };
    unsafe { pb.setString_forType(&ns_string, &ns_type) };
}
```

### 2.6 Permissions (permissions.rs)

```rust
// permissions.rs
use core_graphics::display::CGPreflightScreenCaptureAccess;

pub struct PermissionsStatus {
    pub accessibility: bool,
    pub screen_recording: bool,
}

pub fn check_permissions() -> PermissionsStatus {
    PermissionsStatus {
        accessibility: super::ax::is_trusted(),
        screen_recording: unsafe { CGPreflightScreenCaptureAccess() },
    }
}

pub fn request_screen_recording() -> bool {
    unsafe { core_graphics::display::CGRequestScreenCaptureAccess() }
}
```

---

## Part 3: napi-rs Binding Layer

The napi layer is thin — it just exposes `mac-bridge-core` functions to Node.js.

```rust
// mac-bridge-napi/src/lib.rs
use napi_derive::napi;
use mac_bridge_core::{input, ax, capture, apps, clipboard, permissions};

// --- Input ---

#[napi]
pub fn mouse_move(x: f64, y: f64) -> napi::Result<()> {
    input::mouse_move(x, y).map_err(|e| napi::Error::from_reason(e.to_string()))
}

#[napi]
pub fn click(x: f64, y: f64, button: String, count: u32) -> napi::Result<()> {
    let btn = match button.as_str() {
        "right" => input::MouseButton::Right,
        _ => input::MouseButton::Left,
    };
    input::click(x, y, btn, count).map_err(|e| napi::Error::from_reason(e.to_string()))
}

#[napi]
pub fn hotkey(keys: String, hold_duration_ms: u32) -> napi::Result<()> {
    input::hotkey(&keys, hold_duration_ms as u64)
        .map_err(|e| napi::Error::from_reason(e.to_string()))
}

#[napi]
pub fn type_text(text: String, delay_ms: u32) -> napi::Result<()> {
    input::type_text(&text, delay_ms as u64)
        .map_err(|e| napi::Error::from_reason(e.to_string()))
}

#[napi]
pub fn scroll(delta_x: i32, delta_y: i32) -> napi::Result<()> {
    input::scroll(delta_x, delta_y)
        .map_err(|e| napi::Error::from_reason(e.to_string()))
}

#[napi]
pub fn drag(
    from_x: f64, from_y: f64,
    to_x: f64, to_y: f64,
    steps: u32, step_delay_ms: u32,
) -> napi::Result<()> {
    input::drag(from_x, from_y, to_x, to_y, steps, step_delay_ms as u64)
        .map_err(|e| napi::Error::from_reason(e.to_string()))
}

// --- Accessibility ---

#[napi]
pub fn is_accessibility_trusted() -> bool {
    ax::is_trusted()
}

#[napi(object)]
pub struct ElementInfo {
    pub role: Option<String>,
    pub title: Option<String>,
    pub label: Option<String>,
    pub value: Option<String>,
    pub x: f64,
    pub y: f64,
    pub width: f64,
    pub height: f64,
}

#[napi]
pub fn get_focused_element() -> napi::Result<Option<ElementInfo>> {
    let sys = ax::system_wide();
    let focused = match sys.string_attr("AXFocusedUIElement") {
        Some(_) => {
            // Actually need to get the element ref, not string.
            // Simplified: get frontmost app, then focused element.
            None
        }
        None => None,
    };
    Ok(focused)
}

#[napi]
pub fn get_app_windows(pid: i32) -> Vec<ElementInfo> {
    let app = ax::application(pid);
    app.set_timeout(10.0);
    app.windows().iter().map(|w| {
        let (x, y, width, height) = w.frame().unwrap_or((0.0, 0.0, 0.0, 0.0));
        ElementInfo {
            role: w.role(),
            title: w.title(),
            label: w.label(),
            value: None,
            x, y, width, height,
        }
    }).collect()
}

// --- Capture ---

#[napi]
pub fn capture_screen(display_id: Option<u32>) -> napi::Result<napi::bindgen_prelude::Buffer> {
    let id = display_id.unwrap_or_else(|| {
        core_graphics::display::CGDisplay::main().id
    });
    let png_bytes = capture::capture_display(id)
        .map_err(|e| napi::Error::from_reason(e.to_string()))?;
    Ok(png_bytes.into())
}

#[napi]
pub fn capture_window(window_id: u32) -> napi::Result<napi::bindgen_prelude::Buffer> {
    let png_bytes = capture::capture_window(window_id)
        .map_err(|e| napi::Error::from_reason(e.to_string()))?;
    Ok(png_bytes.into())
}

// --- Apps ---

#[napi(object)]
pub struct AppInfoJs {
    pub pid: i32,
    pub bundle_id: Option<String>,
    pub name: Option<String>,
    pub is_active: bool,
    pub is_hidden: bool,
}

#[napi]
pub fn list_applications() -> Vec<AppInfoJs> {
    apps::list_applications().into_iter().map(|a| AppInfoJs {
        pid: a.pid,
        bundle_id: a.bundle_id,
        name: a.name,
        is_active: a.is_active,
        is_hidden: a.is_hidden,
    }).collect()
}

#[napi]
pub fn activate_application(bundle_id: String) -> napi::Result<()> {
    apps::activate_app(&bundle_id)
        .map_err(|e| napi::Error::from_reason(e.to_string()))
}

#[napi]
pub fn launch_application(bundle_id: String) -> napi::Result<()> {
    apps::launch_app(&bundle_id)
        .map_err(|e| napi::Error::from_reason(e.to_string()))
}

#[napi]
pub fn quit_application(bundle_id: String, force: bool) -> napi::Result<bool> {
    apps::quit_app(&bundle_id, force)
        .map_err(|e| napi::Error::from_reason(e.to_string()))
}

// --- Clipboard ---

#[napi]
pub fn read_clipboard() -> Option<String> {
    clipboard::read_clipboard()
}

#[napi]
pub fn write_clipboard(text: String) {
    clipboard::write_clipboard(&text);
}

// --- Permissions ---

#[napi(object)]
pub struct PermissionsStatusJs {
    pub accessibility: bool,
    pub screen_recording: bool,
}

#[napi]
pub fn check_permissions() -> PermissionsStatusJs {
    let status = permissions::check_permissions();
    PermissionsStatusJs {
        accessibility: status.accessibility,
        screen_recording: status.screen_recording,
    }
}
```

---

## Part 4: TypeScript Client

```typescript
// ts/mac-bridge.ts
import {
  mouseMove, click, hotkey, typeText, scroll, drag,
  isAccessibilityTrusted, getAppWindows,
  captureScreen, captureWindow,
  listApplications, activateApplication, launchApplication, quitApplication,
  readClipboard, writeClipboard,
  checkPermissions,
  type ElementInfo, type AppInfoJs, type PermissionsStatusJs,
} from '../index.js'; // napi-generated bindings

// Re-export raw bindings for direct use
export {
  mouseMove, click, hotkey, typeText, scroll, drag,
  readClipboard, writeClipboard,
  checkPermissions,
};

// --- Higher-level typed wrappers ---

export type ClickButton = 'left' | 'right';

export interface Point { x: number; y: number; }
export interface Rect { x: number; y: number; width: number; height: number; }

export interface AppInfo {
  pid: number;
  bundleId: string | null;
  name: string | null;
  isActive: boolean;
  isHidden: boolean;
}

export interface WindowInfo {
  role: string | null;
  title: string | null;
  label: string | null;
  bounds: Rect;
}

export interface Permissions {
  accessibility: boolean;
  screenRecording: boolean;
}

// --- Input ---

export async function clickAt(
  point: Point,
  button: ClickButton = 'left',
  count = 1,
): Promise<void> {
  click(point.x, point.y, button, count);
}

export async function doubleClick(point: Point): Promise<void> {
  click(point.x, point.y, 'left', 2);
}

export async function rightClick(point: Point): Promise<void> {
  click(point.x, point.y, 'right', 1);
}

export async function moveMouse(point: Point): Promise<void> {
  mouseMove(point.x, point.y);
}

export async function pressHotkey(keys: string): Promise<void> {
  hotkey(keys, 0);
}

export async function type(text: string, delayMs = 50): Promise<void> {
  typeText(text, delayMs);
}

export async function scrollDown(amount = 3): Promise<void> {
  scroll(0, -amount);
}

export async function scrollUp(amount = 3): Promise<void> {
  scroll(0, amount);
}

export async function dragFromTo(
  from: Point, to: Point,
  steps = 20, stepDelayMs = 10,
): Promise<void> {
  drag(from.x, from.y, to.x, to.y, steps, stepDelayMs);
}

// --- Screen Capture ---

export async function captureMainDisplay(): Promise<Buffer> {
  return captureScreen(null);
}

export async function captureWindowById(windowId: number): Promise<Buffer> {
  return captureWindow(windowId);
}

// --- Applications ---

export function getApplications(): AppInfo[] {
  return listApplications().map(a => ({
    pid: a.pid,
    bundleId: a.bundleId,
    name: a.name,
    isActive: a.isActive,
    isHidden: a.isHidden,
  }));
}

export async function activateApp(bundleId: string): Promise<void> {
  activateApplication(bundleId);
}

export async function launchApp(bundleId: string): Promise<void> {
  launchApplication(bundleId);
}

export async function quitApp(bundleId: string, force = false): Promise<boolean> {
  return quitApplication(bundleId, force);
}

// --- Windows (via AX) ---

export function getWindows(pid: number): WindowInfo[] {
  return getAppWindows(pid).map(w => ({
    role: w.role,
    title: w.title,
    label: w.label,
    bounds: { x: w.x, y: w.y, width: w.width, height: w.height },
  }));
}

// --- Permissions ---

export function ensurePermissions(): Permissions {
  const status = checkPermissions();
  if (!status.accessibility) {
    console.warn(
      'Accessibility permission not granted. ' +
      'Go to System Settings → Privacy & Security → Accessibility ' +
      'and add this application.'
    );
  }
  if (!status.screenRecording) {
    console.warn(
      'Screen Recording permission not granted. ' +
      'Go to System Settings → Privacy & Security → Screen Recording ' +
      'and add this application.'
    );
  }
  return status;
}
```

---

## Part 5: Agent Usage Example

```typescript
// agent.ts — example OpenClaw-like agent loop
import {
  ensurePermissions, captureMainDisplay,
  clickAt, type, pressHotkey, scrollDown,
  getApplications, activateApp, getWindows,
} from './mac-bridge.js';
import { writeFileSync } from 'fs';

async function agentLoop() {
  // 1. Check permissions on startup
  const perms = ensurePermissions();
  if (!perms.accessibility || !perms.screenRecording) {
    console.error('Missing required permissions. Exiting.');
    process.exit(1);
  }

  // 2. Capture the screen
  const screenshot = await captureMainDisplay();
  writeFileSync('/tmp/screen.png', screenshot);

  // 3. Send to your LLM for analysis
  // const actions = await llm.analyze(screenshot);
  // Example: LLM says "click at (500, 300) then type hello"

  // 4. Execute actions
  await clickAt({ x: 500, y: 300 });
  await type('hello world');
  await pressHotkey('return');

  // 5. Capture again to verify
  const after = await captureMainDisplay();
  writeFileSync('/tmp/screen_after.png', after);
}

agentLoop().catch(console.error);
```

---

## Part 6: Build and Package

### package.json

```json
{
  "name": "@anthropic/mac-bridge",
  "version": "0.1.0",
  "main": "index.js",
  "types": "index.d.ts",
  "napi": {
    "name": "mac-bridge",
    "triples": {
      "defaults": false,
      "additional": [
        "aarch64-apple-darwin",
        "x86_64-apple-darwin"
      ]
    }
  },
  "scripts": {
    "build": "napi build --platform --release",
    "build:debug": "napi build --platform",
    "prepublishOnly": "napi prepublish -t npm"
  },
  "devDependencies": {
    "@napi-rs/cli": "^3.0.0"
  }
}
```

### Build

```bash
# Install napi-rs CLI
npm install -g @napi-rs/cli

# Build the native addon
cd mac-bridge
npm run build

# This produces:
#   mac-bridge.darwin-arm64.node   (for Apple Silicon)
#   mac-bridge.darwin-x64.node     (for Intel)
#   index.js                       (JS loader)
#   index.d.ts                     (TypeScript types)
```

### Linking macOS Frameworks

In `mac-bridge-core/build.rs`:

```rust
fn main() {
    println!("cargo:rustc-link-lib=framework=ApplicationServices");
    println!("cargo:rustc-link-lib=framework=CoreGraphics");
    println!("cargo:rustc-link-lib=framework=AppKit");
    println!("cargo:rustc-link-lib=framework=CoreFoundation");
    // Only link ScreenCaptureKit if targeting macOS 13+
    if std::env::var("MACOSX_DEPLOYMENT_TARGET")
        .map(|v| v >= "13.0")
        .unwrap_or(false)
    {
        println!("cargo:rustc-link-lib=framework=ScreenCaptureKit");
    }
}
```

---

## Comparison: This Approach vs. Peekaboo Bridge

| Aspect | Peekaboo Bridge (existing) | Your Own Rust Bridge |
|---|---|---|
| **Runtime dependency** | Peekaboo.app or daemon must run | None — self-contained |
| **IPC overhead** | UNIX socket per call (~1-5ms) | Zero — in-process function call |
| **TCC permissions** | Held by Peekaboo host | Held by your Node process |
| **Auth complexity** | Code signing + TeamID checks | None needed |
| **Language** | Swift + AXorcist | Rust + objc2/core-graphics |
| **Build complexity** | Just npm install a client | Must compile Rust for macOS |
| **API surface** | 60+ operations pre-built | Build what you need |
| **ScreenCaptureKit** | Full async SCK support | CGWindowList initially; SCK requires ObjC bridging |
| **AX tree walking** | AXorcist (mature, timeout-aware) | Hand-rolled AXUIElement FFI |
| **Maintenance** | Peekaboo team maintains APIs | You maintain macOS API bindings |

### Recommendation for Incremental Build

1. **Start with input.rs + capture.rs + apps.rs** — these give you mouse,
   keyboard, screenshots, and app management with the simplest APIs.
2. **Add ax.rs** when you need element-aware clicking (by role/title) instead
   of coordinate-only clicking.
3. **Skip ScreenCaptureKit initially** — CGWindowList works fine and avoids
   the ObjC async bridging complexity. Add SCK later if you need the quality
   improvement or to avoid the screen recording "badge" indicator.
4. **Menu traversal** is the most complex part (deep AX tree walks with
   timeouts). Defer it unless your agent specifically needs menu interaction.
