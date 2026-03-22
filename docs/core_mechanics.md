# Core Framework Mechanics

This document breaks down the foundational mechanics of the Makepad framework. It explores how the engine interacts with host operating systems and hardware, and deconstructs the internal event loop that drives the UI.

---

## 1. OS and GPU Abstractions

### The "What" and "Why"
**What it is:** The platform abstraction layer (found in `/platform/src/os` and `/platform/src/cx.rs`) serves as the bridge between the high-level Makepad framework and the low-level operating system. It defines a unified, cross-platform interface for window creation, input gathering, and hardware-accelerated rendering.

**Why it exists:** Makepad is designed for extreme portability and maximum performance. Rather than relying on heavy middleware layers (like SDL or Winit for windowing, or WGPU for graphics), Makepad builds thin, direct wrappers around native APIs. This ensures minimal overhead, rapid compilation times, and allows the framework to exploit platform-specific optimizations while presenting a singular API to the widget layer.

### How Data Flows

#### The God Object: `Cx`
The beating heart of the Makepad abstraction is the Context struct (`Cx`). It acts as a massive, centralized state container passed mutably throughout the application. 
*   **Memory Pooling:** To guarantee high performance and avoid heap fragmentation during the render loop, `Cx` utilizes pre-allocated pools for almost all resources: `CxWindowPool`, `CxDrawPassPool`, `CxTexturePool`, and `CxGeometryPool`. When the UI requests a new window or texture, it receives an ID referencing an index in these pools.

#### Windowing & Input Routing
Each operating system has a dedicated application loop implementation (e.g., `Win32App`, `MacosApp`, `WaylandApp`, `IosApp`).
1.  **Capture:** The native app object captures OS-specific messages (e.g., `WM_SIZE` on Windows, `NSEvent` on macOS).
2.  **Translation:** These native structs are immediately translated into Makepad's unified `Event` enum (e.g., `Event::WindowGeomChange`, `Event::MouseDown`, `Event::KeyDown`).
3.  **Injection:** The translated `Event` is injected into the global `Cx` object to be distributed to the widget tree.

#### The GPU Backend
Makepad avoids abstraction bloat by writing direct backend implementations (`metal.rs`, `d3d11.rs`, `opengl.rs`, `web_gl.rs`).
*   **Data Structures:** When a UI component requests a draw operation, it writes to a `CxDrawCall` within `Cx`. The OS-specific implementation maintains parallel structures (e.g., `D3d11Buffer`, `MetalBuffer`).
*   **Shader Compilation:** Makepad features a custom, Rust-like shader DSL. During the draw phase, `Cx` natively transpiles this DSL into the target shading language (HLSL, MSL, GLSL) at runtime, caching the compiled bytecode and pushing it to the GPU context.

---

## 2. The Event Loop

### The "What" and "Why"
**What it is:** The event loop (`Cx::event_loop`) is the continuous heartbeat of the application. It is a managed lifecycle that polls for OS events, dispatches them through the UI tree, processes state mutations, and issues hardware paint commands.

**Why it exists:** A rigidly defined loop ensures predictable state transitions. By separating the *Input Phase* from the *Layout Phase* and the *Paint Phase*, Makepad prevents race conditions and ensures that the GPU only receives fully resolved, synchronized frame data.

### How Data Flows (The Ticks and Threads)

The lifecycle begins when `Cx::event_loop()` is invoked. It takes a reference-counted, mutable borrow of `Cx` (`Rc<RefCell<Cx>>`), initializes the native GPU context, and starts the platform's message pump.

#### A Single Tick Lifecycle
1.  **Wait / Poll:** The native event loop sits in an `EventFlow::Wait` state, yielding to the OS to save battery. A high-frequency signal timer (e.g., polling every 8ms) ensures the app remains responsive to asynchronous tasks.
2.  **Dispatch (`cx.call_event_handler`):** When an event arrives (e.g., a mouse click), the native callback triggers `cx.call_event_handler(&Event::...)`. 
3.  **UI Traversal:** The event propagates down the widget tree. If a widget detects interaction (e.g., a button is clicked), it updates its internal state and flags the UI as dirty by calling `cx.redraw_all()` or targeting a specific render pass with `cx.redraw_pass_and_child_passes()`.
4.  **Side-Effect Draining:** Immediately after the widget tree processes the event, `Cx` systematically drains deferred queues:
    *   **Actions:** Processes `ActionsBuf` (inter-widget communication, like "Button X was clicked").
    *   **Triggers:** Processes localized area triggers.
    *   **Script Tasks:** Drains `handle_script_tasks()` so the live-scripting engine executes immediately rather than waiting for a subsequent frame.
5.  **The Paint Phase:** Makepad is Retained-Mode on the frontend; it *only* renders when dirty. If `redraw` was flagged during the tick, the OS loop generates a `Paint` event. Control is handed to the native GPU context (e.g., `d3d11_cx`). The GPU context iterates over the `draw_lists` inside `Cx`, updates the corresponding native instance and uniform buffers, and submits the batched draw calls to the screen.

#### Thread Synchronization
To maintain simplicity and speed, the core Event Loop and UI traversal operate strictly on the **Main Thread**. 
When heavy asynchronous work is required (e.g., network fetching, video decoding, asset loading), Makepad spawns background workers. These workers communicate back to the Main Thread via a `SignalToUI` mechanism. Firing a signal interrupts the Main Thread's `Wait` state, injecting a `Signal` event into the loop so the `Cx` can safely integrate the asynchronous results into the UI state without mutex locking.