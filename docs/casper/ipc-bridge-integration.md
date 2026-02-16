# Integrating with Peekaboo via IPC Bridge

This document explains how to connect a non-Swift agent (e.g., a JavaScript client with Rust device bridges) to Peekaboo's Mac automation capabilities over IPC, without shelling out to the `peekaboo` CLI.

## Architecture Overview

```
┌─────────────────────────────────────┐
│  Your Agent (JS + Rust)             │
│                                     │
│  JS orchestrator                    │
│    ↓                                │
│  Rust bridge layer (napi-rs / FFI)  │
│    ↓                                │
│  UNIX domain socket client          │
│    (connect, write JSON, read JSON) │
└──────────────┬──────────────────────┘
               │ ~/.../Peekaboo/bridge.sock
               ▼
┌─────────────────────────────────────┐
│  Peekaboo Host (daemon or .app)     │
│                                     │
│  PeekabooBridgeHost (socket listen) │
│    ↓                                │
│  PeekabooBridgeServer (JSON router) │
│    ↓                                │
│  PeekabooServices (local)           │
│    ├─ ScreenCaptureKit              │
│    ├─ AXUIElement (AXorcist)        │
│    ├─ CGEvent (mouse/keyboard)      │
│    └─ NSWorkspace, etc.             │
└─────────────────────────────────────┘
```

## Prerequisites

1. **Peekaboo host must be running** — either:
   - Peekaboo.app (hosts the bridge in GUI mode), or
   - `peekaboo daemon` (headless bridge server)
2. **TCC permissions** on the host process:
   - Screen Recording (for capture operations)
   - Accessibility (for click, type, scroll, window, menu operations)
   - Automation/AppleScript (for app launch/quit/activate)
3. **Client authentication** — one of:
   - Your process is signed with an allowlisted Team ID (production)
   - Host is launched with `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` (development)

## Socket Path

The default socket path is:

```
~/Library/Application Support/Peekaboo/bridge.sock
```

Fallbacks (checked by the Swift client in order):
- `~/Library/Application Support/Claude/bridge.sock`
- `~/Library/Application Support/clawdbot/bridge.sock`

## Connection Protocol

**Single-request-per-connection**. For each operation:

1. Connect to the UNIX domain socket (`AF_UNIX`, `SOCK_STREAM`)
2. Write the full JSON request body
3. Half-close the write side (`shutdown(fd, SHUT_WR)`)
4. Read the full JSON response
5. Close the socket

This means every operation opens a new connection. There is no multiplexing or keepalive.

## JSON Wire Format

### Top-Level Envelope

The request and response are Swift `Codable` enums with auto-synthesized encoding. The wire format is:

**Cases with associated values:**
```json
{"caseName": {"_0": <payload>}}
```

**Cases without associated values:**
```json
{"caseName": {}}
```

### Handshake (required first call)

**Request:**
```json
{
  "handshake": {
    "_0": {
      "protocolVersion": {"major": 1, "minor": 0},
      "client": {
        "bundleIdentifier": "com.example.myagent",
        "teamIdentifier": null,
        "processIdentifier": 12345,
        "hostname": null
      },
      "requestedHostKind": null
    }
  }
}
```

**Response (success):**
```json
{
  "handshake": {
    "_0": {
      "negotiatedVersion": {"major": 1, "minor": 0},
      "hostKind": "gui",
      "build": "3.0.0 (42)",
      "supportedOperations": ["captureScreen", "click", "type", ...],
      "permissions": {
        "screenRecording": true,
        "accessibility": true,
        "appleScript": false
      },
      "enabledOperations": ["captureScreen", "click", "type", ...],
      "permissionTags": {
        "captureScreen": ["screenRecording"],
        "click": ["accessibility"]
      }
    }
  }
}
```

### Screen Capture

**Request:**
```json
{
  "captureScreen": {
    "_0": {
      "displayIndex": null,
      "visualizerMode": "screenshotFlash",
      "scale": "logical1x"
    }
  }
}
```

**Response:**
```json
{
  "capture": {
    "_0": {
      "imageData": "<base64-encoded PNG>",
      "savedPath": "/path/to/screenshot.png",
      "metadata": {
        "size": {"width": 1920, "height": 1080},
        "timestamp": "2026-02-15T10:30:00Z"
      }
    }
  }
}
```

### Click

**Request (by element ID):**
```json
{
  "click": {
    "_0": {
      "target": {"kind": "elementId", "value": "B1"},
      "clickType": "single",
      "snapshotId": "snap-abc123"
    }
  }
}
```

