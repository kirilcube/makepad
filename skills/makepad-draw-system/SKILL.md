---
name: makepad-draw-system
description: "Explain or implement draw-related work in the current Makepad repository. Use when working on `draw_walk`, `DrawStep`, `Walk` and `Layout`, turtle allocation, `Cx2d` and `Cx3d`, `Scope`, `Area`, `View`-backed widgets, draw-list optimization, passes, render-to-texture, overlays, 3D interop, or current blur and capture limitations."
---

# Makepad Draw System

Use this skill when a task is about how Makepad widgets draw, how to structure custom draw code, or how drawing maps to passes, textures, and GPU-facing wrappers in this repo.

## Start Here

1. Read the canonical repo reference first:
   - `../../docs/draw_system.md`
2. Read [references/source-files.md](references/source-files.md) for the exact engine and widget files to inspect.
3. Search for an existing pattern in the repo before inventing a new draw structure.

Prefer real repo patterns over assumptions.

## Working Rules

- Treat `../../docs/draw_system.md` as the primary reference for draw behavior in this repo.
- Start from the simplest valid draw shape:
  - deref `View` and reuse `self.view.draw_walk(...)` when the widget is mainly composition
  - write a custom `draw_walk` when layout allocation, stepping, passes, or draw order are genuinely custom
- Keep the widget draw contract precise:
  - `draw_walk` records draw data
  - it does not perform immediate-mode painting
  - `DrawStep` is the yield and resume protocol
- Respect turtle ownership:
  - every `begin_turtle(...)` must be paired with `end_turtle(...)`
  - allocate with `walk_turtle(...)` only when you really need a rect
- Prefer child widgets and existing draw primitives before inventing new low-level wrappers.
- For offscreen or multi-pass work, prefer explicit `DrawPass` plus render textures instead of implying a magical screen capture API.

## Decision Rules

- Use embedded `View` drawing when you want:
  - child traversal
  - template expansion
  - deferred fill handling
  - optional `ViewOptimize` caching
  - a normal widget subtree with custom logic around specific children
- Write your own outer `draw_walk` when you need:
  - custom turtle structure
  - stepped traversal over children
  - manual item virtualization or draw-order control
  - multi-pass or render-to-texture orchestration
  - custom 3D and 2D interop
- Do not claim that a general widget-level "capture what is already behind me and blur it" API exists unless you verified a concrete implementation in the current repo.
  - The default answer in this repo is explicit passes and textures, or OS window backdrop materials when that is the actual requirement.

## Optimization Rules

- Keep draw structs stable and reuse them across frames.
- Prefer `ViewOptimize::DrawList` or `ViewOptimize::Texture` only when redraw locality justifies caching.
- Minimize unnecessary pass creation, texture allocation, and redraw invalidation.
- Reuse shipped primitives like `DrawQuad`, `DrawText`, `DrawSvg`, `DrawVector`, and `DrawPbr` before writing new wrappers.
- Reuse existing geometry, textures, and buffers when possible instead of rebuilding them per draw.

## When You Need More Detail

- Use `../../docs/draw_system.md` as the primary draw reference.
- Use [references/source-files.md](references/source-files.md) to jump to the key engine, widget, and example files.
