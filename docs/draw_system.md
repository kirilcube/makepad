# Makepad Draw System Guide

This document explains how drawing works in the current Makepad library, with emphasis on:

- `draw_walk`
- `Walk`, `Layout`, turtle allocation, and `Cx2d`
- `Scope`
- when to embed and deref `View`
- when to write your own `draw_walk`
- draw-time optimization rules
- advanced draw topics: batching, stepped drawing, offscreen passes, render-to-texture, overlays, 3D interop, GPU-facing wrappers, and current limits around blur/capture

This guide is based on the current code in:

- `widgets/src/widget.rs`
- `widgets/src/view.rs`
- `draw/src/cx_2d.rs`
- `draw/src/cx_draw.rs`
- `draw/src/turtle.rs`
- `draw/src/draw_list_2d.rs`
- `platform/src/draw_pass.rs`
- `platform/src/texture.rs`
- `platform/src/draw_vars.rs`
- `platform/src/area.rs`
- `draw/src/shader/*`
- `widgets/src/3d/*`

## 1. The mental model

Makepad drawing is easiest to understand if you split it into four layers:

1. Widget traversal
   - `Widget::draw_walk` is the widget-level draw entry point.
   - `DrawStep` is the yield/resume protocol for widgets that need multiple passes over their subtree.

2. Layout allocation
   - `Walk`, `Layout`, `Flow`, `Size`, and the turtle determine where drawing is allowed to happen.
   - `cx.walk_turtle(...)` allocates a rectangle.
   - `cx.begin_turtle(...)` / `cx.end_turtle(...)` create nested layout scopes.

3. Draw-list recording
   - `DrawQuad`, `DrawText`, `DrawSvg`, `DrawVector`, `DrawPbr`, etc. do not draw immediately to the screen.
   - They record draw calls and instances into a `DrawList`.

4. Pass submission
   - `DrawPass` decides where those draw lists land: the window, a child pass, or an offscreen render target texture.
   - Backends in `platform/src/os/*` turn those recorded draw calls into Metal, D3D11, OpenGL, Vulkan, or WebGL work.

If you keep those layers separate in your head, most of the API becomes predictable.

## 2. The core widget draw contract

The widget-level API lives in `widgets/src/widget.rs`.

Primary signatures:

```rust
pub trait WidgetNode {
    fn walk(&mut self, _cx: &mut Cx) -> Walk;
    fn area(&self) -> Area;
    fn redraw(&mut self, _cx: &mut Cx);
}

pub trait Widget: WidgetNode {
    fn draw_walk(&mut self, _cx: &mut Cx2d, _scope: &mut Scope, _walk: Walk) -> DrawStep {
        DrawStep::done()
    }

    fn draw(&mut self, cx: &mut Cx2d, scope: &mut Scope) -> DrawStep {
        let walk = self.walk(cx);
        self.draw_walk(cx, scope, walk)
    }

    fn draw_walk_all(&mut self, cx: &mut Cx2d, scope: &mut Scope, walk: Walk) {
        while self.draw_walk(cx, scope, walk).is_step() {}
    }

    fn draw_all(&mut self, cx: &mut Cx2d, scope: &mut Scope) {
        while self.draw(cx, scope).is_step() {}
    }
}
```

`DrawStep` is:

```rust
pub type DrawStep = Result<(), WidgetRef>;
```

Helpers:

```rust
DrawStep::done()
DrawStep::make_step()
DrawStep::make_step_here(widget_ref)
```

Meaning:

- `Ok(())`: this widget is done drawing for this redraw cycle
- `Err(widget_ref)`: this widget wants the caller to continue drawing in multiple steps

In practice, most widgets return `DrawStep::done()`. The stepping protocol is used by container widgets and virtualized widgets.

## 3. What `draw_walk` is really for

`draw_walk` does three jobs at once:

1. Accept layout constraints from the parent
2. Allocate screen space through the turtle
3. Record draw calls into the current draw list / pass

The important point is that `draw_walk` is not just "paint". It is "allocate and paint".

That is why the signature includes `walk: Walk`: the parent already decided how this child should be placed.

If your widget ignores the provided `walk`, it is either:

- intentionally absolute / overlay style, or
- wrong

## 4. `Walk`, `Size`, `Layout`, `Flow`

These types live in `draw/src/turtle.rs`.

Primary types:

```rust
pub struct Walk {
    pub abs_pos: Option<Vec2d>,
    pub margin: Inset,
    pub width: Size,
    pub height: Size,
    pub metrics: Metrics,
}

pub enum Size {
    Fill { weight: f64, min: Option<f64>, max: Option<f64> },
    Fixed(f64),
    Fit { min: Option<FitBound>, max: Option<FitBound> },
}

pub struct Layout {
    pub scroll: Vec2d,
    pub clip_x: bool,
    pub clip_y: bool,
    pub flow: Flow,
    pub spacing: f64,
    pub padding: Inset,
    pub align: Align,
}

pub enum Flow {
    Right { row_align: RowAlign, wrap: bool },
    Down,
    Overlay,
}
```

Important constructors:

```rust
Walk::empty()
Walk::fixed(width, height)
Walk::fit()
Walk::fill()
Walk::fill_fit()

Layout::flow_right()
Layout::flow_right_wrap()
Layout::flow_down()
Layout::flow_overlay()
```

Rules of thumb:

- `Walk` describes the child.
- `Layout` describes the children of a container.
- `Flow::Right` means horizontal placement.
- `Flow::Down` means vertical placement.
- `Flow::Overlay` means all children occupy the same origin and stack.
- `Size::Fit` means "my size depends on content".
- `Size::Fill` means "distribute remaining room".
- `walk.abs_pos` makes the walk absolute instead of turtle-relative.

## 5. The turtle: the real layout engine

The turtle is the core 2D layout primitive in Makepad.

Useful `Cx2d` turtle methods:

```rust
pub fn begin_root_turtle(&mut self, size: Vec2d, layout: Layout)
pub fn begin_root_turtle_for_pass(&mut self, layout: Layout)
pub fn begin_unclipped_root_turtle_for_pass(&mut self, layout: Layout)

pub fn begin_turtle(&mut self, walk: Walk, layout: Layout)
pub fn end_turtle(&mut self) -> Rect
pub fn end_turtle_with_area(&mut self, area: &mut Area) -> Rect

pub fn walk_turtle(&mut self, walk: Walk) -> Rect
pub fn walk_turtle_with_area(&mut self, area: &mut Area, walk: Walk) -> Rect
pub fn peek_walk_turtle(&self, walk: Walk) -> Rect
pub fn walk_turtle_would_be_visible(&mut self, walk: Walk) -> bool

pub fn defer_walk_turtle(&mut self, walk: Walk) -> Option<DeferredWalk>

pub fn turtle(&self) -> &Turtle
pub fn turtle_mut(&mut self) -> &mut Turtle
pub fn current_pass_size(&self) -> Vec2d
```