**Request (by coordinates):**
```json
{
  "click": {
    "_0": {
      "target": {"kind": "coordinates", "x": 500.0, "y": 300.0},
      "clickType": "double",
      "snapshotId": null
    }
  }
}
```

**Request (by text query):**
```json
{
  "click": {
    "_0": {
      "target": {"kind": "query", "value": "Submit"},
      "clickType": "single",
      "snapshotId": null
    }
  }
}
```

**Response (success):**
```json
{"ok": {}}
```

### Type Text

**Request:**
```json
{
  "type": {
    "_0": {
      "text": "Hello, world!",
      "target": null,
      "clearExisting": false,
      "typingDelay": 50,
      "snapshotId": null
    }
  }
}
```

### Hotkey

**Request:**
```json
{
  "hotkey": {
    "_0": {
      "keys": "cmd+shift+s",
      "holdDuration": 0
    }
  }
}
```

### Scroll

**Request:**
```json
{
  "scroll": {
    "_0": {
      "request": {
        "direction": "down",
        "amount": 3,
        "target": null,
        "smooth": false,
        "delay": 10,
        "snapshotId": null
      }
    }
  }
}
```

### List Windows

**Request:**
```json
{
  "listWindows": {
    "_0": {
      "target": {"kind": "application", "app": "com.apple.Safari"}
    }
  }
}
```

**Response:**
```json
{
  "windows": {
    "_0": [
      {
        "window_id": 9001,
        "title": "Google",
        "bounds": [100, 100, 800, 600],
        "isMinimized": false,
        "isMainWindow": true
      }
    ]
  }
}
```

### Focus / Move / Resize Window

```json
{
  "focusWindow": {
    "_0": {
      "target": {"kind": "frontmost"}
    }
  }
}
```

```json
{
  "moveWindow": {
    "_0": {
      "target": {"kind": "title", "title": "My Doc"},
      "position": [200, 100]
    }
  }
}
```

### List / Launch / Quit Applications

```json
{"listApplications": {}}
```

```json
{
  "launchApplication": {
    "_0": {"identifier": "com.apple.Safari"}
  }
}
```

```json
{
  "quitApplication": {
    "_0": {"identifier": "com.apple.TextEdit", "force": false}
  }
}
```

### Menu Operations

```json
{
  "listFrontmostMenus": {}
}
```

```json
{
  "clickMenuItem": {
    "_0": {
      "appIdentifier": "com.apple.Safari",
      "itemPath": "File > New Window"
    }
  }
}
```

### No-Payload Cases

These require no arguments:

```json
{"permissionsStatus": {}}
{"daemonStatus": {}}
{"getFocusedWindow": {}}
{"listApplications": {}}
{"getFrontmostApplication": {}}
{"showAllApplications": {}}
{"listFrontmostMenus": {}}
{"listMenuExtras": {}}
{"listSnapshots": {}}
{"hideDock": {}}
{"showDock": {}}
{"isDockHidden": {}}
{"cleanAllSnapshots": {}}
```

### Error Response

Any operation can return an error:
```json
{
  "error": {
    "_0": {
      "code": "permissionDenied",
      "message": "Operation click requires accessibility permission",
      "details": null
    }
  }
}
```

Error codes: `permissionDenied`, `notFound`, `timeout`, `invalidRequest`,
`operationNotSupported`, `serverBusy`, `versionMismatch`,
`unauthorizedClient`, `decodingFailed`, `internalError`.

## Nested Type Encodings

### ClickTarget
```json
{"kind": "elementId",   "value": "B1"}
{"kind": "coordinates", "x": 100.0, "y": 200.0}
{"kind": "query",       "value": "Submit button"}
```

### WindowTarget
```json
{"kind": "application",         "app": "com.apple.Safari"}
{"kind": "title",               "title": "My Document"}
{"kind": "index",               "app": "com.apple.TextEdit", "index": 0}
{"kind": "applicationAndTitle", "app": "com.apple.TextEdit", "title": "Readme"}
{"kind": "frontmost"}
{"kind": "windowId",            "windowId": 9001}
```

### ClickType (string enum)
`"single"`, `"double"`, `"right"`, `"tripleClick"`

### ScrollDirection (string enum)
`"up"`, `"down"`, `"left"`, `"right"`

### CaptureVisualizerMode (string enum)
`"screenshotFlash"`, `"watchCapture"`

### CaptureScalePreference (string enum)
`"logical1x"`, `"native"`

