# WindowSmith Architecture

Reflects WindowSmith 1.1 (build 11).

```mermaid
graph TD
    %% Define Styles
    classDef ui fill:#0f4c75,stroke:#3282b8,stroke-width:2px,color:#fff,rx:5px,ry:5px;
    classDef core fill:#1b262c,stroke:#bbe1fa,stroke-width:2px,color:#fff,rx:5px,ry:5px;
    classDef sys fill:#3282b8,stroke:#0f4c75,stroke-width:2px,color:#fff,rx:5px,ry:5px;
    classDef net fill:#16213e,stroke:#9d8cff,stroke-width:2px,color:#fff,rx:5px,ry:5px;
    classDef target fill:#1a1a2e,stroke:#e94560,stroke-width:3px,color:#fff,rx:10px,ry:10px;

    subgraph UI ["User Interface (SwiftUI)"]
        App["WindowSmithApp<br/>@main · MenuBarExtra(.window)"]
        MenuBarView["MenuBarView"]
        Overlay["PermissionOverlay"]
        SWM["SettingsWindowManager<br/>NSWindow + NSHostingController"]
        SettingsView["SettingsView"]
        Snap["SnapLayoutView"]
        Cell["PresetCell"]
        Grid["InteractiveGridSelector"]
        Preview["SnapPreviewOverlay<br/>borderless · click-through NSWindow"]

        App -->|hosts| MenuBarView
        MenuBarView -->|when untrusted| Overlay
        MenuBarView -->|renders| Snap
        MenuBarView -->|renders| Cell
        MenuBarView -->|opens| SWM
        SWM -->|hosts| SettingsView
        SettingsView -->|renders| Grid
    end

    subgraph Core ["Core Engine (Swift)"]
        WC["WindowController<br/>singleton · ObservableObject"]
        GKM["GlobalKeyMonitor<br/>local + global keyDown"]
        UPD["UpdaterController<br/>wraps SPUStandardUpdaterController"]

        subgraph WC_Logic ["WindowController Internals"]
            Geo["Geometry Engine<br/>relative rects · grid math · throw"]
            Perm["Permission Watch<br/>trust notification + 1s poll"]
            Cycle["Cycle State<br/>5s arrow-key timeout"]
        end

        DSM["DragSnapMonitor<br/>global mouse monitors · drag detection"]
        DSS["DragSnapSettings<br/>layout · trigger · enabled"]
        WL["WindowLookup<br/>window at point · coordinate flips"]

        UD[("UserDefaults<br/>presets · hotkeys · drag-snap · prompt flag")]
    end

    subgraph MacOS ["macOS System APIs"]
        AX["Accessibility API (AXUIElement)<br/>answered by the owning app"]
        CGW["CGWindowList<br/>answered by the WindowServer"]
        WS["NSWorkspace<br/>frontmost app tracking"]
        CG["NSScreen / CoreGraphics<br/>display bounds"]
        GCD["Grand Central Dispatch<br/>userInteractive queue"]
        SM["SMAppService<br/>Launch at Login"]
        DNC["DistributedNotificationCenter<br/>com.apple.accessibility.api"]
        OSL["OSLog"]
    end

    subgraph Updates ["Update Channel"]
        SPK["Sparkle 2<br/>embedded framework"]
        Feed["appcast.xml<br/>EdDSA-signed feed"]
    end

    subgraph Target ["Target Environment"]
        TA["Third-Party OS Window<br/>(Safari, Xcode, etc.)"]
    end

    %% UI to Core
    MenuBarView -->|Triggers Snap / Layout| WC
    SettingsView -->|Saves Presets & Hotkeys| WC
    Overlay -->|Grant Access| Perm
    MenuBarView -->|Check for Updates| UPD
    WC <-->|Combine @Published Updates| MenuBarView
    WC <-->|Persists / Loads| UD
    WC -->|Toggles| SM

    %% Core internals
    GKM == "Intercepts Ctrl+Opt+1-4 / Arrows" ==> WC
    WC --> Geo
    WC --> Cycle
    WC --> Perm
    WC --> OSL
    DNC -.->|AX trust changed| Perm
    Perm -.->|Re-arms monitors on grant| GKM

    %% Core to OS
    WC -->|Queries Frontmost PID| WS
    WC -->|Queries visibleFrame| CG
    Geo == "Dispatches Async Resize/Move" ==> GCD
    GCD == "Injects Coordinates (w/ usleep buffer)" ==> AX
    AX == "Translates & Modifies Frame" ==> TA

    %% Drag to snap
    SettingsView -->|Configures| DSS
    DSS -.->|Arms / disarms| DSM
    DSM -->|Which window is moving| WL
    WL -->|Live frame during a drag| CGW
    WL -->|Element to move · fallback frame| AX
    DSM -->|Previews the target zone| Preview
    DSM == "Applies the drop to that exact window" ==> WC

    %% Updates
    UPD --> SPK
    SPK -.->|Polls & verifies signature| Feed

    %% Assign Classes
    class App,MenuBarView,Overlay,SWM,SettingsView,Snap,Cell,Grid,Preview ui;
    class WC,GKM,UPD,Geo,Perm,Cycle,UD,WC_Logic,DSM,DSS,WL core;
    class AX,WS,CG,GCD,SM,DNC,OSL,CGW sys;
    class SPK,Feed net;
    class TA target;
```