What each one is for:

- `begin_root_turtle(...)`
  - start a pass-wide layout root
  - usually used by `Window`, overlays, or whole-pass rendering

- `begin_turtle(walk, layout)`
  - start a nested layout scope inside a parent-allocated rectangle

- `walk_turtle(walk)`
  - allocate one rectangle and return it
  - ideal for primitive-like widgets

- `peek_walk_turtle(walk)`
  - inspect what rect would be allocated without mutating the turtle
  - useful for aspect-ratio decisions or cheap dirty checks

- `defer_walk_turtle(walk)`
  - reserve a `Fill` walk whose exact size is not yet known
  - used by container widgets when child size depends on remaining room

- `end_turtle_with_area(&mut area)`
  - finish the turtle and also register a valid `Area` for later event/redraw updates

The turtle keeps clip state, alignment state, and row metrics. If you use it incorrectly, the failure mode is usually not a Rust error, but bad layout or mismatched begin/end panics.

## 6. `CxDraw`, `Cx2d`, `Cx3d`

These live in `draw/src/cx_draw.rs`, `draw/src/cx_2d.rs`, and `draw/src/cx_3d.rs`.

Core types:

```rust
pub struct CxDraw<'a> {
    pub cx: &'a mut Cx,
    pub draw_event: &'a DrawEvent,
    pub(crate) pass_stack: Vec<PassStackItem>,
    pub draw_list_stack: Vec<DrawListId>,
}

pub struct Cx2d<'a, 'b> {
    pub cx: &'b mut CxDraw<'a>,
    pub(crate) overlay_id: Option<DrawListId>,
    pub(crate) turtles: Vec<Turtle>,
    ...
}

pub struct Cx3d<'a, 'b> {
    pub cx: &'b mut CxDraw<'a>,
}
```

Important facts:

- `Cx2d` derefs to `CxDraw`, and `CxDraw` derefs to `Cx`
- from `Cx2d`, you can access:
  - turtle/layout methods
  - draw list methods
  - pass methods
  - lower-level `Cx` APIs

Useful `CxDraw` methods:

```rust
pub fn time(&self) -> f64
pub fn current_dpi_factor(&self) -> f64
pub fn get_current_window_id(&self) -> Option<WindowId>
pub fn inside_pass(&self) -> bool

pub fn make_child_pass(&mut self, pass: &DrawPass)
pub fn begin_pass(&mut self, pass: &DrawPass, dpi_override: Option<f64>)
pub fn end_pass(&mut self, pass: &DrawPass)

pub fn set_pass_area(&mut self, pass: &DrawPass, area: Area)
pub fn set_pass_area_with_origin(&mut self, pass: &DrawPass, area: Area, origin: Vec2d)
pub fn set_pass_shift_scale(&mut self, pass: &DrawPass, shift: Vec2d, scale: Vec2d)
```

`Cx3d` is deliberately thin. Most 3D widgets eventually create a `Cx2d` from it when they want to record draw calls.

## 7. `Scope`: what it is and what it is not

`Scope` lives in `platform/script/src/apply.rs`.

Signature:

```rust
pub struct Scope<'a, 'b> {
    pub data: ScopeDataMut<'a>,
    pub props: ScopeDataRef<'b>,
    pub index: usize,
}
```

Constructors:

```rust
Scope::with_data(&mut data)
Scope::with_props(&props)
Scope::with_data_props(&mut data, &props)
Scope::with_data_index(&mut data, index)
Scope::with_props_index(&props, index)
Scope::empty()
```

Access:

```rust
scope.data.get::<T>()
scope.data.get_mut::<T>()
scope.props.get::<T>()
scope.override_props(...)
```

What `Scope` is for:

- passing contextual state down a draw/event/apply subtree
- giving child widgets access to parent-owned state without storing references inside the widget
- parameterizing repeated template draws

What `Scope` is not for:

- global state
- layout state
- draw list state
- long-lived storage

Good examples in the current tree:

- `widgets/src/3d/scene_3d.rs`
  - scene state and per-draw-call anchors are carried through `Scope`

- `widgets/src/text_flow.rs`
  - `TextFlow` passes itself through `Scope::with_data(...)` so nested helper widgets can emit content into the parent text flow

- `widgets/src/html.rs`
  - HTML node props and current text flow are threaded through scope

If you are writing a drawing widget, the normal question is:

- do I need `Scope` to pass contextual state to children?

If no, pass `Scope::empty()` when drawing a child manually.

## 8. `Area`: the handle to something already drawn

`Area` lives in `platform/src/area.rs`.

Core type:

```rust
pub enum Area {
    Empty,
    Instance(InstanceArea),
    Rect(RectArea),
}
```

Useful methods:

```rust
pub fn redraw(&self, cx: &mut Cx)
pub fn is_empty(&self) -> bool
pub fn is_valid(&self, cx: &Cx) -> bool
pub fn valid_instance(&self, cx: &Cx) -> Option<&InstanceArea>
pub fn clipped_rect(&self, cx: &Cx) -> Rect
pub fn rect(&self, cx: &Cx) -> Rect
pub fn draw_list_id(&self) -> Option<DrawListId>
```

This is a crucial concept:

- an `Area` points at draw output from the current redraw generation
- it can become stale after the next redraw
- you can use it for:
  - hit testing
  - redraw targeting
  - patching uniforms/instances after draw

Do not treat an `Area` as a permanent geometry object. It is a live pointer into the current recorded draw data.

## 9. The three most common `draw_walk` patterns

### Pattern A: primitive widget, no `View`

Use this when:

- the widget draws one or a few primitives
- it does not host a child widget tree
- it does not need View's child traversal, scroll bars, caching, or background container behavior

Shape:

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct MyPrimitive {
    #[walk] walk: Walk,
    #[layout] layout: Layout,
    #[redraw] #[live] draw_bg: DrawQuad,
    #[rust] area: Area,
}

impl Widget for MyPrimitive {
    fn draw_walk(&mut self, cx: &mut Cx2d, _scope: &mut Scope, walk: Walk) -> DrawStep {
        let rect = cx.walk_turtle_with_area(&mut self.area, walk);
        self.draw_bg.draw_abs(cx, rect);
        DrawStep::done()
    }
}
```

Use this style for:

- custom canvas-like widgets
- chart primitives
- direct shader-backed widgets
- single-surface renderers

### Pattern B: `#[deref] view: View` and simple delegation