### MouseMovementProfile
```json
{"kind": "linear"}
{"kind": "human", "profile": {
  "jitterAmplitude": 0.35,
  "overshootProbability": 0.2,
  "overshootFractionRange": [0.02, 0.06],
  "settleRadius": 6.0,
  "randomSeed": null
}}
```

### TypeAction
```json
{"kind": "text",  "text": "Hello"}
{"kind": "key",   "key": "return"}
{"kind": "clear"}
```

### TypingCadence
```json
{"kind": "fixed", "milliseconds": 50}
{"kind": "human", "wordsPerMinute": 60}
```

## Complete List of Operations

| Operation | Request payload | Response type |
|---|---|---|
| `handshake` | `PeekabooBridgeHandshake` | `handshake` |
| `permissionsStatus` | (none) | `permissionsStatus` |
| `daemonStatus` | (none) | `daemonStatus` |
| `daemonStop` | (none) | `bool` |
| `captureScreen` | displayIndex, visualizerMode, scale | `capture` |
| `captureWindow` | appIdentifier, windowIndex, windowId, ... | `capture` |
| `captureFrontmost` | visualizerMode, scale | `capture` |
| `captureArea` | rect, visualizerMode, scale | `capture` |
| `detectElements` | imageData, snapshotId, windowContext | `elementDetection` |
| `click` | target, clickType, snapshotId | `ok` |
| `type` | text, target, clearExisting, typingDelay, snapshotId | `ok` |
| `typeActions` | actions, cadence, snapshotId | `typeResult` |
| `scroll` | request (direction, amount, ...) | `ok` |
| `hotkey` | keys, holdDuration | `ok` |
| `swipe` | from, to, duration, steps, profile | `ok` |
| `drag` | from, to, duration, steps, modifiers, profile | `ok` |
| `moveMouse` | to, duration, steps, profile | `ok` |
| `waitForElement` | target, timeout, snapshotId | `waitResult` |
| `listWindows` | target | `windows` |
| `focusWindow` | target | `ok` |
| `moveWindow` | target, position | `ok` |
| `resizeWindow` | target, size | `ok` |
| `setWindowBounds` | target, bounds | `ok` |
| `closeWindow` | target | `ok` |
| `minimizeWindow` | target | `ok` |
| `maximizeWindow` | target | `ok` |
| `getFocusedWindow` | (none) | `window` |
| `listApplications` | (none) | `applications` |
| `findApplication` | identifier | `application` |
| `getFrontmostApplication` | (none) | `application` |
| `isApplicationRunning` | identifier | `bool` |
| `launchApplication` | identifier | `application` |
| `activateApplication` | identifier | `ok` |
| `quitApplication` | identifier, force | `bool` |
| `hideApplication` | identifier | `ok` |
| `unhideApplication` | identifier | `ok` |
| `hideOtherApplications` | identifier | `ok` |
| `showAllApplications` | (none) | `ok` |
| `listMenus` | appIdentifier | `menuStructure` |
| `listFrontmostMenus` | (none) | `menuStructure` |
| `clickMenuItem` | appIdentifier, itemPath | `ok` |
| `clickMenuItemByName` | appIdentifier, itemName | `ok` |
| `listMenuExtras` | (none) | `menuExtras` |
| `clickMenuExtra` | name | `ok` |
| `listMenuBarItems` | Bool (includeRaw) | `menuBarItems` |
| `clickMenuBarItemNamed` | name | `clickResult` |
| `clickMenuBarItemIndex` | index | `clickResult` |
| `listDockItems` | includeAll | `dockItems` |
| `launchDockItem` | appName | `ok` |
| `rightClickDockItem` | appName, menuItem | `ok` |
| `hideDock` | (none) | `ok` |
| `showDock` | (none) | `ok` |
| `isDockHidden` | (none) | `bool` |
| `findDockItem` | name | `dockItem` |
| `dialogFindActive` | windowTitle, appName | `dialogInfo` |
| `dialogClickButton` | buttonText, windowTitle, appName | `dialogResult` |
| `dialogEnterText` | text, fieldIdentifier, clearExisting, windowTitle, appName | `dialogResult` |
| `dialogHandleFile` | path, filename, actionButton, ensureExpanded, appName | `dialogResult` |
| `dialogDismiss` | force, windowTitle, appName | `dialogResult` |
| `dialogListElements` | windowTitle, appName | `dialogElements` |
| `createSnapshot` | (none) | `snapshotId` |
| `listSnapshots` | (none) | `snapshots` |
| `getMostRecentSnapshot` | applicationBundleId | `snapshotId` |
| `cleanSnapshot` | snapshotId | `ok` |
| `cleanSnapshotsOlderThan` | days | `int` |
| `cleanAllSnapshots` | (none) | `int` |

