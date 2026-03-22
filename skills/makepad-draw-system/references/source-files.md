# Source Files To Inspect

Use these files as the ground truth for drawing behavior in the current Makepad repository.

## Primary Reference

- `../../docs/draw_system.md`

Read this first when it exists. It is the consolidated reference for the current repo state.

## Widget Draw Contract

- `../../widgets/src/widget.rs`
  - `Widget`, `WidgetNode`, `DrawStep`, widget traversal helpers
- `../../widgets/src/view.rs`
  - `View` draw stepping, deferred fill walks, draw-list caching, texture caching, overlays
- `../../platform/script/src/apply.rs`
  - `Scope`, `ScopeDataRef`, `ScopeDataMut`, prop and data plumbing

## 2D Draw Engine

- `../../draw/src/cx_2d.rs`
  - `Cx2d`, draw-state helpers, nesting, overlay helpers
- `../../draw/src/cx_draw.rs`
  - draw-list stack management, passes, redraw tracking
- `../../draw/src/turtle.rs`
  - `Walk`, `Layout`, turtle allocation, defer logic, pass-sized roots
- `../../draw/src/draw_list_2d.rs`
  - draw-list internals, append and reuse behavior

## GPU-Facing Wrappers

- `../../platform/src/draw_pass.rs`
  - `DrawPass`, pass targets, clear settings, pass graph
- `../../platform/src/texture.rs`
  - `Texture`, `TextureFormat`, render targets, shared and video textures
- `../../platform/src/draw_vars.rs`
  - `DrawVars`, instance payload packing, texture bindings
- `../../platform/src/geometry.rs`
  - custom geometry and vertex buffers
- `../../platform/src/uniform_buffer.rs`
  - user uniform buffers
- `../../platform/src/area.rs`
  - `Area`, hit and redraw handles

## Shipped Draw Primitives

- `../../draw/src/shader/draw_quad.rs`
  - baseline 2D quad primitive
- `../../draw/src/shader/draw_text.rs`
  - text draw path and atlas-backed sampling
- `../../draw/src/shader/draw_svg.rs`
  - SVG texture-backed drawing
- `../../draw/src/shader/draw_vector.rs`
  - vector drawing
- `../../draw/src/shader/draw_pbr.rs`
  - 3D and PBR draw path

## Widgets And Examples

- `../../widgets/src/window.rs`
  - root window pass orchestration
- `../../widgets/src/modal.rs`
  - overlay-style draw ordering
- `../../widgets/src/popup_notification.rs`
  - overlay entry usage
- `../../widgets/src/image.rs`
  - showing textures and render textures in the widget layer
- `../../widgets/src/glass_panel.rs`
  - current blur-related styling surface and its limits
- `../../widgets/src/3d/scene_3d.rs`
  - custom 3D scene orchestration
- `../../widgets/src/3d/view_3d.rs`
  - 3D view wrappers
- `../../examples/teamtalk/src/main.rs`
  - render-to-texture and display pattern
- `../../examples/arracing/src/main.rs`
  - explicit pass and texture setup
- `../../examples/exf/src/main.rs`
  - multi-pass style patterns in a real app

## Search Strategy

Use repo searches before inventing a pattern:

- Search for `fn draw_walk`
- Search for `self.view.draw_walk`
- Search for `while let Some(item)`
- Search for `begin_turtle`
- Search for `DrawPass::new`
- Search for `TextureFormat::Render`
- Search for `set_pass_area`
- Search for `redraw_id`

When a task asks for "how drawing works", start with `../../docs/draw_system.md`, then confirm the exact implementation in the files above.