Use this when:

- your widget is semantically a specialized `View`
- you want child widgets, layout, events, scroll bars, background, visibility, animation, and View optimizations
- you do not need to intercept child drawing

Shape:

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct MyWrapper {
    #[deref] view: View,
}

impl Widget for MyWrapper {
    fn handle_event(&mut self, cx: &mut Cx, event: &Event, scope: &mut Scope) {
        self.view.handle_event(cx, event, scope);
    }

    fn draw_walk(&mut self, cx: &mut Cx2d, scope: &mut Scope, walk: Walk) -> DrawStep {
        self.view.draw_walk(cx, scope, walk)
    }
}
```

Use this style when your widget is really just:

- a named/styled view
- a thin behavior wrapper around existing child widgets

### Pattern C: `View` wrapper plus step loop

Use this when:

- you want View to handle the outer container mechanics
- but you need to intercept one or more child "step points"
- typical examples are `PortalList`, `FileTree`, nested `TextFlow`, and other widgets that expose stepped child draw APIs

Typical pattern:

```rust
fn draw_walk(&mut self, cx: &mut Cx2d, scope: &mut Scope, walk: Walk) -> DrawStep {
    while let Some(step) = self.view.draw_walk(cx, scope, walk).step() {
        if let Some(mut list) = step.as_portal_list().borrow_mut() {
            list.set_item_range(cx, 0, item_count);
            while let Some(item_id) = list.next_visible_item(cx) {
                let item = list.item(cx, item_id, id!(Item));
                item.draw_all(cx, &mut Scope::empty());
            }
        }
    }
    DrawStep::done()
}
```

Good examples:

- `examples/git/src/main.rs`
- `examples/splash/src/main.rs`
- `widgets/src/pdf_view.rs`

This is the most important "advanced but normal" pattern in widget code.

## 10. When to deref `View` and when not to

Use `#[deref] view: View` when most of the following are true:

- your widget owns a child widget tree
- your widget wants View's normal event bubbling
- your widget's outer layout is a normal container
- you want `show_bg`, scroll bars, animator, visibility, cursor handling, and View batching/caching
- your main custom logic is inside stepped child drawing, not raw pass management

Do not force `View` as your base when most of the following are true:

- the widget is really a primitive renderer
- the widget manages its own pass or draw list explicitly
- the widget uses a dedicated offscreen texture or pass graph
- the widget's draw lifecycle is not "background + child traversal + end"
- the widget is primarily 3D

In between those extremes:

- keep `View` if it still provides most of the outer widget mechanics
- write your own `draw_walk` if the outer draw lifecycle stops looking like a normal `View`

Concrete examples:

- `widgets/src/view.rs`
  - full container machinery

- `widgets/src/3d/scene_3d.rs`
  - does not use `View` as the outer renderer
  - it owns its draw list and 3D child traversal

- `widgets/src/popup_notification.rs`
  - owns an overlay draw list and root turtle, but still reuses an inner `View`

- `widgets/src/modal.rs`
  - same idea: custom overlay orchestration, inner view content

## 11. What `View::draw_walk` actually buys you

`View` is not just a convenience wrapper. It contains real draw-time machinery:

- child traversal
- visibility handling
- background drawing
- scroll bar integration
- deferred fill walks
- draw-list caching
- texture-pass caching
- stepped traversal protocol

Relevant fields in `widgets/src/view.rs`:

```rust
pub draw_bg: DrawQuad
pub show_bg: bool
pub layout: Layout
pub walk: Walk
optimize: ViewOptimize
draw_list: Option<DrawList2d>
texture_cache: Option<ViewTextureCache>
defer_walks: SmallVec<[(LiveId, DeferredWalk); 1]>
draw_state: DrawStateWrap<DrawState>
pub children: SmallVec<[(LiveId, WidgetRef); 2]>
```

`ViewOptimize`:

```rust
pub enum ViewOptimize {
    None,
    DrawList,
    Texture,
}
```

Interpretation:

- `None`
  - redraw the subtree normally

- `DrawList`
  - cache the draw list recording and skip subtree redraw if layout/dirty state says nothing changed

- `Texture`
  - render the subtree into a child pass with a color texture, then draw that texture as one quad

Important limitation in current `View`:

- texture caching cannot be combined with `show_bg`
- `view.rs` explicitly panics if both are used together

## 12. `DrawStateWrap`: how stepped drawing resumes

`DrawStateWrap<T>` lives in `widgets/src/widget.rs`.

Signature:

```rust
pub struct DrawStateWrap<T: Clone> {
    state: Option<T>,
    redraw_id: u64,
}
```

Useful methods:

```rust
pub fn begin(&mut self, cx: &mut CxDraw, init: T) -> bool
pub fn get(&self) -> Option<T>
pub fn set(&mut self, value: T)
pub fn end(&mut self)
```

How it works:

- on the first entry during a redraw generation, `begin(...)` stores the initial state
- later calls during the same redraw can fetch and mutate that state
- when done, call `end()`

This is how widgets like `View` can return `DrawStep::make_step()` and continue later without reconstructing traversal state.

Use `DrawStateWrap` when:

- drawing must suspend and resume
- you need to interleave parent traversal and child sub-draws
- you are implementing a stepped container, not a simple leaf widget

## 13. The draw-list layer

`DrawList2d` lives in `draw/src/draw_list_2d.rs`.

Core API:

```rust
pub struct DrawList2d {
    pub(crate) draw_list: DrawList,
    pub(crate) dirty_check_rect: Rect,
}

pub fn new(cx: &mut Cx) -> Self
pub fn begin(&mut self, cx: &mut Cx2d, walk: Walk) -> Redrawing
pub fn begin_always(&mut self, cx: &mut CxDraw)
pub fn begin_overlay_last(&mut self, cx: &mut Cx2d)
pub fn begin_overlay_reuse(&mut self, cx: &mut Cx2d)
pub fn end(&mut self, cx: &mut CxDraw)
```

`Redrawing`:

```rust
pub type Redrawing = Result<(), ()>;
```

Meaning:

- `Ok(())`: redraw this list
- `Err(())`: reuse old recorded draw items

This is a retained-mode optimization layer. The draw list remembers previously recorded draw items and can avoid rebuilding them if:

- the dirty check says the allocated rectangle is unchanged
- nothing asked that list to redraw