## Notable Mechanisms

**Primary-display coordinate anchoring.** AppKit measures from the bottom-left of the primary display; the Accessibility API measures from the top-left. Every conversion is anchored to the display whose origin is `(0,0)` rather than `NSScreen.main`, which follows the key window and therefore reports a different — and differently sized — screen whenever the target window sits on a secondary monitor.

**Permission detection without polling the user.** A menu bar app never receives `didBecomeActive` when the user flips the Accessibility switch in System Settings, so trust changes arrive two ways: an observer on the `com.apple.accessibility.api` distributed notification, plus a 1-second timer that runs only while untrusted and invalidates itself on grant. The timer is scheduled in `.common` run loop mode so it keeps firing while the menu bar popover is open.

**Re-arming the key monitors.** Global `NSEvent` monitors installed before Accessibility is granted never receive `keyDown` and do not begin working retroactively, so the grant transition tears them down and reinstalls them.

**Modifier normalization.** Arrow keys set `.function` and `.numericPad` alongside the real modifiers, and caps lock can be latched, so all three are stripped before matching the `Ctrl+Opt` chord. Digits are read from `charactersIgnoringModifiers`, since Option remaps the printable character.

**Two disagreeing answers to "where is this window".** A title-bar drag is carried out by the WindowServer, not by the application — which is why a hung app's window can still be dragged. The Accessibility API asks the *owning application* for its position, so during a drag it returns a stale value and then jumps, measured at roughly 105ms behind the cursor across repeated drags. `CGWindowList` asks the WindowServer instead and tracks the cursor essentially 1:1. Drag detection therefore reads the WindowServer, which also cannot be stalled by an unresponsive app the way synchronous Accessibility IPC into that app's run loop can. The two sources are checked for agreement at mouse-down, and detection falls back to Accessibility if they disagree, so a mismatch can never move the wrong window.

**Distinguishing a window drag from a drag inside a window.** Nothing announces that a window drag has begun, so a candidate window is captured on mouse-down and promoted only once its frame actually moves while its size holds steady — a changed size means a resize handle, not a title bar. Cursor distance cannot make this call: the reported position lags far behind the pointer before catching up, so "the cursor moved but the window has not" describes an ordinary drag just as well as a text selection. Only elapsed time separates them, and nothing is drawn before the gesture is confirmed, so waiting costs a few extra probes and no visible latency.

**A preview that cannot swallow its own gesture.** The zone preview is a borderless, transparent `NSWindow` above the normal window level. It sits directly under the cursor for the entire drag, so it sets `ignoresMouseEvents` — without that it would consume the very drag it exists to illustrate.

**Asynchronous injection.** Sizing and positioning are dispatched on a background `userInteractive` queue with a short `usleep` buffer between the size and position writes, allowing the target application's UI thread to settle before the final coordinate snap.
