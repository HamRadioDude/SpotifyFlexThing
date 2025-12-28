# Ultimate DeskThing App Development Guide
## Complete Technical Reference for Building Car Thing Applications

**Version:** 2.1  
**Date:** December 2025  
**Based on:** DeskThing Server v0.11.17, SDK v0.11.6  
**Derived from:** Extensive real-world debugging of FlexRadio controller app (3200+ lines)  
**Updated:** Added Key Registration section and modes bug documentation based on FlexThing v2.0.6 scroll fix

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Critical Rules - READ FIRST](#2-critical-rules---read-first)
3. [Project Structure](#3-project-structure)
4. [Server Development](#4-server-development)
5. [Handler Registration - THE MOST IMPORTANT SECTION](#5-handler-registration---the-most-important-section)
6. [Client Development](#6-client-development)
7. [Actions and Button Mapping](#7-actions-and-button-mapping)
8. [Key Registration - CRITICAL FOR SCROLL/BUTTONS](#8-key-registration---critical-for-scrollbuttons)
9. [Settings System](#9-settings-system)
10. [State Management](#10-state-management)
11. [Network Communication Patterns](#11-network-communication-patterns)
12. [Real-Time Data Streaming](#12-real-time-data-streaming)
13. [Multi-Screen Applications](#13-multi-screen-applications)
14. [Complete Server Template](#14-complete-server-template)
15. [Complete Client Template](#15-complete-client-template)
16. [Debugging Guide](#16-debugging-guide)
17. [Common Bugs and Solutions](#17-common-bugs-and-solutions)
18. [Build and Deployment](#18-build-and-deployment)
19. [Quick Reference Cheat Sheet](#19-quick-reference-cheat-sheet)

---

## 1. Architecture Overview

### System Components

```
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│  External API   │ ◄─────► │  Your Server    │ ◄─────► │   Car Thing     │
│  (Radio/Device) │   TCP   │  (Your PC)      │  WS/IPC │   (Display)     │
│  (Web API)      │   UDP   │  Node.js        │         │   React App     │
└─────────────────┘         └─────────────────┘         └─────────────────┘
                            DeskThing.send() ──────────► DeskThing.on()
                            DeskThing.on()   ◄────────── deskthing.send()
```

### Data Flow Rules

1. **Server** runs on your PC - has full network/filesystem access
2. **Client** runs on Car Thing - is a sandboxed React app
3. **Client CANNOT** access external APIs, the internet, or filesystem
4. **ALL external data** must flow through the server
5. **Communication** is bidirectional via DeskThing messaging

### Message Types

| Direction | Method | Purpose |
|-----------|--------|---------|
| Server → Client | `DeskThing.send({ type, payload })` | Send data to display |
| Client → Server | `deskthing.send({ type, request, payload })` | Request data or trigger action |
| Server receives | `DeskThing.on("messageType", handler)` | Handle client requests |
| Client receives | `deskthing.on("messageType", handler)` | Handle server data |

---

## 2. Critical Rules - READ FIRST

### ⚠️ YOUR APP WILL CRASH OR MALFUNCTION WITHOUT THESE:

### Rule 1: Use DESKTHING_EVENTS Enum for START/STOP
```javascript
// ❌ WRONG - Will crash
DeskThing.on('start', start)

// ✅ CORRECT
import { DESKTHING_EVENTS } from '@deskthing/types'
DeskThing.on(DESKTHING_EVENTS.START, start)
DeskThing.on(DESKTHING_EVENTS.STOP, stop)
```

### Rule 2: Call initializeSettings() First
```javascript
const start = async () => {
  await initializeSettings();  // MUST be first!
  await initializeActions();   // Then actions
  // Then everything else
};
```

### Rule 3: Use Async Handlers
```javascript
const start = async () => { };
const stop = async () => { };
```

### Rule 4: Only START and STOP at Module Level
```javascript
// At the END of your file, only these two:
DeskThing.on(DESKTHING_EVENTS.START, start);
DeskThing.on(DESKTHING_EVENTS.STOP, stop);

// All other handlers go in registerAllHandlers()
```

### Rule 5: Register Handlers ONLY ONCE
```javascript
// ❌ WRONG - Handlers inside start() accumulate on every restart!
const start = async () => {
  DeskThing.on("buttonClick", () => { /* fires 2x, 3x, 4x... */ });
};

// ✅ CORRECT - Use a registration guard
let handlersRegistered = false;

const start = async () => {
  if (!handlersRegistered) {
    handlersRegistered = true;
    registerAllHandlers();
  }
  await doInitialize();
};
```

### Rule 6: Wrap Async in Sync for Intervals
```javascript
// ❌ WRONG
setInterval(async () => { await fetch(url) }, 30000)

// ✅ CORRECT
const doFetch = async () => { await fetch(url) }
const fetchData = () => { doFetch() }  // No await!
setInterval(fetchData, 30000)
```

### Rule 7: Icon Must Match App ID
If manifest has `"id": "my-app"`, icon must be at `icons/my-app.svg`

### Rule 8: Include package.json with type: module
Both root and server directories need:
```json
{"type": "module"}
```

### Rule 9: NEVER Use modes: ["default"] in Key Registration
```javascript
// ❌ WRONG - "default" is NOT a valid EventMode, silently breaks key routing!
const keys = [
  { id: "scroll", description: "...", modes: ["default"] }
];

// ✅ CORRECT - Omit modes entirely
const keys = [
  { id: "scroll", description: "Wheel scroll" }
];
```
This bug causes scroll/buttons to work globally but NOT when your app is focused.

---

## 3. Project Structure

### Standard Layout
```
my-app/
├── manifest.json              # App metadata (REQUIRED)
├── icons/
│   └── my-app.svg             # App icon (REQUIRED - matches manifest id)
├── server/
│   ├── index.js               # Compiled server code
│   └── package.json           # {"type": "module"}
├── client/
│   ├── index.html             # Client entry point
│   ├── index-*.js             # Compiled React app
│   ├── index-*.css            # Compiled styles
│   ├── polyfills-legacy-*.js  # Legacy browser support
│   └── icons/
│       └── my-app.svg         # Client-side icon copy
└── package.json               # {"type": "module"}
```

### Development Layout (Before Build)
```
my-app/
├── deskthing/
│   └── manifest.json
├── public/
│   └── icons/
│       └── my-app.svg
├── server/
│   └── index.ts
├── src/
│   ├── App.tsx
│   ├── main.tsx
│   └── index.css
├── index.html
├── package.json
├── vite.config.ts
├── tsconfig.json
├── tailwind.config.js
└── postcss.config.js
```

---

## 4. Server Development

### Server File Structure

The server file should be organized in this order:

```javascript
// 1. IMPORTS
import { DeskThing } from '@deskthing/server';

// 2. CONSTANTS
const BAND_FREQUENCIES = { "160m": 1800000, "80m": 3500000 };
const MODES = ["USB", "LSB", "CW", "AM"];
const TUNING_STEPS = [1, 10, 100, 1000, 10000];

// 3. STATE
var state = {
  connected: false,
  frequency: 14074000,
  mode: "USB",
  // ... all state variables
};

// 4. MODULE-LEVEL VARIABLES
var socket = null;
var pingInterval = null;
var handlersRegistered = false;  // CRITICAL: Handler guard

// 5. HELPER FUNCTIONS
var sendStateToClient = () => {
  DeskThing.send({ type: "flexState", payload: state });
};

var sendCommand = (cmd) => {
  if (socket && state.connected) {
    socket.write(cmd + "\n");
  }
};

// 6. INITIALIZATION FUNCTIONS
var initializeSettings = async () => { /* ... */ };
var initializeActions = async () => { /* ... */ };

// 7. BUSINESS LOGIC FUNCTIONS
var connect = () => { /* ... */ };
var processData = (data) => { /* ... */ };

// 8. START FUNCTION
var start = async () => {
  console.log("[MyApp] Starting...");
  await initializeSettings();
  await initializeActions();
  
  if (!handlersRegistered) {
    handlersRegistered = true;
    registerAllHandlers();
  }
  
  await doInitialize();
};

// 9. HANDLER REGISTRATION FUNCTION
var registerAllHandlers = () => {
  // ALL DeskThing.on() handlers go here
  DeskThing.on("actionName", () => { /* ... */ });
  DeskThing.on(DESKTHING_EVENTS.ACTION, (data) => { /* ... */ });
};

// 10. STOP FUNCTION
var stop = async () => {
  console.log("[MyApp] Stopping...");
  // Cleanup all intervals and connections
};

// 11. MODULE-LEVEL EVENT REGISTRATION (ONLY START AND STOP!)
DeskThing.on(DESKTHING_EVENTS.START, start);
DeskThing.on(DESKTHING_EVENTS.STOP, stop);
```

---

## 5. Handler Registration - THE MOST IMPORTANT SECTION

### The Handler Accumulation Bug

This is the #1 cause of "double-firing" bugs. When handlers are registered inside `start()`, they accumulate every time the app restarts:

```javascript
// ❌ BUG: Every restart adds another handler
var start = async () => {
  DeskThing.on("bandUp", () => {
    // First start: fires 1x
    // After restart: fires 2x
    // After another restart: fires 3x
    changeBand(1);
  });
};
```

### The Correct Pattern

```javascript
var handlersRegistered = false;

var start = async () => {
  console.log("[MyApp] Starting...");
  await initializeSettings();
  await initializeActions();
  sendStateToClient();
  
  // Guard: Only register handlers once, ever
  if (!handlersRegistered) {
    handlersRegistered = true;
    console.log("[MyApp] Registering handlers (first time)...");
    registerAllHandlers();
  } else {
    console.log("[MyApp] Handlers already registered, skipping...");
  }
  
  // Always do initialization (connect, fetch data, etc.)
  await doInitialize();
};

// Separate function containing ALL handlers
var registerAllHandlers = () => {
  // On-screen button handlers
  DeskThing.on("bandUp", () => { changeBand(1); });
  DeskThing.on("bandDown", () => { changeBand(-1); });
  DeskThing.on("cycleMode", () => { cycleMode(); });
  
  // Physical button ACTION handler
  DeskThing.on(DESKTHING_EVENTS.ACTION, (data) => {
    const actionId = data?.payload?.id || data?.id;
    switch (actionId) {
      case "tuneUp": tune(1); break;
      case "tuneDown": tune(-1); break;
    }
  });
  
  // Scroll wheel handler
  DeskThing.on("scroll", (data) => {
    const direction = data?.payload?.direction || 0;
    if (direction !== 0) tune(direction);
  });
  
  // Other handlers...
  DeskThing.on("getState", () => { sendStateToClient(); });
};
```

### Two Types of Button Events

There are TWO distinct paths for button events:

#### Path 1: On-Screen Button Clicks (from Client UI)
```javascript
// Client sends:
deskthing.send({ type: "bandUp" });

// Server receives via:
DeskThing.on("bandUp", () => {
  console.log("On-screen button clicked");
  changeBand(1);
});
```

#### Path 2: Mapped Physical Buttons (from DeskThing server)
```javascript
// User presses physical button mapped to "bandUp" action
// DeskThing server sends ACTION event

// Server receives via:
DeskThing.on(DESKTHING_EVENTS.ACTION, (data) => {
  const actionId = data?.payload?.id;
  if (actionId === "bandUp") {
    console.log("Physical button pressed");
    changeBand(1);
  }
});
```

#### CRITICAL: These Are NOT Duplicates!

If your action works from both on-screen buttons AND physical buttons, you need BOTH handlers. They serve different input sources.

If you only want physical button support, use only the ACTION handler.
If you only want on-screen button support, use only the direct handler.

---

## 6. Client Development

### Client Architecture

```typescript
import { useEffect, useState, useCallback } from 'react';
import { DeskThing } from '@deskthing/client';

const deskthing = DeskThing;

// Define your state interface
interface AppState {
  connected: boolean;
  frequency: number;
  mode: string;
  sMeter: number;
}

function App() {
  const [state, setState] = useState<AppState>({
    connected: false,
    frequency: 14074000,
    mode: "USB",
    sMeter: -100
  });
  const [currentScreen, setCurrentScreen] = useState<string>("main");

  useEffect(() => {
    // Set up listeners
    const offState = deskthing.on("appState", (data: any) => {
      if (data?.payload) {
        setState(data.payload);
      }
    });

    const offScreen = deskthing.on("screenChange", (data: any) => {
      if (data?.payload?.screen) {
        setCurrentScreen(data.payload.screen);
      }
    });

    // Request initial state after small delay
    setTimeout(() => {
      deskthing.send({ type: "getState" });
    }, 500);

    // Cleanup on unmount
    return () => {
      offState();
      offScreen();
    };
  }, []);

  // Button click handler
  const handleButtonClick = useCallback((action: string) => {
    deskthing.send({ type: action });
  }, []);

  return (
    <div className="app-container">
      {/* Your UI here */}
    </div>
  );
}
```

### Client Communication Patterns

#### Sending Messages to Server
```typescript
// Simple action trigger
deskthing.send({ type: "bandUp" });

// Action with payload
deskthing.send({ 
  type: "setFrequency", 
  payload: { frequency: 14074000 } 
});

// Request data
deskthing.send({ type: "getState" });
deskthing.send({ type: "getPotaSpots" });
```

#### Receiving Messages from Server
```typescript
useEffect(() => {
  // Listen for specific message types
  const offState = deskthing.on("appState", (data) => {
    if (data?.payload) {
      setState(data.payload);
    }
  });

  const offSpots = deskthing.on("potaSpots", (data) => {
    if (data?.payload && Array.isArray(data.payload)) {
      setSpots(data.payload);
    }
  });

  return () => {
    offState();
    offSpots();
  };
}, []);
```

### Multi-Screen Navigation

```typescript
const [currentScreen, setCurrentScreen] = useState("vfo");

// Render different screens
const renderScreen = () => {
  switch (currentScreen) {
    case "vfo": return <VFOScreen state={state} onAction={handleAction} />;
    case "dsp": return <DSPScreen state={state} onAction={handleAction} />;
    case "memory": return <MemoryScreen state={state} onAction={handleAction} />;
    default: return <VFOScreen state={state} onAction={handleAction} />;
  }
};

// Listen for screen change from server
useEffect(() => {
  const offScreen = deskthing.on("screenChange", (data) => {
    if (data?.payload?.screen) {
      setCurrentScreen(data.payload.screen);
    }
  });
  return () => offScreen();
}, []);
```

---

## 7. Actions and Button Mapping

### Registering Actions

Actions are what users can map to physical buttons on the Car Thing.

```javascript
var initializeActions = async () => {
  // Simple action (no value needed)
  DeskThing.registerAction({
    id: "bandUp",
    name: "Band Up",
    description: "Switch to next higher band",
    tag: "radio",    // Category
    version: "1.0.0",
    version_code: 1,
    enabled: true
  });

  // Action with value options
  DeskThing.registerAction({
    id: "setBand",
    name: "Set Band",
    description: "Set a specific band",
    tag: "radio",
    version: "1.0.0",
    version_code: 1,
    enabled: true,
    value_options: ["160m", "80m", "40m", "20m", "15m", "10m"],
    value_instructions: "Select the band to switch to",
    value: "20m"  // Default value
  });

  console.log("[MyApp] Actions registered");
};
```

### Handling Mapped Physical Buttons

When a user maps a physical button to an action, the ACTION event fires:

```javascript
DeskThing.on(DESKTHING_EVENTS.ACTION, (data) => {
  const actionId = data?.payload?.id || data?.id;
  const actionValue = data?.payload?.value;
  
  console.log(`[MyApp] ACTION: ${actionId}, value: ${actionValue}`);
  
  switch (actionId) {
    case "bandUp":
      changeBand(1);
      break;
    case "bandDown":
      changeBand(-1);
      break;
    case "setBand":
      if (actionValue && BAND_FREQUENCIES[actionValue]) {
        setFrequency(BAND_FREQUENCIES[actionValue]);
      }
      break;
    case "tuneUp":
      tune(state.tuningStep);
      break;
    case "tuneDown":
      tune(-state.tuningStep);
      break;
    default:
      console.log(`[MyApp] Unknown action: ${actionId}`);
  }
});
```

### Scroll Wheel Handler

```javascript
DeskThing.on("scroll", (data) => {
  const direction = data?.payload?.direction || 0;
  if (direction > 0) {
    // Scroll right/down
    tune(state.tuningStep);
  } else if (direction < 0) {
    // Scroll left/up
    tune(-state.tuningStep);
  }
});
```

### Key/Button Press Handler

```javascript
DeskThing.on("button", (data) => {
  const button = data?.payload?.button;
  const state = data?.payload?.state; // "down" or "up"
  
  if (state === "down") {
    switch (button) {
      case "m1": handleButton1(); break;
      case "m2": handleButton2(); break;
      case "m3": handleButton3(); break;
      case "m4": handleButton4(); break;
    }
  }
});
```

---

## 8. Key Registration - CRITICAL FOR SCROLL/BUTTONS

### Understanding Keys vs Actions

**Actions** are what users can map to buttons - they define WHAT happens.  
**Keys** tell DeskThing which events should be routed to your app when it's focused.

Without proper key registration, scroll wheel and button events may not reach your app when it's the active/focused application.

### ⚠️ CRITICAL: The modes Property Bug

**DO NOT add `modes: ["default"]` to key registration!**

The string `"default"` is NOT a valid EventMode value. This will cause your key registration to fail silently, and scroll/button events will NOT reach your app when it's in focus.

```javascript
// ❌ WRONG - Will cause scroll to stop working when app is focused
const keys = [
  {
    id: "scroll",
    description: "Wheel scroll for tuning",
    modes: ["default"]  // ← THIS BREAKS EVERYTHING
  }
];

// ✅ CORRECT - Simple registration works
const keys = [
  {
    id: "scroll",
    description: "Wheel scroll for tuning"
  }
];
```

### Valid EventMode Values

If you need to specify modes, use these integer values or enum names:

| Value | Name | Description |
|-------|------|-------------|
| 0 | KeyUp | Key/button released |
| 1 | KeyDown | Key/button pressed |
| 2 | ScrollUp | Scroll wheel up |
| 3 | ScrollDown | Scroll wheel down |
| 4 | ScrollLeft | Scroll wheel left (Car Thing maps this) |
| 5 | ScrollRight | Scroll wheel right (Car Thing maps this) |
| 6 | SwipeUp | Touch swipe up |
| 7 | SwipeDown | Touch swipe down |
| 8 | SwipeLeft | Touch swipe left |
| 9 | SwipeRight | Touch swipe right |
| 10 | PressShort | Short press |
| 11 | PressLong | Long press |

### Registering Keys Correctly

```javascript
var initializeActions = async () => {
  // First register actions...
  const actions = [
    { id: "tuneUp", name: "Tune Up", description: "Increase frequency", tag: "basic", version: "1.0.0", version_code: 1, enabled: true },
    { id: "tuneDown", name: "Tune Down", description: "Decrease frequency", tag: "basic", version: "1.0.0", version_code: 1, enabled: true },
  ];
  
  for (const action of actions) {
    await DeskThing.registerAction(action);
  }
  
  // Then register keys (for physical input routing)
  const keys = [
    {
      id: "scroll",
      description: "Wheel scroll for tuning"
      // NO modes property - let DeskThing handle it
    },
    {
      id: "dialPress", 
      description: "Dial/wheel press"
      // NO modes property
    }
  ];
  
  for (const key of keys) {
    try {
      await DeskThing.registerKey(key);
    } catch (e) {
      // Log but don't crash - key registration can fail silently
      console.log(`[MyApp] Key registration: ${e.message}`);
    }
  }
  
  console.log("[MyApp] Actions and keys registered");
};
```

### How Button Mapping Works with Keys

When a user creates a mapping like:
```json
"Scroll": {
  "4": { "id": "tuneDown", "source": "my-app" },
  "5": { "id": "tuneUp", "source": "my-app" }
}
```

DeskThing needs to know:
1. Your app registered actions `tuneDown` and `tuneUp` ✓
2. Your app registered a key for `scroll` events ✓
3. When your app is focused, route scroll events to it ✓

If key registration fails (e.g., due to invalid `modes`), step 3 breaks - events won't reach your app when focused.

### Symptoms of Failed Key Registration

- Scroll/buttons work when app drawer is visible
- Scroll/buttons work when OTHER apps are focused
- Scroll/buttons DON'T work when YOUR app is focused
- No errors in console (registration fails silently)

### Debugging Key Registration

Add logging to catch silent failures:

```javascript
for (const key of keys) {
  try {
    await DeskThing.registerKey(key);
    console.log(`[MyApp] Key "${key.id}" registered successfully`);
  } catch (e) {
    console.error(`[MyApp] Key "${key.id}" registration FAILED: ${e.message}`);
  }
}
```

---

## 9. Settings System

### Defining Settings

```javascript
var initializeSettings = async () => {
  const settings = {
    // String setting
    'device_ip': {
      id: 'device_ip',
      type: SETTING_TYPES.STRING,
      value: '',
      label: 'Device IP Address',
      description: 'Leave blank for auto-discovery'
    },
    
    // Number setting
    'refresh_rate': {
      id: 'refresh_rate',
      type: SETTING_TYPES.NUMBER,
      value: 30000,
      label: 'Refresh Rate (ms)',
      min: 5000,
      max: 300000,
      step: 5000,
      description: 'How often to fetch new data'
    },
    
    // Boolean setting
    'auto_connect': {
      id: 'auto_connect',
      type: SETTING_TYPES.BOOLEAN,
      value: true,
      label: 'Auto Connect',
      description: 'Automatically connect on startup'
    },
    
    // Select setting
    'default_mode': {
      id: 'default_mode',
      type: SETTING_TYPES.SELECT,
      value: 'USB',
      label: 'Default Mode',
      description: 'Mode to use on startup',
      options: [
        { value: 'USB', label: 'USB' },
        { value: 'LSB', label: 'LSB' },
        { value: 'CW', label: 'CW' },
        { value: 'AM', label: 'AM' }
      ]
    }
  };
  
  await DeskThing.initSettings(settings);
  console.log("[MyApp] Settings initialized");
};
```

### Reading Settings

```javascript
// In your registerAllHandlers or after initialization:
DeskThing.on(DESKTHING_EVENTS.SETTINGS, (data) => {
  const settings = data?.payload;
  if (settings) {
    if (settings.device_ip?.value) {
      deviceIp = settings.device_ip.value;
    }
    if (settings.refresh_rate?.value) {
      refreshInterval = settings.refresh_rate.value;
    }
  }
});
```

---

## 10. State Management

### State Object Pattern

```javascript
var state = {
  // Connection state
  connected: false,
  deviceModel: "",
  
  // Main data
  frequency: 14074000,
  mode: "USB",
  
  // UI state
  currentScreen: "vfo",
  tuningStep: 100,
  tuningStepIndex: 2,
  
  // Feature states
  txActive: false,
  nbEnabled: false,
  nrEnabled: false,
  
  // Meter values
  sMeter: -100,
  powerMeter: 0,
  swrMeter: 1.0
};
```

### Sending State to Client

```javascript
var sendStateToClient = () => {
  DeskThing.send({ 
    type: "appState", 
    payload: state 
  });
};

// Call after any state change:
state.frequency = newFrequency;
sendStateToClient();
```

### Sending Specific Data

```javascript
// Screen change
DeskThing.send({ 
  type: "screenChange", 
  payload: { screen: "dsp" } 
});

// Spots/list data
DeskThing.send({ 
  type: "potaSpots", 
  payload: spotsArray 
});

// Meter update (frequent updates)
DeskThing.send({ 
  type: "meterUpdate", 
  payload: { sMeter: -73.2, power: 50 } 
});
```

---

## 11. Network Communication Patterns

### TCP Client (for devices like radios, controllers)

```javascript
const net = require("net");

var socket = null;
var commandCounter = 1;

var connect = (host, port) => {
  return new Promise((resolve, reject) => {
    socket = new net.Socket();
    
    socket.connect(port, host, () => {
      console.log(`[MyApp] Connected to ${host}:${port}`);
      state.connected = true;
      sendStateToClient();
      resolve();
    });
    
    socket.on("data", (data) => {
      const lines = data.toString().split("\n");
      lines.forEach(line => {
        if (line.trim()) {
          processResponse(line.trim());
        }
      });
    });
    
    socket.on("error", (err) => {
      console.error("[MyApp] Socket error:", err.message);
      state.connected = false;
      sendStateToClient();
      reject(err);
    });
    
    socket.on("close", () => {
      console.log("[MyApp] Disconnected");
      state.connected = false;
      sendStateToClient();
    });
  });
};

var sendCommand = (cmd) => {
  if (socket && state.connected) {
    const fullCmd = `C${commandCounter++}|${cmd}`;
    console.log(`[MyApp] TX: ${fullCmd}`);
    socket.write(fullCmd + "\n");
  }
};

var disconnect = () => {
  if (socket) {
    socket.destroy();
    socket = null;
  }
};
```

### UDP Discovery

```javascript
const dgram = require("dgram");

var discoverySocket = null;

var startDiscovery = () => {
  discoverySocket = dgram.createSocket({ type: "udp4", reuseAddr: true });
  
  discoverySocket.on("message", (msg, rinfo) => {
    const message = msg.toString();
    console.log(`[MyApp] Discovery from ${rinfo.address}: ${message}`);
    
    // Parse discovery response
    if (message.includes("device_type=mydevice")) {
      const ip = rinfo.address;
      console.log(`[MyApp] Found device at ${ip}`);
      discoverySocket.close();
      connect(ip, 4992);
    }
  });
  
  discoverySocket.on("error", (err) => {
    console.error("[MyApp] Discovery error:", err.message);
  });
  
  discoverySocket.bind(4992, "0.0.0.0", () => {
    discoverySocket.setBroadcast(true);
    console.log("[MyApp] Listening for discovery...");
  });
};
```

### HTTP/REST API

```javascript
var fetchData = async () => {
  try {
    console.log("[MyApp] Fetching data...");
    const response = await fetch("https://api.example.com/data");
    
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    
    const data = await response.json();
    console.log(`[MyApp] Got ${data.length} items`);
    
    DeskThing.send({ type: "data", payload: data });
  } catch (error) {
    console.error("[MyApp] Fetch error:", error.message);
  }
};

// Sync wrapper for intervals
var doFetchData = () => { fetchData(); };
```

---

## 12. Real-Time Data Streaming

### UDP Meter/Data Streaming

```javascript
const dgram = require("dgram");

var meterSocket = null;
var METER_UDP_PORT = 54993;

var startMeterListener = () => {
  meterSocket = dgram.createSocket({ type: "udp4", reuseAddr: true });
  
  meterSocket.on("message", (data, rinfo) => {
    try {
      const parsed = parseMeterPacket(data);
      if (parsed) {
        state.sMeter = parsed.sMeter;
        // Don't sendStateToClient() on every packet - too frequent
        // Instead, use a throttled update
      }
    } catch (err) {
      console.error("[MyApp] Meter parse error:", err.message);
    }
  });
  
  meterSocket.on("error", (err) => {
    console.error("[MyApp] Meter socket error:", err.message);
  });
  
  meterSocket.bind(METER_UDP_PORT, "0.0.0.0", () => {
    console.log(`[MyApp] Meter listener on port ${METER_UDP_PORT}`);
  });
};

var parseMeterPacket = (data) => {
  // Example: Parse VITA-49 style packet
  if (data.length < 16) return null;
  
  const meterId = data.readUInt16BE(0);
  const rawValue = data.readInt16BE(2);  // SIGNED 16-bit
  const scaledValue = rawValue / 128.0;
  
  return { meterId, value: scaledValue };
};

// Throttled state update (every 3 seconds)
var lastMeterUpdate = 0;
var updateMeterThrottled = (sMeter) => {
  const now = Date.now();
  if (now - lastMeterUpdate > 3000) {
    lastMeterUpdate = now;
    state.sMeter = sMeter;
    sendStateToClient();
  }
};
```

### Periodic State Updates

```javascript
var stateInterval = null;

var startStateUpdates = () => {
  stateInterval = setInterval(() => {
    sendStateToClient();
  }, 1000);  // Update client every second
};

var stopStateUpdates = () => {
  if (stateInterval) {
    clearInterval(stateInterval);
    stateInterval = null;
  }
};
```

---

## 13. Multi-Screen Applications

### Server-Side Screen Management

```javascript
var currentScreen = "vfo";

var switchScreen = (screen) => {
  currentScreen = screen;
  console.log(`[MyApp] Switch to ${screen} screen`);
  DeskThing.send({ 
    type: "screenChange", 
    payload: { screen } 
  });
};

// In registerAllHandlers:
DeskThing.on("screenVFO", () => switchScreen("vfo"));
DeskThing.on("screenDSP", () => switchScreen("dsp"));
DeskThing.on("screenMemory", () => switchScreen("memory"));
DeskThing.on("screenTX", () => switchScreen("tx"));
```

### Client-Side Screen Rendering

```typescript
const [currentScreen, setCurrentScreen] = useState("vfo");

useEffect(() => {
  const offScreen = deskthing.on("screenChange", (data) => {
    if (data?.payload?.screen) {
      setCurrentScreen(data.payload.screen);
    }
  });
  return () => offScreen();
}, []);

const renderScreen = () => {
  switch (currentScreen) {
    case "vfo":
      return <VFOScreen 
        frequency={state.frequency}
        mode={state.mode}
        sMeter={state.sMeter}
        onBandUp={() => handleAction("bandUp")}
        onBandDown={() => handleAction("bandDown")}
      />;
    case "dsp":
      return <DSPScreen
        nbEnabled={state.nbEnabled}
        nrEnabled={state.nrEnabled}
        onToggleNB={() => handleAction("toggleNB")}
        onToggleNR={() => handleAction("toggleNR")}
      />;
    case "memory":
      return <MemoryScreen
        memories={memories}
        onRecall={(slot) => handleAction(`memory${slot}`)}
      />;
    default:
      return <VFOScreen {...props} />;
  }
};
```

### Navigation Buttons

```typescript
// Bottom navigation bar component
const NavBar = ({ currentScreen, onNavigate }) => (
  <div className="nav-bar">
    <button 
      className={currentScreen === "vfo" ? "active" : ""}
      onClick={() => onNavigate("screenVFO")}
    >
      VFO
    </button>
    <button 
      className={currentScreen === "dsp" ? "active" : ""}
      onClick={() => onNavigate("screenDSP")}
    >
      DSP
    </button>
    <button 
      className={currentScreen === "memory" ? "active" : ""}
      onClick={() => onNavigate("screenMemory")}
    >
      MEM
    </button>
  </div>
);
```

---

## 14. Complete Server Template

```javascript
import { DeskThing } from '@deskthing/server';

// ============================================
// CONSTANTS
// ============================================
const REFRESH_INTERVAL = 60000;

// ============================================
// STATE
// ============================================
var state = {
  connected: false,
  data: [],
  lastUpdate: null
};

// ============================================
// MODULE VARIABLES
// ============================================
var refreshInterval = null;
var handlersRegistered = false;

// ============================================
// HELPER FUNCTIONS
// ============================================
var sendStateToClient = () => {
  DeskThing.send({ type: "appState", payload: state });
};

// ============================================
// SETTINGS
// ============================================
var initializeSettings = async () => {
  const settings = {
    'refresh_rate': {
      id: 'refresh_rate',
      type: 'number',
      value: REFRESH_INTERVAL,
      label: 'Refresh Rate (ms)',
      min: 10000,
      max: 300000,
      step: 5000,
      description: 'How often to refresh data'
    }
  };
  await DeskThing.initSettings(settings);
};

// ============================================
// ACTIONS
// ============================================
var initializeActions = async () => {
  DeskThing.registerAction({
    id: "refresh",
    name: "Refresh Data",
    description: "Manually refresh data",
    tag: "data",
    version: "1.0.0",
    version_code: 1,
    enabled: true
  });
  
  console.log("[MyApp] Actions registered");
};

// ============================================
// DATA FETCHING
// ============================================
var doFetch = async () => {
  try {
    console.log("[MyApp] Fetching data...");
    const response = await fetch("https://api.example.com/data");
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    state.data = await response.json();
    state.lastUpdate = new Date().toISOString();
    console.log(`[MyApp] Got ${state.data.length} items`);
    sendStateToClient();
  } catch (error) {
    console.error("[MyApp] Fetch error:", error.message);
  }
};

var fetchData = () => { doFetch(); };

// ============================================
// INITIALIZATION
// ============================================
var doInitialize = async () => {
  fetchData();
  refreshInterval = setInterval(fetchData, REFRESH_INTERVAL);
};

// ============================================
// START
// ============================================
var start = async () => {
  console.log("[MyApp] Starting...");
  await initializeSettings();
  await initializeActions();
  sendStateToClient();
  
  if (!handlersRegistered) {
    handlersRegistered = true;
    console.log("[MyApp] Registering handlers...");
    registerAllHandlers();
  }
  
  await doInitialize();
};

// ============================================
// HANDLER REGISTRATION
// ============================================
var registerAllHandlers = () => {
  // State request
  DeskThing.on("getState", () => {
    sendStateToClient();
  });
  
  // Manual refresh
  DeskThing.on("refresh", () => {
    fetchData();
  });
  
  // ACTION handler for physical buttons
  DeskThing.on("action", (data) => {
    const actionId = data?.payload?.id || data?.id;
    switch (actionId) {
      case "refresh":
        fetchData();
        break;
      default:
        console.log(`[MyApp] Unknown action: ${actionId}`);
    }
  });
};

// ============================================
// STOP
// ============================================
var stop = async () => {
  console.log("[MyApp] Stopping...");
  if (refreshInterval) {
    clearInterval(refreshInterval);
    refreshInterval = null;
  }
};

// ============================================
// EVENT REGISTRATION
// ============================================
DeskThing.on("start", start);
DeskThing.on("stop", stop);
```

---

## 15. Complete Client Template

```typescript
import { useEffect, useState, useCallback } from 'react';
import { DeskThing } from '@deskthing/client';

const deskthing = DeskThing;

interface AppState {
  connected: boolean;
  data: any[];
  lastUpdate: string | null;
}

function App() {
  const [state, setState] = useState<AppState>({
    connected: false,
    data: [],
    lastUpdate: null
  });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Listen for state updates
    const offState = deskthing.on('appState', (event: any) => {
      if (event?.payload) {
        setState(event.payload);
        setLoading(false);
      }
    });

    // Request initial state
    setTimeout(() => {
      deskthing.send({ type: 'getState' });
    }, 500);

    return () => {
      offState();
    };
  }, []);

  const handleAction = useCallback((action: string, payload?: any) => {
    deskthing.send({ type: action, payload });
  }, []);

  if (loading) {
    return (
      <div className="h-screen w-screen bg-gray-900 text-white flex items-center justify-center">
        <div className="text-2xl">Loading...</div>
      </div>
    );
  }

  return (
    <div className="h-screen w-screen bg-gray-900 text-white flex flex-col">
      {/* Header */}
      <div className="p-4 bg-gray-800 flex justify-between items-center">
        <h1 className="text-xl font-bold">My App</h1>
        <span className="text-sm text-gray-400">
          {state.lastUpdate ? `Updated: ${state.lastUpdate}` : ''}
        </span>
      </div>
      
      {/* Content */}
      <div className="flex-1 overflow-y-auto p-4">
        {state.data.map((item, index) => (
          <div key={index} className="p-3 mb-2 bg-gray-800 rounded">
            {/* Render item */}
          </div>
        ))}
      </div>
      
      {/* Footer/Actions */}
      <div className="p-4 bg-gray-800 flex justify-center gap-4">
        <button 
          onClick={() => handleAction('refresh')}
          className="px-6 py-2 bg-blue-600 rounded"
        >
          Refresh
        </button>
      </div>
    </div>
  );
}

export default App;
```

---

## 16. Debugging Guide

### Essential Logging

```javascript
// Always prefix logs with app name for filtering
console.log("[MyApp] Starting...");
console.log("[MyApp] Connecting to device...");
console.error("[MyApp] Connection failed:", error.message);

// Log all commands sent
var sendCommand = (cmd) => {
  console.log(`[MyApp] TX: ${cmd}`);
  socket.write(cmd + "\n");
};

// Log important state changes
state.frequency = newFreq;
console.log(`[MyApp] Frequency: ${newFreq} Hz`);
```

### Debug Handler Registration

```javascript
if (!handlersRegistered) {
  handlersRegistered = true;
  console.log("[MyApp] Registering handlers (first time)...");
  registerAllHandlers();
} else {
  console.log("[MyApp] Handlers already registered, skipping...");
}
```

### Debug Message Flow

```javascript
// In registerAllHandlers:
DeskThing.on("bandUp", () => {
  console.log("[MyApp] bandUp handler fired");
  changeBand(1);
});

DeskThing.on("action", (data) => {
  console.log("[MyApp] ACTION received:", JSON.stringify(data));
  // ...
});
```

### Tracking Double-Fire Issues

```javascript
// Add counters to suspect handlers
var bandUpCount = 0;
DeskThing.on("bandUp", () => {
  bandUpCount++;
  console.log(`[MyApp] bandUp fired (count: ${bandUpCount})`);
  changeBand(1);
});
```

---

## 17. Common Bugs and Solutions

### Bug: Actions Fire Multiple Times

**Symptom:** Pressing a button causes the action to happen 2x, 3x, or more times.

**Cause:** Handlers registered inside `start()` accumulate on restarts.

**Solution:**
```javascript
var handlersRegistered = false;

var start = async () => {
  if (!handlersRegistered) {
    handlersRegistered = true;
    registerAllHandlers();
  }
  // ...
};
```

### Bug: App Crashes on Restart

**Symptom:** App works first time, crashes on restart with socket/port errors.

**Cause:** Resources not cleaned up in `stop()`.

**Solution:**
```javascript
var stop = async () => {
  if (pingInterval) clearInterval(pingInterval);
  if (stateInterval) clearInterval(stateInterval);
  if (socket) socket.destroy();
  if (udpSocket) udpSocket.close();
};
```

### Bug: Settings Don't Load

**Symptom:** Settings are always default values.

**Cause:** `initializeSettings()` not called first, or SETTINGS handler not registered.

**Solution:**
```javascript
var start = async () => {
  await initializeSettings();  // MUST be first
  // ...
};

// In registerAllHandlers:
DeskThing.on("settings", (data) => {
  if (data?.payload) {
    // Apply settings
  }
});
```

### Bug: Physical Buttons Don't Work

**Symptom:** On-screen buttons work but mapped physical buttons don't.

**Cause:** Missing ACTION handler.

**Solution:**
```javascript
DeskThing.on("action", (data) => {
  const actionId = data?.payload?.id || data?.id;
  switch (actionId) {
    case "myAction": doSomething(); break;
  }
});
```

### Bug: On-Screen Buttons Don't Work

**Symptom:** Physical buttons work but on-screen buttons don't.

**Cause:** Missing direct message handler.

**Solution:**
```javascript
DeskThing.on("myAction", () => {
  doSomething();
});
```

### Bug: Client Shows "Loading" Forever

**Symptom:** Client never receives state.

**Cause:** Server not sending state, or wrong message type.

**Solution:**
```javascript
// Server sends:
DeskThing.send({ type: "appState", payload: state });

// Client listens for same type:
deskthing.on("appState", (data) => {
  setState(data.payload);
  setLoading(false);
});
```

### Bug: UDP Port Already in Use

**Symptom:** "EADDRINUSE" error on restart.

**Cause:** Socket not properly closed.

**Solution:**
```javascript
var stop = async () => {
  if (udpSocket) {
    try {
      udpSocket.close();
    } catch (e) {
      // Ignore if already closed
    }
    udpSocket = null;
  }
};

// Use reuseAddr when creating:
dgram.createSocket({ type: "udp4", reuseAddr: true });
```

### Bug: Scroll/Buttons Only Work When App Drawer Open

**Symptom:** Scroll wheel or mapped buttons work when the app drawer is visible, or when other apps are focused, but NOT when your app is the focused/active app.

**Cause:** Invalid `modes` property in key registration causing silent failure.

**Solution:**
```javascript
// ❌ WRONG - "default" is not a valid EventMode
const keys = [
  { id: "scroll", description: "...", modes: ["default"] }
];

// ✅ CORRECT - Don't include modes at all
const keys = [
  { id: "scroll", description: "Wheel scroll for tuning" }
];
```

**Why This Happens:**
- The `modes` property expects EventMode integer values (0-11), not strings
- `"default"` fails validation silently
- DeskThing doesn't know to route events to your app when focused
- Events still work globally (drawer open) but not when app-specific routing is needed

### Bug: Key Registration Fails Silently

**Symptom:** No errors in console, but physical inputs don't work for your app.

**Cause:** Key registration wrapped in try/catch swallows errors.

**Solution:** Add explicit logging to catch failures:
```javascript
for (const key of keys) {
  try {
    await DeskThing.registerKey(key);
    console.log(`[MyApp] ✓ Key "${key.id}" registered`);
  } catch (e) {
    console.error(`[MyApp] ✗ Key "${key.id}" FAILED: ${e.message}`);
    // Now you'll see what went wrong!
  }
}
```

---

## 18. Build and Deployment

### Package.json (Development)

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build && npm run build:server",
    "build:server": "esbuild server/index.ts --bundle --platform=node --outfile=dist/server/index.js --format=esm --external:@deskthing/server"
  },
  "dependencies": {
    "@deskthing/client": "^0.11.2",
    "@deskthing/server": "^0.11.6",
    "@deskthing/types": "^0.11.6",
    "react": "^19.1.0",
    "react-dom": "^19.1.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.5.0",
    "@vitejs/plugin-legacy": "^5.4.3",
    "typescript": "^5.8.3",
    "vite": "^6.4.1"
  }
}
```

### Vite Config

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import legacy from '@vitejs/plugin-legacy';

export default defineConfig({
  plugins: [
    react(),
    legacy({
      targets: ['chrome >= 69'],
      additionalLegacyPolyfills: ['regenerator-runtime/runtime']
    })
  ],
  base: './'
});
```

### Build Process

1. `npm install`
2. `npm run build`
3. Create ZIP with structure:
   ```
   my-app.zip
   ├── manifest.json
   ├── package.json          # {"type": "module"}
   ├── icons/my-app.svg
   ├── server/
   │   ├── index.js
   │   └── package.json      # {"type": "module"}
   └── client/
       ├── index.html
       └── ...
   ```
4. Install via DeskThing UI

### Manifest

```json
{
  "label": "My App Name",
  "id": "my-app",
  "version": "1.0.0",
  "description": "What this app does",
  "author": "Your Name",
  "platforms": ["windows", "linux", "mac"],
  "requiredVersions": {
    "server": ">=0.11.0",
    "client": ">=0.11.2"
  },
  "template": "full",
  "tags": ["category"],
  "requires": [],
  "repository": "",
  "homepage": "",
  "updateUrl": ""
}
```

---

## 19. Quick Reference Cheat Sheet

### Server Message Sending
```javascript
DeskThing.send({ type: "appState", payload: state });
DeskThing.send({ type: "screenChange", payload: { screen: "dsp" } });
DeskThing.send({ type: "dataList", payload: dataArray });
```

### Server Event Handling
```javascript
DeskThing.on("getState", () => { sendStateToClient(); });
DeskThing.on("myAction", () => { doAction(); });
DeskThing.on("action", (data) => { handleAction(data); });
DeskThing.on("scroll", (data) => { handleScroll(data); });
DeskThing.on("button", (data) => { handleButton(data); });
DeskThing.on("settings", (data) => { applySettings(data); });
```

### Client Message Sending
```typescript
deskthing.send({ type: "getState" });
deskthing.send({ type: "myAction" });
deskthing.send({ type: "setData", payload: { value: 123 } });
```

### Client Event Handling
```typescript
deskthing.on("appState", (data) => { setState(data.payload); });
deskthing.on("screenChange", (data) => { setScreen(data.payload.screen); });
deskthing.on("dataList", (data) => { setData(data.payload); });
```

### File Structure Checklist
- [ ] `manifest.json` with correct `id`
- [ ] `icons/{id}.svg`
- [ ] `server/index.js` with handlers
- [ ] `server/package.json` with `{"type": "module"}`
- [ ] `client/index.html`
- [ ] `package.json` with `{"type": "module"}`

### Handler Registration Checklist
- [ ] `handlersRegistered` flag defined
- [ ] Guard in `start()` to prevent duplicate registration
- [ ] All handlers in `registerAllHandlers()` function
- [ ] Only `START` and `STOP` at module level
- [ ] ACTION handler for physical buttons
- [ ] Direct handlers for on-screen buttons

### Key Registration Checklist
- [ ] Keys registered with `id` and `description` only
- [ ] NO `modes: ["default"]` - this breaks everything!
- [ ] If using modes, use integers 0-11 (EventMode values)
- [ ] Wrap in try/catch with explicit logging
- [ ] Test scroll/buttons WHEN APP IS FOCUSED (not just drawer open)

### Stop Function Checklist
- [ ] Clear all intervals
- [ ] Close all sockets
- [ ] Destroy all connections
- [ ] Reset state variables if needed

---

## Conclusion

This guide represents hundreds of hours of debugging and development experience with DeskThing. The patterns here are battle-tested and proven to work. Follow them exactly, and your app will work correctly the first time.

**Key Takeaways:**
1. Handler registration MUST happen only once
2. Use the guard pattern for handlers
3. Understand the difference between on-screen and physical button events
4. Always clean up resources in stop()
5. Use proper async patterns for network calls
6. Log everything during development
7. Test restarts - that's where most bugs hide
8. **NEVER use `modes: ["default"]` in key registration - it will silently break scroll/button handling!**
9. Test physical inputs when your app is FOCUSED, not just when drawer is open

Now you're ready to build any DeskThing app. What do you want to make?  Also, you should subscribe to DudeTested on youtube, who shared this guide! 