Important rule:

- every `begin` / `begin_always` must be paired with `end`
- if you mismatch draw list stack depth, `CxDraw::end_pass` will panic

## 14. Draw primitives: what you usually draw with

### 14.1 `DrawQuad`

`DrawQuad` is the default 2D primitive.

Core fields and methods:

```rust
pub struct DrawQuad {
    pub many_instances: Option<ManyInstances>,
    pub draw_vars: DrawVars,
    pub rect_pos: Vec2f,
    pub rect_size: Vec2f,
    pub draw_clip: Vec4f,
    pub depth_clip: f32,
    pub draw_depth: f32,
}

pub fn begin(&mut self, cx: &mut Cx2d, walk: Walk, layout: Layout)
pub fn end(&mut self, cx: &mut Cx2d)
pub fn draw_walk(&mut self, cx: &mut Cx2d, walk: Walk) -> Rect
pub fn draw_abs(&mut self, cx: &mut Cx2d, rect: Rect)
pub fn draw_rel(&mut self, cx: &mut Cx2d, rect: Rect)
pub fn update_abs(&mut self, cx: &mut Cx, rect: Rect)
pub fn begin_many_instances(&mut self, cx: &mut Cx2d)
pub fn end_many_instances(&mut self, cx: &mut Cx2d)
```

Use `DrawQuad` when:

- your shader is rectangle-based
- you want background, borders, gradient-like fills, clipped image surfaces
- you want a single widget area mapped to one instance or a batch of instances

`draw_abs` vs `draw_rel`:

- `draw_abs`
  - rect is in absolute pass coordinates

- `draw_rel`
  - rect is relative to current turtle origin

`begin` / `end` on `DrawQuad`:

- wraps a nested turtle and also pushes a draw-call parent group
- useful when the primitive itself is also a container background

### 14.2 `DrawText`

Key methods:

```rust
pub fn draw_abs(&mut self, cx: &mut Cx2d, pos: Vec2d, text: &str)
pub fn draw_walk(&mut self, cx: &mut Cx2d, walk: Walk, align: Align, text: &str) -> Rect
pub fn begin_many_instances(&mut self, cx: &mut Cx2d)
pub fn end_many_instances(&mut self, cx: &mut Cx2d)
pub fn prepare_single_line_run(&self, cx: &mut Cx2d, text: &str) -> Option<PreparedTextRun>
```

Use `DrawText` when:

- you need direct text drawing without a full `Label` widget
- you are implementing a custom text widget or editor surface

Important detail:

- `DrawText` records into the content draw-call lane, not the background lane
- it binds the font atlas textures internally through `DrawVars`

### 14.3 `DrawSvg`

Key methods:

```rust
pub fn draw_walk(&mut self, cx: &mut Cx2d, walk: Walk) -> Rect
pub fn draw_walk_time(&mut self, cx: &mut Cx2d, walk: Walk, time: f32) -> Rect
pub fn draw_abs(&mut self, cx: &mut Cx2d, rect: Rect)
```

Important behavior:

- static SVGs cache tessellated geometry
- animated SVGs retessellate every frame

### 14.4 `DrawVector`

This is the vector/tessellation API.

Useful methods:

```rust
pub fn begin(&mut self)
pub fn end(&mut self, cx: &mut Cx2d)
pub fn draw_walk(&mut self, cx: &mut Cx2d, walk: Walk, draw_fn: impl FnOnce(&mut Self, f32, f32))

pub fn rect(&mut self, x: f32, y: f32, w: f32, h: f32)
pub fn rounded_rect(&mut self, x: f32, y: f32, w: f32, h: f32, r: f32)
pub fn circle(&mut self, cx: f32, cy: f32, r: f32)
pub fn ellipse(&mut self, cx: f32, cy: f32, rx: f32, ry: f32)

pub fn set_color(&mut self, r: f32, g: f32, b: f32, a: f32)
pub fn set_color_hex(&mut self, hex: u32, alpha: f32)
pub fn set_paint(&mut self, paint: VectorPaint)

pub fn fill(&mut self)
pub fn fill_gpu(&mut self)
pub fn stroke(&mut self, stroke_width: f32)
pub fn shape_shadow(&mut self, blur: f32)
pub fn shadow(&mut self, x: f32, y: f32, w: f32, h: f32, corner: f32, blur: f32, offset_x: f32, offset_y: f32)

pub fn submit_existing_geometry(&mut self, cx: &mut Cx2d) -> bool
```

Use `DrawVector` when:

- you need retained vector geometry
- you want gradients or vector shadows
- you want to cache geometry uploads yourself

Important optimization already built in:

- `DrawVector` caches gradient rows into a texture
- `DrawSvg` can reuse uploaded vector geometry for static documents

### 14.5 `DrawPbr`

This is the main 3D draw wrapper.

Useful methods:

```rust
pub fn begin(&mut self)
pub fn reset_matrix(&mut self)
pub fn push_matrix(&mut self)
pub fn pop_matrix(&mut self)
pub fn translate(&mut self, x: f32, y: f32, z: f32)
pub fn rotate_xyz(&mut self, x_rad: f32, y_rad: f32, z_rad: f32)
pub fn scale_xyz(&mut self, x: f32, y: f32, z: f32)

pub fn set_camera_state(&mut self, view: Mat4f, projection: Mat4f, camera_pos: Vec3f)
pub fn set_clip_ndc(&mut self, clip_ndc: Vec4f)
pub fn set_depth_range(&mut self, near: f32, far: f32)
pub fn set_depth_forward_bias(&mut self, bias: f32)
pub fn set_depth_write(&mut self, depth_write: bool)

pub fn apply_material_state(&mut self, material: &DrawPbrMaterialState)
```

Use `DrawPbr` when:

- drawing retained 3D meshes
- drawing generated geometry in perspective
- bridging GLTF content to Makepad draw lists

## 15. `DrawVars`: the GPU-facing wrapper you actually patch

`DrawVars` lives in `platform/src/draw_vars.rs`.

Core type:

```rust
pub struct DrawVars {
    pub area: Area,
    pub dyn_instance_start: usize,
    pub dyn_instance_slots: usize,
    pub options: CxDrawShaderOptions,
    pub append_group_id: u64,
    pub draw_shader_id: Option<DrawShaderId>,
    pub geometry_id: Option<GeometryId>,
    pub dyn_uniforms: [f32; 256],
    pub texture_slots: [Option<Texture>; 16],
    pub uniform_buffer_slots: [Option<UniformBuffer>; 2],
    pub dyn_instances: [f32; 32],
}
```

