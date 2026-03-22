# Source Files To Inspect

Use these files as the ground truth for the current Makepad shader DSL.

## Primary Reference

- `../../docs/shader_dsl.md`

Read this first when it exists. It is the consolidated reference for the current repo state.

## Compiler Surface

- `../../platform/script/src/mod_shader.rs`
  - Public shader IO markers like `instance`, `uniform`, `varying`, `texture_2d`, `fragment_output`
- `../../platform/script/src/mod_pod.rs`
  - Built-in POD scalar/vector/matrix types
- `../../platform/script/src/shader_builtins.rs`
  - Built-in functions and type rules
- `../../platform/script/src/shader_calls.rs`
  - Method call lowering, built-in pod methods, texture methods
- `../../platform/script/src/shader_vars.rs`
  - Field access, implicit IO inference, assignment rules
- `../../platform/script/src/shader_ops.rs`
  - Unary/binary operator lowering
- `../../platform/script/src/shader_control.rs`
  - `if`, `match`, `for`, `while`, `loop`, `return`, `break`, `continue`
- `../../platform/script/src/shader_tables.rs`
  - Operator type tables and matrix/vector arithmetic compatibility

## Shipped Helper Modules

- `../../draw/src/shader/sdf.rs`
  - `Sdf2d`, `Pal`, `Math`, `GaussShadow`
- `../../draw/src/shader/draw_quad.rs`
  - Core `DrawQuad` fields and conventions
- `../../draw/src/shader/draw_text.rs`
  - Text-specific shader patterns and texture sampling

## Real Examples

- `../../examples/shader/src/main.rs`
  - Clean fullscreen `pixel: fn()` example
- `../../code_editor/src/draw_selection.rs`
  - Good `Sdf2d` usage with `box`, `gloop`, and `fill`
- `../../widgets/src/view_ui.rs`
  - Many real widget shader patterns
- `../../draw/src/shader/*.rs`
  - Shipped draw shader implementations

## Search Strategy

Use repo searches before inventing syntax:

- Search for `pixel: fn(`
- Search for `vertex: fn(`
- Search for `fragment: fn(`
- Search for `texture_2d(`
- Search for `Sdf2d.viewport`
- Search for `sample_as_bgra`
- Search for `fragment_output(`

When converting an idea from GLSL/HLSL into Makepad syntax, find the closest existing pattern in these files and adapt it instead of translating blindly.
