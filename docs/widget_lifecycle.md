# The Widget System: Lifecycle & Hooks

This document details the architecture and lifecycle of the Makepad UI Widget system (`/widgets/src/widget.rs`). It explains the traits that define a Widget, how the framework traverses the UI tree, and how developers can utilize specific hooks for state management and rendering.

---

## 1. The Core Traits

The Makepad Widget system relies on a composition of traits rather than heavy inheritance. Every UI component implements `WidgetNode`, which requires `ScriptApply`, and the overarching `Widget` trait itself.

### `ScriptApply`
The foundation of Makepad's Live Design language. This trait allows a Rust struct to be dynamically constructed, updated, and modified at runtime via Makepad's scripting and styling DSL. 
*   **Trigger:** Executed during the initial parse of the live DSL or during a hot-reload event.

### `WidgetNode`
Defines structural identity within the DOM-like tree.
*   `widget_uid()`: Returns a unique `WidgetUid` (a monotonically increasing atomic integer) used to track the widget across frames.
*   `children()`: Provides an iterator to traverse child nodes, essential for structural event propagation.
*   `walk()`: Returns a `Walk` struct. This is the widget declaring its space requirements (width, height, margins) to the parent layout engine before rendering.
*   `area()`: Returns the absolute screen `Area` the widget currently occupies.
*   `redraw()`: An explicit signal sent from the Widget to the global `Cx` requesting a screen repaint.

### `Widget` (The Main Trait)
The central trait handling the core lifecycle of Event Processing and Drawing.

---

## 2. The Widget Lifecycle (The Hooks)

The lifecycle of a Makepad widget is strictly segregated into two distinct phases per frame: **Event Handling** (State Mutation) and **Drawing** (Layout & Rendering).

### Phase 1: Event Handling
The framework receives native inputs (mouse clicks, network responses, keyboard strokes) and converts them into the agnostic `Event` enum. `Cx` then walks the Widget tree.

#### `handle_event_with` / `handle_event`
*   **Signature:** `fn handle_event(&mut self, cx: &mut Cx, event: &Event, scope: &mut Scope)`
*   **Trigger:** Fired by the framework when an event enters the system. The parent widget calls this on its children.
*   **Use Case (Simple):** Checking if the event is a `Hit::FingerDown` (mouse click) that occurred within the widget's `self.area()`. If true, the widget might toggle a boolean `is_active` flag.
*   **Use Case (Complex):** A virtualization component like `PortalList` intercepting `Event::Scroll` to calculate mathematical offsets for items that are completely off-screen, thereby updating an internal buffer of which child widgets should even be initialized for the upcoming draw phase.

#### State Mutation and Redraw Requests
If `handle_event` changes the visual state of the widget, the widget *must* manually call `self.redraw(cx)`. The framework does not use generic dirty-checking. 
*   **Why?** This guarantees extreme efficiency. A massive tree can receive an event, but if no widget explicitly calls `redraw()`, the layout and paint phases are skipped entirely.

### Phase 2: Drawing and Layout
If a redraw is flagged, the framework begins the Layout/Paint phase using a "Turtle" graphics paradigm.

#### `draw_walk`
*   **Signature:** `fn draw_walk(&mut self, cx: &mut Cx2d, scope: &mut Scope, walk: Walk) -> DrawStep`
*   **Trigger:** Called by the parent widget when the framework requires the screen to be updated. The parent passes down a `Walk` struct containing layout constraints.
*   **Use Case (Simple - Styling):** A basic `Button` uses this hook to push a `DrawQuad` shader primitive into the `Cx2d` draw list, using its internal text string and color variables. It then returns `DrawStep::done()`.
*   **Use Case (Intermediate - Container Layout):** A `View` container calls `cx.begin_turtle(walk, self.layout)`. It then iterates through all its child widgets, calling `.draw_walk()` on them, allowing the turtle engine to calculate positions sequentially. Finally, it calls `cx.end_turtle_with_area()`.
*   **Use Case (Complex - State Machines):** Complex widgets use a `DrawStateWrap` to yield execution. For instance, a `Splitter` might render the left panel, yield to the framework by returning `DrawStep::make_step()`, and on the subsequent frame, resume from that exact state to calculate the dynamic drag-bar position before rendering the right panel.

#### `draw_all` / `draw_3d`
Variants of the draw loop. `draw_all` acts as a while-loop wrapper over `draw_walk` for widgets that utilize complex, multi-step yielding (`DrawStep::make_step`). `draw_3d` is utilized for nodes interacting with the 3D GLTF renderer.

---

## 3. Communication & Interactivity

Widgets in Makepad do not usually pass callbacks directly. Instead, they communicate upwards using the Actions system.

#### The `Action` System
During `handle_event`, if a widget determines something conceptually important happened (e.g., a button was clicked), it does not execute application logic. Instead, it fires an Action.
*   **Mechanism:** `cx.widget_action(self.widget_uid(), ButtonAction::Clicked)`
*   **Flow:** The event continues bubbling up. A parent application component (like `App` or `Root`) captures these actions after the event traversal finishes using `cx.capture_actions()`. It then matches against the specific widget UID and the emitted Action type to perform business logic.

#### `is_interactive`
*   **Signature:** `fn is_interactive(&self) -> bool`
*   **Trigger:** Queried by the framework during hit-testing and event bubbling.
*   **Use Case:** Overriding this to return `false` on decorative widgets (like a static `Html` text flow) ensures the layout engine completely skips them when calculating mouse hover states, saving CPU cycles.