Important methods:

```rust
pub fn set_texture(&mut self, slot: usize, texture: &Texture)
pub fn empty_texture(&mut self, slot: usize)
pub fn set_uniform_buffer(&mut self, slot: usize, uniform_buffer: &UniformBuffer)

pub fn area(&self) -> Area
pub fn redraw(&self, cx: &mut Cx)
pub fn can_instance(&self) -> bool
pub fn as_slice(&self) -> &[f32]

pub fn update_rect(&mut self, cx: &mut Cx, rect: Rect)
pub fn update_instance_area_value(&mut self, cx: &mut Cx, id: &[LiveId])
pub fn set_uniform(&mut self, cx: &Cx, uniform: LiveId, value: &[f32])
pub fn set_uniform_on_area(&mut self, cx: &mut Cx, id: LiveId, value: &[f32])
pub fn set_instance_on_area(&mut self, cx: &mut Cx, id: LiveId, value: &[f32])
```

This is the bridge between a high-level primitive and an already recorded GPU draw call.

Key use cases:

- bind textures
- bind uniform buffers
- patch a color, pan, offset, transform, or state after drawing
- update rect position/size without rebuilding the subtree

Very important constraint:

- `set_uniform_on_area` and `set_instance_on_area` only work while the `Area` is still valid for the current redraw generation

## 16. `Geometry`, `Texture`, `UniformBuffer`, `DrawPass`

These are the four main GPU-adjacent wrapper types.

### 16.1 `Geometry`

`platform/src/geometry.rs`

```rust
pub struct Geometry(...)

pub fn new(cx: &mut Cx) -> Self
pub fn geometry_id(&self) -> GeometryId
pub fn update(&self, cx: &mut Cx, indices: Vec<u32>, vertices: Vec<f32>)
pub fn update_with_recycled_buffers(&self, cx: &mut Cx, indices: &mut Vec<u32>, vertices: &mut Vec<f32>)
pub fn update_indices(&self, cx: &mut Cx, indices: Vec<u32>)
```

Use `Geometry` when:

- you want explicit control over uploaded mesh buffers
- you want to reuse geometry across frames instead of regenerating it each draw

### 16.2 `Texture`

`platform/src/texture.rs`

```rust
pub struct Texture(...)

pub fn new(cx: &mut Cx) -> Self
pub fn new_with_format(cx: &mut Cx, format: TextureFormat) -> Self
pub fn texture_id(&self) -> TextureId
pub fn set_animation(&self, cx: &mut Cx, animation: Option<TextureAnimation>)
pub fn animation(&self, cx: &mut Cx) -> &Option<TextureAnimation>
pub fn get_format(&self, cx: &mut Cx) -> &mut TextureFormat
```

Important texture formats:

```rust
TextureFormat::VecBGRAu8_32 { width, height, data, updated }
TextureFormat::VecRGBAf32 { width, height, data, updated }
TextureFormat::VecMipRGBAf32 { width, height, data, max_level, updated }
TextureFormat::VecRu8 { ... }
TextureFormat::VecRGu8 { ... }

TextureFormat::DepthD32 { size, initial }
TextureFormat::RenderBGRAu8 { size, initial }
TextureFormat::RenderRGBAf16 { size, initial }
TextureFormat::RenderRGBAf32 { size, initial }

TextureFormat::SharedBGRAu8 { width, height, id, initial }
TextureFormat::VideoYuvPlane
TextureFormat::VideoExternal
```

Interpretation:

- `Vec*`
  - CPU-owned pixel buffers uploaded to the GPU

- `Render*`
  - GPU render targets, suitable for pass color attachments

- `DepthD32`
  - depth attachment for a pass

- `SharedBGRAu8`
  - cross-process / presentable-image interop path

- `Video*`
  - backend-managed video surfaces

Important practical detail:

- render target textures are treated as Y-up
- `widgets/src/image.rs` flips them when displaying through `Image`
- if you sample a render texture manually in your own shader, handle the Y orientation yourself

### 16.3 `UniformBuffer`

`platform/src/uniform_buffer.rs`

```rust
pub struct UniformBuffer(...)

pub fn new(cx: &mut Cx) -> Self
pub fn clear(&self, cx: &mut Cx)
pub fn set_bytes(&self, cx: &mut Cx, data: &[u8])
pub fn set_struct<T: Copy>(&self, cx: &mut Cx, value: &T)
pub fn set_struct_slice<T: Copy>(&self, cx: &mut Cx, values: &[T])
```

Use `UniformBuffer` when:

- your shader needs structured non-instance data too large or awkward for `dyn_uniforms`
- you want to share the same block between draw calls

### 16.4 `DrawPass`

`platform/src/draw_pass.rs`

Important API:

```rust
pub struct DrawPass(...)

pub fn new(cx: &mut Cx) -> Self
pub fn new_with_name(cx: &mut Cx, name: &str) -> Self

pub fn set_pass_parent(&self, cx: &mut Cx, pass: &DrawPass)
pub fn set_size(&self, cx: &mut Cx, pass_size: Vec2d)
pub fn size(&self, cx: &mut Cx) -> Option<Vec2d>
pub fn set_window_clear_color(&self, cx: &mut Cx, clear_color: Vec4f)

pub fn clear_color_textures(&self, cx: &mut Cx)
pub fn add_color_texture(&self, cx: &mut Cx, texture: &Texture, clear_color: DrawPassClearColor)
pub fn set_color_texture(&self, cx: &mut Cx, texture: &Texture, clear_color: DrawPassClearColor)
pub fn set_depth_texture(&self, cx: &mut Cx, texture: &Texture, clear_depth: DrawPassClearDepth)

pub fn set_debug(&mut self, cx: &mut Cx, debug: bool)
pub fn set_dpi_factor(&mut self, cx: &mut Cx, dpi: f64)
```

Pass clear enums:

```rust
pub enum DrawPassClearColor {
    InitWith(Vec4f),
    ClearWith(Vec4f),
}

pub enum DrawPassClearDepth {
    InitWith(f32),
    ClearWith(f32),
}
```

Use `DrawPass` when:

- rendering to a window
- rendering offscreen to a texture
- building multi-pass effects
- separating different scenes or render scales

## 17. Normal drawing recipes

### Recipe A: leaf widget that draws one rect

```rust
fn draw_walk(&mut self, cx: &mut Cx2d, _scope: &mut Scope, walk: Walk) -> DrawStep {
    let rect = cx.walk_turtle_with_area(&mut self.area, walk);
    self.draw_bg.draw_abs(cx, rect);
    DrawStep::done()
}
```