## Implementation: Rust Bridge Client

Here's a sketch of a Rust client that connects to the Peekaboo bridge:

```rust
use serde::{Deserialize, Serialize};
use serde_json::Value;
use std::io::{Read, Write};
use std::os::unix::net::UnixStream;
use std::path::PathBuf;
use std::time::Duration;

/// Socket path for Peekaboo bridge
fn socket_path() -> PathBuf {
    let home = std::env::var("HOME").unwrap();
    PathBuf::from(format!(
        "{}/Library/Application Support/Peekaboo/bridge.sock",
        home
    ))
}

/// Send a single request to the Peekaboo bridge and return the response.
fn bridge_request(request_json: &Value) -> Result<Value, Box<dyn std::error::Error>> {
    let path = socket_path();
    let mut stream = UnixStream::connect(&path)?;
    stream.set_read_timeout(Some(Duration::from_secs(10)))?;

    // Write full request
    let payload = serde_json::to_vec(request_json)?;
    stream.write_all(&payload)?;

    // Half-close write side
    stream.shutdown(std::net::Shutdown::Write)?;

    // Read full response
    let mut response_bytes = Vec::new();
    stream.read_to_end(&mut response_bytes)?;

    let response: Value = serde_json::from_slice(&response_bytes)?;
    Ok(response)
}

/// Perform the handshake
fn handshake() -> Result<Value, Box<dyn std::error::Error>> {
    let req = serde_json::json!({
        "handshake": {
            "_0": {
                "protocolVersion": {"major": 1, "minor": 0},
                "client": {
                    "bundleIdentifier": "com.example.myagent",
                    "teamIdentifier": null,
                    "processIdentifier": std::process::id(),
                    "hostname": null
                },
                "requestedHostKind": null
            }
        }
    });
    bridge_request(&req)
}

/// Capture the screen
fn capture_screen() -> Result<Value, Box<dyn std::error::Error>> {
    let req = serde_json::json!({
        "captureScreen": {
            "_0": {
                "displayIndex": null,
                "visualizerMode": "screenshotFlash",
                "scale": "logical1x"
            }
        }
    });
    bridge_request(&req)
}

/// Click an element by ID
fn click_element(element_id: &str, snapshot_id: Option<&str>) -> Result<Value, Box<dyn std::error::Error>> {
    let req = serde_json::json!({
        "click": {
            "_0": {
                "target": {"kind": "elementId", "value": element_id},
                "clickType": "single",
                "snapshotId": snapshot_id
            }
        }
    });
    bridge_request(&req)
}

/// Click at coordinates
fn click_at(x: f64, y: f64) -> Result<Value, Box<dyn std::error::Error>> {
    let req = serde_json::json!({
        "click": {
            "_0": {
                "target": {"kind": "coordinates", "x": x, "y": y},
                "clickType": "single",
                "snapshotId": null
            }
        }
    });
    bridge_request(&req)
}

/// Type text
fn type_text(text: &str) -> Result<Value, Box<dyn std::error::Error>> {
    let req = serde_json::json!({
        "type": {
            "_0": {
                "text": text,
                "target": null,
                "clearExisting": false,
                "typingDelay": 50,
                "snapshotId": null
            }
        }
    });
    bridge_request(&req)
}

/// Press a hotkey combination
fn hotkey(keys: &str) -> Result<Value, Box<dyn std::error::Error>> {
    let req = serde_json::json!({
        "hotkey": {
            "_0": {
                "keys": keys,
                "holdDuration": 0
            }
        }
    });
    bridge_request(&req)
}

/// List all windows for an app
fn list_windows(app_bundle_id: &str) -> Result<Value, Box<dyn std::error::Error>> {
    let req = serde_json::json!({
        "listWindows": {
            "_0": {
                "target": {"kind": "application", "app": app_bundle_id}
            }
        }
    });
    bridge_request(&req)
}

/// Check if response is an error
fn is_error(response: &Value) -> bool {
    response.get("error").is_some()
}

/// Check if response is ok
fn is_ok(response: &Value) -> bool {
    response.get("ok").is_some()
}
```

## Implementation: Node.js Client

If you prefer to connect from the JavaScript layer directly:

