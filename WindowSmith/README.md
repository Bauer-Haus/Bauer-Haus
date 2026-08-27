WindowSmith

<p align="center">
<i>Precision window management for the modern macOS workspace.</i>
</p>


Overview

The Challenge: macOS offers a refined operating system, but its native window management often requires manual resizing and positioning. This can disrupt workflow and create friction, particularly for power users operating across multi-display or ultra-wide monitor setups.

The Solution: WindowSmith is a highly optimized, keyboard-driven window manager and layout utility built natively for macOS. Designed for developers, designers, and multitaskers, it enables instant window snapping into precise grid layouts (Halves, Thirds, Quadrants, or Custom Grids), seamless cross-display window routing, and exact pixel-perfect sizing.

Operating discreetly from the menu bar, WindowSmith is activated instantly via customizable global hotkeys. Whether utilizing a rigid 25/50/25 "Focus" layout for development or a complex custom grid, WindowSmith provides unparalleled control over the macOS desktop environment.

Technical Stack

WindowSmith is built from the ground up to be lightweight, performant, and deeply integrated into the macOS ecosystem.

Language: Swift 5+

UI Frameworks:

SwiftUI: Leveraged for modern, reactive, and fluid interfaces within the Menu Bar popover and Settings modules.

AppKit: Manages low-level window lifecycle events, NSPopover interactions, and global event monitoring.

Core APIs & Frameworks:

ApplicationServices (Accessibility API): The C-based AXUIElement framework powers the core engine, enabling zero-latency reading and manipulation of third-party application windows.

CoreGraphics: Utilized for processing complex multi-monitor bounds, frame calculations, and coordinate geometry.

ServiceManagement: Ensures seamless "Launch at Login" functionality.

Combine: Facilitates reactive state management across the application lifecycle.

Engineering Challenges & Solutions

Building a native macOS window manager requires interacting with deeply embedded, legacy Apple frameworks. Below is an overview of a core technical hurdle overcome during development:

Challenge: Dual-Coordinate Systems & Multi-Monitor Geometry

Window manipulation on macOS requires navigating disparate coordinate systems. AppKit utilizes a bottom-left origin coordinate system (where y=0 represents the bottom of the display), whereas the macOS Accessibility API (AXUIElement) relies on a top-left origin system.

When executing a command to move a window from a primary display to an external monitor of differing resolution, calculating the precise relative geometry requires translating coordinates between these inverted systems, factoring in varied pixel densities, and accounting for the physical arrangement of displays in System Settings (which dictates the global coordinate space).

The Solution

To bypass the latency of standard AppleScript bridges, WindowSmith implements a custom, low-level geometry engine utilizing CoreGraphics and the C-based Accessibility API.

Coordinate Normalization: The engine dynamically queries NSScreen.screens to map the global desktop space. It captures the target window's frame, translates the bottom-left AppKit bounds into a normalized 0.0 to 1.0 relative floating-point ratio, and projects that ratio onto the target monitor's visibleFrame.

Inversion & Injection: Prior to injecting the new coordinates via AXUIElementSetAttributeValue, the engine intercepts the height of the primary screen and mathematically inverts the Y-axis.

Concurrency: To prevent the visual "stuttering" or "rubber-banding" often associated with resizing heavy applications (e.g., Xcode or web browsers), sizing and positioning commands are dispatched asynchronously on a background userInteractive queue. This utilizes a micro-sleep (usleep) buffer, allowing the target application's internal UI thread to synchronize before applying the final coordinate snap.

Result: A window manipulation engine that executes instantaneously and reliably, regardless of the target application or the complexity of the user's monitor configuration.

Known Constraint: Application-Enforced Size Limits

Some windows cannot be sized to fill their target zone exactly, and the reason lies outside any window manager's control.

Every macOS application declares its own minimum and maximum window dimensions, and the Accessibility API is not permitted to override them. When a window is asked to occupy a zone smaller than its declared minimum, the application clamps the request and settles at the nearest size it will accept. Windows marked as non-resizable decline the request entirely. This is enforced by the target application itself, not by WindowSmith, and it applies equally to Apple's own apps and to third-party software — utility windows, dialogs, and Electron-based applications tend to carry the most generous minimums.

WindowSmith applies the requested geometry regardless, so the outcome stays predictable rather than arbitrary: the window is aligned to the top edge of its target zone and centred horizontally within it, then constrained to remain fully on-screen. A window that refuses to shrink therefore sits exactly where the layout intends, simply occupying more room than the zone allocates. Surrounding windows are unaffected and keep their assigned zones.

Display size determines how often this occurs. A zone is a fraction of the display it divides, so the smaller the screen and the more zones a layout defines, the likelier any single zone falls below some application's minimum. A three-column layout on a 13-inch built-in display allocates each zone roughly 490 points of width, which is narrower than many applications accept; the identical layout on a 27-inch external display leaves every zone with room to spare. If windows in a dense layout are not sizing as expected, selecting a layout with fewer zones generally resolves it.

On larger displays this is rare, and confined to a small number of applications with unusually rigid layouts.