### Recipe B: custom container without `View`

```rust
fn draw_walk(&mut self, cx: &mut Cx2d, scope: &mut Scope, walk: Walk) -> DrawStep {
    self.draw_bg.begin(cx, walk, self.layout);

    self.header.draw_all(cx, scope);
    self.body.draw_all(cx, scope);

    self.draw_bg.end(cx);
    DrawStep::done()
}
```

Use this if you want a very explicit container lifecycle and do not need `View`.

### Recipe C: `View` plus interception

```rust
fn draw_walk(&mut self, cx: &mut Cx2d, scope: &mut Scope, walk: Walk) -> DrawStep {
    while let Some(step) = self.view.draw_walk(cx, scope, walk).step() {
        if let Some(mut portal) = step.as_portal_list().borrow_mut() {
            portal.set_item_range(cx, 0, self.items.len());
            while let Some(item_id) = portal.next_visible_item(cx) {
                let item = portal.item(cx, item_id, id!(Row));
                item.draw_all(cx, &mut Scope::empty());
            }
        }
    }
    DrawStep::done()
}
```

This is the standard pattern for virtualized child drawing.

## 18. Optimization rules that matter

These are the rules that actually affect performance in the current system.

### 18.1 Prefer `View` caching modes over custom pass graphs unless needed

If your subtree is a normal widget subtree:

- use `new_batch` / `ViewOptimize::DrawList` first
- use `texture_caching` / `ViewOptimize::Texture` only if compositing the subtree as one textured quad is worthwhile

Texture caching helps when:

- subtree is expensive to redraw
- subtree can be reused as one image
- transforms/compositing are cheaper than rebuilding the subtree

Draw-list caching helps when:

- subtree draw recording is expensive
- but you do not want an offscreen render target

### 18.2 Do not rebuild geometry if you can update or reuse it

Prefer:

- `Geometry::update_with_recycled_buffers(...)`
- `DrawVector::submit_existing_geometry(...)`
- `DrawVars::update_rect(...)`
- `DrawVars::set_uniform_on_area(...)`
- `DrawVars::set_instance_on_area(...)`

over:

- recreating widget subtrees every frame
- regenerating vector/mesh buffers when only small parameters changed

### 18.3 Use `peek_walk_turtle` before you mutate expensive state

If aspect ratio, texture selection, or caching depends on the resulting rect:

- inspect with `peek_walk_turtle(walk)` first
- only allocate after you know what you need to draw

### 18.4 Use `defer_walk_turtle` when writing custom containers with fill children

If your container has `Fill` children and you do not use `View`, you need to deal with deferred walks yourself.

`View` already does this:

- try `cx.defer_walk_turtle(walk)`
- if successful, store the `DeferredWalk`
- resolve later with `dw.resolve(cx)`

If you ignore this and just call `walk_turtle`, `Fill` children in right/down flow containers will behave incorrectly.

### 18.5 Batch repeated primitives

Prefer `begin_many_instances` / `end_many_instances` when drawing lots of the same primitive with the same shader/material state.

Available on:

- `DrawQuad`
- `DrawText`
- `DrawPbr`

Batching helps when:

- only per-instance data changes
- textures/uniforms/material do not change between instances

### 18.6 Keep texture and uniform changes stable across batches

Draw call appendability depends on:

- same shader
- same geometry
- same dynamic uniforms
- same texture slots
- same uniform buffer slots
- same draw-call options
- same append group

If any of those change, Makepad usually has to start a new draw call.

### 18.7 Use visibility checks for large virtualized content

Useful helpers:

```rust
cx.walk_turtle_would_be_visible(walk)
```

Or use widgets built around virtualization:

- `PortalList`
- `FileTree`
- stepped child drawing patterns

### 18.8 Respect begin/end pairing

Bad pairing causes stack corruption or hard panics.

Pair these correctly:

- `begin_pass` / `end_pass`
- `begin_root_turtle` / `end_pass_sized_turtle`
- `begin_turtle` / `end_turtle`
- `DrawList2d::begin*` / `end`
- `DrawQuad::begin` / `end`
- `Overlay::begin` / `end`

### 18.9 Do not hold stale `Area` handles across redraw generations

If you store `Area` and patch it later:

- always assume it may be stale on the next redraw
- use `area.is_valid(cx)` or methods that already validate

## 19. Advanced: draw-call grouping and ordering

`Cx2d` exposes implicit grouping helpers:

```rust
pub fn push_draw_call_parent(&mut self)
pub fn pop_draw_call_parent(&mut self)
pub fn draw_call_group_parent(&self) -> LiveId
pub fn draw_call_group_current(&self) -> LiveId
pub fn draw_call_group_background(&self) -> LiveId
pub fn draw_call_group_content(&self) -> LiveId
```

These exist so background and content draw calls can batch without breaking local ordering.

By convention:

- background-like primitives use `draw_call_group_background()`
- content-like primitives use `draw_call_group_content()`

That is why:

- `DrawQuad` writes to the background lane
- `DrawText` writes to the content lane

There is also explicit shader-side `draw_call_group`.

Example:

```rust
draw_selection +: {
    draw_call_group: @selection
}
```

Use explicit groups when:

- you need a stable sub-layer inside a shared parent lane
- you want selection/highlight layers that should not merge with ordinary content

Do not overuse this. Every extra ordering barrier can reduce batching opportunities.

## 20. Advanced: overlays

Overlays are draw-list routing, not a separate window or pass.

Core API:

```rust
pub struct Overlay {
    pub draw_list: DrawList,
}

pub fn begin(&self, cx: &mut Cx2d)
pub fn end(&self, cx: &mut Cx2d)
```

Typical overlay draw-list entry:

```rust
draw_list.begin_overlay_reuse(cx);
cx.begin_root_turtle_for_pass(layout);
...
cx.end_pass_sized_turtle();
draw_list.end(cx);
```

Overlay use cases:

- modal surfaces
- tooltips
- popup menus
- notifications
- always-on-top transient UI

Good examples:

- `widgets/src/modal.rs`
- `widgets/src/popup_notification.rs`
- `widgets/src/tooltip.rs`
- `widgets/src/popup_menu.rs`

Two useful overlay modes:

- `begin_overlay_reuse`
  - normal overlay reuse path

- `begin_overlay_last`
  - force overlay draw list to append last

## 21. Advanced: offscreen passes and render-to-texture

This is the main path for multi-pass rendering in Makepad.

The pattern is:

