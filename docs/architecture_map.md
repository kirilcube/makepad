# Architecture Map

## 1. Table of Contents: Codebase Structure

**1. Platform & OS Abstraction (`/platform`)**
The bedrock of the engine, providing a unified interface over diverse operating systems and hardware.
*   **OS Integrations (`/platform/src/os`)**: Windowing, event loops, file systems, and input.
    *   **Apple**: macOS, iOS, tvOS (Windowing, AVFoundation).
    *   **Linux**: X11, Wayland, Android, OpenHarmony.
    *   **Windows**: Win32, Media Foundation.
    *   **Web**: WebAssembly, Web Audio, WebSockets, JS Interop.
*   **GPU Backends**: Native graphics API wrappers.
    *   Apple: `metal.rs`
    *   Windows: `d3d11.rs`, `angle.rs`
    *   Linux/Android: `vulkan.rs`, `opengl.rs`
    *   Web: `web_gl.rs`
*   **Core Context (`Cx`)**: The global state container (`cx.rs`, `cx_api.rs`).
*   **Event System**: Unified input and application events (`event.rs`, `action.rs`, `game_input.rs`).
*   **Media Interfaces**: Audio, Video, and MIDI abstractions.
*   **Live/Scripting Engine**: The runtime representation of Makepad's UI DSL (`script/`).

**2. Drawing Engine (`/draw`)**
The intermediate layer bridging raw GPU calls and high-level UI layouts.
*   **Drawing Contexts**: Context wrappers for specific dimensions (`cx_2d.rs`, `cx_3d.rs`, `cx_draw.rs`).
*   **Layout Engine (`turtle.rs`)**: A flow-based layout system operating on a turtle-graphics paradigm (walk, align, bounds).
*   **Shaders & Primitives (`/draw/src/shader`)**: Abstracted hardware-accelerated drawing elements (Quads, Text, Glyphs, PBR, Vectors, SVG).
*   **Typography & Text (`/draw/src/text`)**: Font handling, rendering, and caching.
*   **Asset Management**: Image caching and network fetching (`image_cache.rs`).

**3. UI Widgets Library (`/widgets`)**
The high-level, reusable building blocks for user interfaces.
*   **Core Layout**: `view`, `adaptive_view`, `scroll_bar`, `splitter`, `dock`, `tab`.
*   **Standard Controls**: `button`, `text_input`, `check_box`, `slider`, `drop_down`.
*   **Complex/Data Views**: `file_tree`, `portal_list` (virtualized lists), `slides_view`.
*   **Media & Advanced**: `video`, `map`, `pdf_view`, `chart`, `widgets_3d`, `voice_wave`.
*   **Theming**: Pre-built aesthetics (`theme_desktop_dark`, `theme_desktop_light`).
*   **Widget Trait & Tree**: The core architecture for traversing and interacting with the widget hierarchy.

**4. Code Editor (`/code_editor`)**
A specialized, high-performance text editing component built from scratch using the drawing and widget layers (supports syntax highlighting, inlays, and text selection).

**5. Tools & Build System (`/tools`)**
*   `cargo_makepad`: Custom CLI for cross-compiling, packaging, and installing toolchains (WASM, Android, Apple).
*   Various shell scripts for fetching assets, system dependencies, and publishing.

**6. External & Internal Libraries (`/libs`)**
A constellation of vendored, forked, or bespoke crates optimized for Makepad's strict dependency and performance requirements (e.g., `makepad_math`, `rustybuzz`, `ttf-parser`, `pulldown-cmark`).

**7. Makepad Studio (`/studio`) & Examples (`/examples`)**
*   **Studio**: The integrated IDE built with Makepad *for* Makepad, facilitating live editing and AI workflows.
*   **Examples**: Standalone apps demonstrating capabilities (3D GLTF, maps, splashing UIs, physics).

---

## 2. High-Level Architectural Overview (The Request Flow)

Makepad utilizes an Immediate-Mode-inspired backend with a Retained-Mode (Tree) frontend, tied together by a reactive Live Design language. Here is how the layers interact from the metal to the pixel.

### Layer 1: The OS & Hardware (The Host)
*   **Initialization:** The application starts via the `app_main!` macro, which yields control to a platform-specific OS loop (e.g., `win32_app.rs`, `mac_os_app.rs`, or the WASM browser event loop).
*   **Event Generation:** The OS intercepts hardware signals (mouse clicks, keystrokes, window resizes) and passes them to the OS abstraction layer. 
*   **Unified Translation:** The OS layer translates platform-specific structs into Makepad's agnostic `Event` enum.

### Layer 2: The Core Context (`Cx`)
*   **The Nervous System:** The translated `Event` is injected into `Cx` (Context). `Cx` is the central God-object passed mutably down the entire application tree. It holds the window state, input state, layout buffers, and the GPU draw command lists.
*   **Pass/View Architecture:** Instead of rendering immediately, the `Cx` allocates memory for a `DrawList` inside a `DrawPass`. 

### Layer 3: The Framework & Layout (`draw/`)
*   **Event Routing:** `Cx` calls down to the root Widget. The Widget tree performs an `Event` traversal. If a button detects a `MouseDownEvent` within its bounding box, it registers state changes and requests a redraw (`cx.need_redraw()`).
*   **Turtle Layout:** On a draw pass, the application uses the `Turtle` layout engine. Widgets declare their space requirements (`Walk`). The Turtle moves a virtual cursor across the screen, calculating absolute coordinates (Quads) and bounding boxes for every element.
*   **Draw Commands:** As the Turtle lays out space, widgets push draw instructions via Shaders (`DrawQuad`, `DrawText`) into the `CxDrawList`. These instructions are mathematically represented as instances of GPU structs.

### Layer 4: The GPU Backend
*   **Batching & Compilation:** Once the application finishes the draw traversal, control returns to the OS loop. The OS loop extracts the `DrawList` and `UniformBuffer` data from `Cx`.
*   **Hardware Submission:** The respective graphics backend (`metal`, `vulkan`, `d3d11`, `web_gl`) takes over. It compiles any dirty shaders (written in Makepad's Rust-like shader DSL) into the native shading language (HLSL, MSL, GLSL, SPIR-V).
*   **Execution:** Instance data arrays and uniform buffers are pushed to the GPU, rendering the entire UI in heavily batched, parallelized draw calls.

### Summary Interaction Loop:
`OS Window` -> *Yields Input* -> `Cx (Event Phase)` -> *Widgets Update State* -> `Cx (Draw Phase)` -> *Turtle calculates layout* -> *Widgets emit Draw Primitives* -> `DrawList` -> `GPU Backend` -> *Pixels on Screen*.