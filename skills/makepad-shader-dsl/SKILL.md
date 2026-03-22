---
name: makepad-shader-dsl
description: "Write or modify Makepad shader DSL code for the current Makepad repository. Use when working on `script_mod!` shader blocks, `vertex: fn()`, `fragment: fn()`, `pixel: fn()`, `DrawQuad`-based shaders, texture sampling, SDF drawing, uniforms, instances, varyings, or when translating GLSL/HLSL-style shader logic into Makepad's current shader syntax."
---

# Makepad Shader DSL

Use this skill to implement or edit shaders in the current Makepad tree.

## Start Here

1. Read the canonical repo reference first if it exists:
   - `../../docs/shader_dsl.md`
2. Read [references/source-files.md](references/source-files.md) for the exact compiler and example files to inspect.
3. Search for an existing pattern in the repo before inventing syntax.

Prefer real repo patterns over assumptions.

## Working Rules

- Treat `vertex` and `fragment` as the only true entry points.
- Treat `pixel` as a normal helper by convention.
- If a shader uses `pixel`, ensure `fragment` writes the fragment output explicitly, usually `self.fb0 = self.pixel()`.
- Preserve current Makepad script syntax:
  - Use `Name: value`, never `Name = value`
  - Use `name := Type{...}` for named widget instances outside shader bodies
- Prefer inheriting `..mod.draw.DrawQuad` when the shader is a 2D rect/quad draw shader.
- Reuse shipped helpers instead of reimplementing them:
  - `Sdf2d`
  - `Pal`
  - `Math`
  - `GaussShadow`

## Field and IO Rules

- Remember the current implicit IO rules:
  - Unmarked scalar `f32`/`f16`/`u32`/`i32` fields become instances
  - Unmarked `vec*`/`mat*` fields become uniforms
- Use explicit markers for fields that are not inferable or where clarity matters:
  - `uniform(...)`
  - `instance(...)`
  - `varying(...)`
  - `uniform_buffer(...)`
  - `vertex_buffer(...)`
  - `texture_* (...)`
  - `fragment_output(...)`
  - `vertex_position(...)`
- Do not rely on implicit inference for `bool`, structs, textures, or uniform buffers.

## Authoring Workflow

1. Identify the base shader.
   - For common UI shaders, start from `mod.draw.DrawQuad`.
   - For text-specific work, inspect `mod.draw.DrawText`.
2. Identify what `self` already provides.
   - Existing instance/live fields
   - Uniform buffers like `draw_call`, `draw_pass`, `draw_list`
   - Vertex buffers like `geom`
   - Existing varyings
3. Add only the fields you need.
4. Implement or update helper functions before entry points if the shader logic is non-trivial.
5. Implement `vertex` and `fragment`.
6. If using `pixel`, keep it focused on color generation and let `fragment` forward to it.
7. Compare the final syntax against existing repo examples before finishing.

## Common Pitfalls

- Do not say "`pixel` is the fragment entry point". It is not.
- Do not use GLSL-only syntax that this compiler does not support.
- Do not use `Name = value` syntax in script blocks.
- Do not rely on undefined `self` fields; verify them against the base shader or Rust struct.
- Use `atan2(y, x)`, not two-arg `atan(y, x)`.
- Use `modf(x, y)` for float modulo in this DSL.
- Use `#x...` for hex colors whose literal would otherwise confuse the tokenizer because of `e`/`E`.
- Do not assume bool fields infer to uniforms or instances; mark them explicitly.

## Implementation Checklist

- Confirm the shader compiles against current repo syntax, not archived `old/` syntax.
- Confirm every `self.<field>` comes from:
  - the current shader block,
  - an inherited shader block,
  - a referenced uniform buffer/vertex buffer/texture,
  - or an outer scope binding intentionally captured by the shader compiler.
- Confirm stage writes are valid:
  - `vertex` may write varyings and vertex position
  - `fragment` may write fragment outputs
- Confirm texture methods and helper calls match current names from the repo.

## When You Need More Detail

- Use `../../docs/shader_dsl.md` as the primary language reference when available.
- Use [references/source-files.md](references/source-files.md) to jump to the compiler, helper modules, and working examples.