1. create a `DrawPass`
2. create one or more render target `Texture`s
3. attach textures to the pass
4. mark the pass as a child of the current pass
5. `begin_pass`
6. draw into that pass
7. `end_pass`
8. sample or display the resulting texture later

Minimal skeleton:

```rust
let pass = DrawPass::new(cx);
let color = Texture::new_with_format(
    cx,
    TextureFormat::RenderBGRAu8 {
        size: TextureSize::Auto,
        initial: true,
    },
);

pass.set_color_texture(
    cx,
    &color,
    DrawPassClearColor::ClearWith(vec4(0.0, 0.0, 0.0, 0.0)),
);

cx.make_child_pass(&pass);
cx.begin_pass(&pass, None);
cx.begin_root_turtle_for_pass(Layout::flow_down());

// draw subscene here

cx.end_pass_sized_turtle();
cx.end_pass(&pass);
```

Then either:

- bind `color` with `draw_vars.set_texture(...)` and sample it in a custom shader
- or show it with `Image`

Important practical details:

- if the pass should track a specific on-screen area, call `cx.set_pass_area(&pass, area)`
- if you show a render texture in `Image`, it will auto-flip Y
- if you sample it in your own shader, account for Y orientation yourself

The built-in example inside the widget system is `ViewOptimize::Texture` in `widgets/src/view.rs`.

## 22. Advanced: multi-pass rendering recipes

### Recipe A: cache a subtree into a texture

This is already implemented in `View`.

The important sequence is:

1. create child pass
2. attach `RenderBGRAu8` texture
3. begin child pass
4. draw subtree
5. end child pass
6. draw the resulting texture back into parent pass as a quad
7. call `set_pass_area` so pass rect follows the on-screen area

Use this for:

- expensive static or semi-static UI subtrees
- thumbnailing a subtree
- reusing a composed subtree in another effect

### Recipe B: render to one texture, then post-process it in another pass

Typical chain:

1. scene pass -> color texture A
2. effect pass -> sample A, output texture B
3. final pass -> sample B

For blur-like pipelines, this becomes:

1. source pass -> texture A
2. horizontal blur pass -> texture B
3. vertical blur pass -> texture C
4. final composite pass -> sample C

The engine gives you the pass and texture primitives to do this. What it does not give you is a ready-made widget-level blur node.

### Recipe C: render a pass at a different scale

You can:

- create a child pass
- set an explicit pass size with `DrawPass::set_size(...)`
- or apply pass shift/scale with `cx.set_pass_shift_scale(...)`

This is useful for:

- low-resolution blur inputs
- thumbnails
- zoomed sub-views
- alternate projection mappings

If you do this, remember that the sampled texture and the on-screen area are now decoupled. Make the mapping explicit.

## 23. Advanced: blur and capture, what exists and what does not

This is where people often over-assume engine support.

### 23.1 What exists directly

1. Offscreen render-to-texture
   - yes, through `DrawPass` + `Render*` textures

2. Shader-side sampling from textures you already own
   - yes, via `texture_2d(float)` and `draw_vars.set_texture(...)`

3. Vector-style shadow blur
   - yes, through `DrawVector::shadow(...)` and `DrawVector::shape_shadow(...)`

4. OS compositor backdrop blur / material
   - yes, through `WindowBackdrop`
   - values: `None`, `Auto`, `Mica`, `Acrylic`, `Vibrancy`, `Blur`
   - this is platform-dependent and not the same as a shader-readable scene texture

5. Backend-level texture capture for video encode and studio tooling
   - yes, inside backend/media/platform internals
   - not exposed as a simple general-purpose widget draw API

### 23.2 What does not exist as a normal high-level widget API

1. "Give me the current framebuffer as a texture"
   - no public general widget-layer API

2. "Sample everything already drawn behind this widget"
   - not as a generic standard draw primitive

3. "Built-in gaussian blur widget that automatically captures the scene behind it"
   - not in the current general draw API

### 23.3 Important current limitation: `GlassPanel` is not true scene capture blur

In `widgets/src/glass_panel.rs`, `use_scene_blur` and `blur_amount` currently affect style mixing in the shader.

That code does not sample the already-drawn scene texture.

So:

- it is a glass-like styling parameter
- it is not proof of a full scene-capture blur pipeline in the widget draw stack

### 23.4 So how do you blur something in practice?

You have three real options:

1. Blur your own source texture
   - render the target content into a pass texture
   - blur that texture in one or more subsequent passes

2. Use vector/shadow blur where appropriate
   - for shadows and soft geometry effects

3. Use OS backdrop materials
   - only for window-level compositor blur/material, not arbitrary scene subregions

If you need true "backdrop blur behind a panel inside the same UI scene", you need to explicitly build the render graph that produces the texture to blur. There is no generic shortcut exposed at the widget layer today.

## 24. Advanced: how to capture already-drawn content to a texture

Short answer:

- you generally do not capture "already drawn" content after the fact
- you structure the draw graph so the content you need is drawn into an offscreen pass first

That is the important architectural shift.

Instead of:

- draw scene
- later ask the GPU to hand it back as a reusable texture

Prefer:

- draw scene into child pass texture
- reuse that texture in later draws or passes

Why:

- it is explicit
- it avoids hidden readback/copy paths
- it is the path the current widget/passes API is designed for

Backend internals do contain readback/capture paths for studio screenshots and video encoding, but those are platform integration mechanisms, not the normal public draw abstraction.

## 25. Advanced: 3D draw path

The 3D path still records into draw lists; it is not a separate renderer bolted onto the side.

Relevant pieces:

- `Cx3d`
- `DrawPbr`
- `DrawCube`
- `DrawText3d`
- `Scene3D`
- `View3D`
- `Gltf3D`

Important current pattern:

- many 3D widgets create a temporary `Cx2d` from `Cx3d`
- then record their draw calls through normal draw infrastructure

Examples:

- `widgets/src/3d/scene_3d.rs`
- `widgets/src/3d/view_3d.rs`
- `widgets/src/3d/gltf_3d.rs`

`Scene3D` is a good reference for a custom outer draw lifecycle:

- allocate a rect with `cx.walk_turtle(walk)`
- draw a background quad
- begin a dedicated `DrawList2d`
- build scoped 3D scene state in `Scope`
- draw 3D children
- optionally reorder draw calls by computed depth anchors

This is a strong example of "write your own `draw_walk` instead of delegating to `View`".

## 26. Advanced: draw-call reordering and anchoring

`Scene3D` also demonstrates an advanced but legitimate technique:

- record draw-call anchors during 3D drawing
- compute desired draw order from those anchors
- patch `draw_item_reorder`