```javascript
const net = require("net");
const path = require("path");
const os = require("os");

const SOCKET_PATH = path.join(
  os.homedir(),
  "Library/Application Support/Peekaboo/bridge.sock"
);

/**
 * Send a single request to the Peekaboo bridge.
 * Returns a promise that resolves with the parsed JSON response.
 */
function bridgeRequest(request) {
  return new Promise((resolve, reject) => {
    const client = net.createConnection(SOCKET_PATH, () => {
      const payload = JSON.stringify(request);
      client.end(payload); // write + half-close
    });

    const chunks = [];
    client.on("data", (chunk) => chunks.push(chunk));
    client.on("end", () => {
      try {
        const response = JSON.parse(Buffer.concat(chunks).toString());
        resolve(response);
      } catch (err) {
        reject(new Error(`Failed to parse bridge response: ${err.message}`));
      }
    });
    client.on("error", reject);
    client.setTimeout(10000, () => {
      client.destroy(new Error("Bridge request timed out"));
    });
  });
}

// --- Convenience wrappers ---

async function handshake() {
  return bridgeRequest({
    handshake: {
      _0: {
        protocolVersion: { major: 1, minor: 0 },
        client: {
          bundleIdentifier: "com.example.myagent",
          teamIdentifier: null,
          processIdentifier: process.pid,
          hostname: null,
        },
        requestedHostKind: null,
      },
    },
  });
}

async function captureScreen() {
  return bridgeRequest({
    captureScreen: {
      _0: {
        displayIndex: null,
        visualizerMode: "screenshotFlash",
        scale: "logical1x",
      },
    },
  });
}

async function clickElement(elementId, snapshotId = null) {
  return bridgeRequest({
    click: {
      _0: {
        target: { kind: "elementId", value: elementId },
        clickType: "single",
        snapshotId,
      },
    },
  });
}

async function clickAt(x, y) {
  return bridgeRequest({
    click: {
      _0: {
        target: { kind: "coordinates", x, y },
        clickType: "single",
        snapshotId: null,
      },
    },
  });
}

async function typeText(text, opts = {}) {
  return bridgeRequest({
    type: {
      _0: {
        text,
        target: opts.target ?? null,
        clearExisting: opts.clearExisting ?? false,
        typingDelay: opts.typingDelay ?? 50,
        snapshotId: opts.snapshotId ?? null,
      },
    },
  });
}

async function hotkey(keys) {
  return bridgeRequest({
    hotkey: { _0: { keys, holdDuration: 0 } },
  });
}

async function listWindows(appBundleId) {
  return bridgeRequest({
    listWindows: {
      _0: { target: { kind: "application", app: appBundleId } },
    },
  });
}

async function listApplications() {
  return bridgeRequest({ listApplications: {} });
}

function isOk(response) {
  return "ok" in response;
}
function isError(response) {
  return "error" in response;
}
function getError(response) {
  return response?.error?._0;
}
```

## Typical Agent Workflow

A typical automation loop for an OpenClaw-like agent:

```javascript
// 1. Handshake
const hs = await handshake();
const enabled = hs.handshake._0.enabledOperations;
console.log("Enabled operations:", enabled);

// 2. Capture the screen and detect elements
const capture = await captureScreen();
const imageData = capture.capture._0.imageData; // base64 PNG

// 3. Send screenshot to your LLM for analysis
// ... LLM returns: "click element B1" ...

// 4. Click the element
const result = await clickElement("B1", snapshotId);

// 5. Type into a field
await typeText("search query");

// 6. Press Enter
await hotkey("return");

// 7. Capture again to see the result
const after = await captureScreen();
```

## Alternative: MCP over stdio

If your agent already speaks MCP (Model Context Protocol), you can skip the bridge entirely and spawn `peekaboo mcp` as a subprocess:

```javascript
const { spawn } = require("child_process");

const mcp = spawn("peekaboo", ["mcp"], {
  stdio: ["pipe", "pipe", "inherit"],
});

// Now speak JSON-RPC over stdin/stdout
// Tools available: see, click, type, scroll, hotkey, app, window, menu, etc.
```

This gives you the same 25 automation tools but over the standard MCP JSON-RPC protocol instead of the bridge's custom envelope. The downside is you need the `peekaboo` binary installed and it spawns a new process.

## Choosing Between Bridge vs MCP vs CLI

| Approach | Latency | Protocol | Language support | Auth |
|---|---|---|---|---|
| **Bridge socket** | ~1-5ms per call | JSON over UNIX socket | Any language | Code signing |
| **MCP stdio** | ~5-20ms per call | JSON-RPC over stdin/out | Any MCP client | Process-level |
| **CLI subprocess** | ~100-500ms per call | exec + parse stdout | Any language | Process-level |

For an agent making many rapid automation calls, the **bridge socket** is the best choice. It eliminates process spawn overhead entirely and gives you the lowest latency path to macOS automation APIs.