This is not normal 2D UI code, but it is useful to know:

- draw lists are retained objects that can be reordered after recording
- `Area` and `DrawListId` can serve as stable handles within one redraw generation

Use this kind of logic only when you really need scene-aware ordering. It is not a substitute for normal local draw order.

## 27. Advanced: updating already-recorded draws

One of the more underused features in the current draw stack is post-draw patching.

Available tools:

- `DrawVars::update_rect(...)`
- `DrawVars::update_instance_area_value(...)`
- `DrawVars::set_uniform_on_area(...)`
- `DrawVars::set_instance_on_area(...)`

This is useful when:

- layout did not change much
- geometry does not need rebuilding
- only a small instance/uniform value changed

Examples:

- `Image` updates panning/scaling state without rebuilding everything
- cursor-like overlays can patch rects directly

Use this carefully:

- the `Area` must still be valid
- the targeted instance/uniform must exist in the compiled shader mapping

## 28. Advanced: `ManyInstances`

`ManyInstances` is the explicit batching API used by several primitives.

Typical flow:

```rust
self.draw_bg.begin_many_instances(cx);
for rect in rects {
    self.draw_bg.rect_pos = rect.pos.into();
    self.draw_bg.rect_size = rect.size.into();
    self.draw_bg.draw(cx);
}
self.draw_bg.end_many_instances(cx);
```

Or equivalent primitive-specific helpers.

Use this when:

- you are drawing many instances of the same shader/geometry/state

Avoid it when:

- texture or material changes every draw
- per-item branching makes the batch unstable
- the count is tiny and code complexity is not justified

## 29. Advanced: custom geometry and custom shader wrappers

If built-in primitives are too high-level, the next layer down is:

- custom `DrawVars`
- optional `Geometry`
- optional `UniformBuffer`
- your own shader struct with `#[repr(C)]`

Rules that matter:

- shader-backed draw structs must keep memory layout valid
- draw shader structs that extend another draw shader with `#[deref]` must respect field ordering rules
- instance-mapped fields must remain contiguous after the base draw shader storage

In practice:

- copy existing patterns from `draw/src/shader/*`
- do not improvise struct layout
- treat `DrawVars::as_slice()` semantics as layout-sensitive

## 30. Advanced: root passes, windows, and pass-sized turtles

The main window path shows the canonical whole-pass render lifecycle:

```rust
cx.begin_pass(&self.pass.handle, None);
self.main_draw_list.begin_always(cx);

let size = cx.current_pass_size();
cx.begin_root_turtle(size, Layout::flow_down());

...

cx.end_pass_sized_turtle();
self.main_draw_list.end(cx);
cx.end_pass(&self.pass.handle);
```

Important point:

- when you render a full pass, use a root turtle sized to the pass
- when you render a child subtree inside another container, use `begin_turtle`

Use `end_pass_sized_turtle()` when the root turtle corresponds to the pass extent.

## 31. Advanced: image and render texture display

`Image` is a useful reference because it handles both normal textures and render textures.

Important behavior in `widgets/src/image.rs`:

- if bound texture format `is_render()`, `Image` flips Y automatically

That makes `Image` the easiest way to inspect a render target texture in UI.

For custom shader sampling:

- bind render texture with `draw_vars.set_texture(slot, &texture)`
- sample in shader with `texture_2d(float)`
- handle orientation explicitly

## 32. Advanced: shared textures, studio, and backend wrappers

The platform layer contains wrappers around OS/GPU interop objects that are relevant if you are working below normal widget code.

Examples:

- `Texture`
  - normal GPU texture wrapper

- `SharedBGRAu8`
  - presentable/shared image wrapper

- `HostSwapchain`
  - studio/runview/shared-presentable image infrastructure in `platform/src/os/shared_framebuf.rs`

- backend media capture hooks
  - `CxMediaApi::video_encoder_capture_texture_frame(...)`

These are real GPU/OS wrappers, but they live below ordinary widget code.

Practical guidance:

- for normal UI work, stay at `DrawPass` + `Texture`
- only go deeper if you are building tooling, custom platform interop, studio integration, or video pipelines

## 33. What to copy from the library

If you want the safest path, copy patterns from these files:

- Container and stepping:
  - `widgets/src/view.rs`
  - `examples/git/src/main.rs`
  - `examples/splash/src/main.rs`

- Overlay orchestration:
  - `widgets/src/modal.rs`
  - `widgets/src/popup_notification.rs`
  - `widgets/src/tooltip.rs`

- Offscreen pass / texture caching:
  - `widgets/src/view.rs`

- Whole-pass rendering:
  - `widgets/src/window.rs`
  - `examples/teamtalk/src/main.rs`
  - `examples/arracing/src/main.rs`

- 3D composition:
  - `widgets/src/3d/scene_3d.rs`
  - `widgets/src/3d/gltf_3d.rs`

- Vector and retained geometry:
  - `draw/src/shader/draw_vector.rs`
  - `draw/src/shader/draw_svg.rs`

## 34. Decision checklist

When implementing a new draw widget, ask these in order:

1. Is this basically a normal container?
   - use `#[deref] view: View`

2. Is this a normal container but I need to intercept a child step?
   - keep `View`, wrap `view.draw_walk(...)` in a step loop

3. Is this mostly a custom primitive?
   - implement your own `draw_walk` with `walk_turtle` or `begin_turtle`

4. Do I need children and a custom outer render lifecycle?
   - own the outer `draw_walk`, possibly keep an inner `View`

5. Do I need offscreen output?
   - use `DrawPass` + render target `Texture`

6. Do I need to reuse already-rendered content?
   - structure the graph so it renders to texture first

7. Do I need blur?
   - blur your own texture, use vector shadow blur, or use `WindowBackdrop`
   - do not assume there is a built-in generic backdrop-capture blur widget

8. Am I changing only a few parameters?
   - patch instances/uniforms on an existing `Area` instead of rebuilding everything

## 35. Short summary

The most important practical rules are:

- `draw_walk` is layout allocation plus draw recording
- the turtle is the real 2D layout engine
- `View` is worth keeping when you want normal container behavior
- step loops are the standard way to customize `View` child drawing
- write a full custom `draw_walk` when your lifecycle stops looking like a normal container
- `DrawPass` + render target `Texture` is the correct advanced path for multi-pass work
- there is no simple public "capture current framebuffer behind me" widget-level API
- `Area` lets you patch what was already recorded, but only while that area is still valid

Once those rules are clear, most draw-related code in the library becomes easy to classify.
