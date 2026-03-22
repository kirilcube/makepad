# Makepad Shader DSL Reference

This document is a source-backed reference for the shader DSL used by the current Makepad tree in this repository.

What this document covers completely:

- The built-in shader language surface compiled by `platform/script/src/*`
- All public shader IO markers from `mod.shader`
- All built-in shader math/utility functions and constants
- All built-in texture methods
- All shipped helper shader objects from `draw/src/shader/sdf.rs`:
  - `GaussShadow`
  - `Math`
  - `Pal`
  - `Sdf2d`
- The common fields/methods exposed by the shipped `mod.draw.DrawQuad` base shader
- The built-in geometry PODs commonly used as vertex buffers

What cannot be a closed list:

- Any field you add to your own shader object
- Any field you add to your own `#[repr(C)]` draw shader struct
- Any custom POD `struct` or methods you define
- Any extra names you bring in from the surrounding script scope

Those names are user-defined by design. This document explains exactly how they become visible inside shader code, but it cannot enumerate identifiers that only exist after you define them.

Primary sources used:

- `platform/script/src/mod_shader.rs`
- `platform/script/src/mod_pod.rs`
- `platform/script/src/shader.rs`
- `platform/script/src/shader_builtins.rs`
- `platform/script/src/shader_calls.rs`
- `platform/script/src/shader_vars.rs`
- `platform/script/src/shader_ops.rs`
- `platform/script/src/shader_control.rs`
- `platform/script/src/shader_tables.rs`
- `platform/script/test/src/main.rs`
- `draw/src/shader/sdf.rs`
- `draw/src/shader/draw_quad.rs`
- `draw/src/shader/draw_text.rs`
- `platform/src/draw_list.rs`
- `platform/src/draw_pass.rs`
- `draw/src/geometry/geometry_gen.rs`

## 1. What Is Actually Special

The real entry points are:

- `vertex: fn()`
- `fragment: fn()`

`pixel: fn()` is not a magical compiler entry point. It is a very common convention used by shipped draw shaders. The usual pattern is:

```rust
fragment: fn() {
    self.fb0 = self.pixel()
}
```

So:

- `vertex` is the vertex-stage entry point
- `fragment` is the fragment-stage entry point
- `pixel` is just a normal helper function unless `fragment` calls it

Inside shader functions:

- `self` is the shader object itself
- `self.<field>` accesses your shader fields, IO fields, uniform buffers, varyings, textures, and inherited draw-shader fields
- Local variables come from `let` and `var`
- Names from the outer script scope are also visible; plain POD values become scope uniforms, while scope textures and uniform buffers stay textures and buffers

## 2. Core Syntax

Typical structure:

```rust
set_type_default() do #(MyShader::script_shader(vm)) {
    ..mod.draw.DrawQuad

    time: 0.0
    tint: instance(#fff)
    tex: texture_2d(float)
    uv: varying(vec2f)

    helper: fn(x: f32) -> f32 {
        return x * 2.0
    }

    fast_helper: |x| x * 2.0

    vertex: fn() {
        self.vertex_pos = self.clip_and_transform_vertex(self.rect_pos, self.rect_size)
    }

    fragment: fn() {
        self.fb0 = self.pixel()
    }

    pixel: fn() -> vec4 {
        let u = self.pos.x
        var c = vec3(u)
        if u > 0.5 {
            c = c.mix(vec3(1.0, 0.0, 0.0), 0.5)
        }
        return vec4(c, 1.0)
    }
}
```

You can define:

- Fields: `name: expr`
- IO fields: `name: instance(...)`, `uniform(...)`, `varying(...)`, etc.
- Functions: `name: fn(args...) -> Ret { ... }`
- Short lambdas: `name: |x, y| expr` or `name: || { ... }`

Local declarations:

- Immutable: `let name = expr`
- Mutable: `var name = expr`
- Both require an initializer in shaders

Function syntax patterns used by the compiler/tests:

```rust
helper: fn(x: f32) -> f32 { return x * 2.0 }
pixel: fn() -> vec4 { return #fff }
translate: fn(x: f32, y: f32) -> vec2 { self.pos -= vec2(x, y); return self.pos }
ctor: fn(p: vec2) -> Self { return self(pos: p, result: vec4(0.0), dist: 0.0) }

short0: || 1.0
short1: |x| x * 2.0
short2: |a, b| { return a + b }
```

## 3. Types

### 3.1 Scalar POD types

| Type | Aliases | Notes |
| --- | --- | --- |
| `void` | none | Used as the absence of a value / return type |
| `bool` | none | Boolean |
| `f32` | `float` | 32-bit float |
| `f16` | none | 16-bit float |
| `u32` | `uint` | 32-bit unsigned integer |
| `i32` | `int` | 32-bit signed integer |
| `atomic_u32` | none | Defined POD type |
| `atomic_i32` | none | Defined POD type |

### 3.2 Vector POD types

Float vectors:

- `vec2f`, alias `vec2`
- `vec3f`, alias `vec3`
- `vec4f`, alias `vec4`

Half vectors:

- `vec2h`
- `vec3h`
- `vec4h`

Unsigned-int vectors:

- `vec2u`
- `vec3u`
- `vec4u`

Signed-int vectors:

- `vec2i`
- `vec3i`
- `vec4i`

Bool vectors:

- `vec2b`
- `vec3b`
- `vec4b`

### 3.3 Matrix POD types

Float matrices only:

- `mat2x2f`
- `mat2x3f`
- `mat2x4f`
- `mat3x2f`
- `mat3x3f`
- `mat3x4f`
- `mat4x2f`
- `mat4x3f`
- `mat4x4f`

### 3.4 Other POD/container types

- `struct`
- `array`

## 4. Literals and Constructors

### 4.1 Numeric literals

Explicit typed forms seen in the compiler tests:

- `1f`, `1.0f` style for `f32`
- `1h` for `f16`
- `1i` for `i32`
- `1u` for `u32`

Untyped literals like `1` and `1.0` exist too. They start as abstract numeric literals and are resolved by context.

### 4.2 Boolean literals

- `true`
- `false`

### 4.3 Color literals

The shader/script DSL accepts color literals such as:

- `#f00`
- `#ff0000`
- `#0000`
- `#ffffffff`

Important parser caveat in this repo:

- If the hex color would contain a digit immediately followed by `e` or `E`, use `#x...`
- Example: write `#x2ecc71`, not `#2ecc71`

### 4.4 Constructors

Scalars:

```rust
f32(x)
f16(x)
i32(x)
u32(x)
bool(x)
```

Vectors:

```rust
vec2(1.0)
vec3(1.0, 2.0, 3.0)
vec4(vec3(1.0, 2.0, 3.0), 1.0)
vec4(1.0)            // splat
vec2u(1u, 2u)
vec4i(1i)            // splat
```

Rules implemented by the constructor checker:

- Scalar constructors take exactly 1 argument
- Vector constructors take either:
  - 1 argument (splat), or
  - enough scalar/vector arguments to fill all lanes
- Matrix constructors take either:
  - 1 argument, or
  - the full element count
- Struct constructors require all fields
- Struct constructors support both:
  - positional arguments
  - named arguments
- `array(...)` builds a fixed-size array with element type inferred from its contents

Examples:

```rust
let a = vec4(1.0)
let b = vec4(vec3(1.0, 2.0, 3.0), 1.0)
let s = self(pos: vec2(0.0), result: vec4(0.0), dist: 0.0)
let arr = array(1.0, 2.0, 3.0, 4.0)
```

## 5. Variables and Name Resolution

Inside a shader function, identifiers can come from four places.

### 5.1 Local variables

- `let` creates an immutable local
- `var` creates a mutable local
- Both require an initializer

Examples:

```rust
let uv = self.pos
var sum = 0.0
```

### 5.2 Function parameters

Parameters are available by name inside the function body.

Examples:

```rust
helper: fn(x: f32, y: f32) -> f32 {
    return x + y
}
```

### 5.3 Shader object fields through `self`

These include:

- Your own shader fields
- Inherited base-shader fields
- Explicit shader IO fields (`instance`, `uniform`, `varying`, etc.)
- Implicitly inferred IO fields
- Uniform-buffer fields, texture fields, vertex-buffer fields

### 5.4 Names from outer script scope

The shader compiler can capture names from the surrounding script scope:

- Plain POD values become scope uniforms
- `shader.uniform_buffer(...)` values stay scope uniform buffers
- `shader.texture_* (...)` values stay scope textures
- Plain objects can be traversed; their POD properties become scope uniforms
- Methods on scope objects can be called if they are script functions

This is why code like this works:

```rust
let scope_time = 1.5
let scope_buf = shader.uniform_buffer(scope_uniforms)
let scope_tex = shader.texture_2d(float)
```

and then inside the shader:

```rust
let t = scope_time
let x = scope_buf.time
let c = scope_tex.sample(vec2(0.5, 0.5))
```

## 6. Shader IO Fields

### 6.1 Explicit IO markers

These are the public DSL entry points from `mod.shader`.

| DSL | Resulting shader-side type | Notes |
| --- | --- | --- |
| `instance(value: T)` | `T` | Per-instance dynamic input |
| `uniform(value: T)` | `T` | Dynamic uniform |
| `uniform_buffer(value: Struct)` | `Struct` | Uniform-buffer object |
| `vertex_buffer(vertex_type: Struct, buffer_handle = nil)` | `Struct` | Vertex-buffer element type; second arg is optional in the current script API |
| `varying(value: T)` | `T` | Vertex-to-fragment varying |
| `vertex_position(value: vec4f)` | `vec4f` | Vertex-stage position output |
| `fragment_output(index: u32, ty: T)` | `T` | Fragment output `0..7` |
| `texture_1d(sample_ty)` | `Texture1d` | Texture handle |
| `texture_1d_array(sample_ty)` | `Texture1dArray` | Texture handle |
| `texture_2d(sample_ty)` | `Texture2d` | Texture handle |
| `texture_2d_array(sample_ty)` | `Texture2dArray` | Texture handle |
| `texture_3d(sample_ty)` | `Texture3d` | Texture handle |
| `texture_3d_array(sample_ty)` | `Texture3dArray` | Texture handle |
| `texture_cube(sample_ty)` | `TextureCube` | Texture handle |
| `texture_cube_array(sample_ty)` | `TextureCubeArray` | Texture handle |
| `texture_depth(sample_ty)` | `TextureDepth` | Texture handle |
| `texture_depth_array(sample_ty)` | `TextureDepthArray` | Texture handle |
| `texture_video(sample_ty)` | `TextureVideo` | Texture handle |

Notes:

- `fragment_output(index, ty)` clamps the index to `0..7`
- The public shader module currently exposes texture constructors, but not a public `sampler(...)` constructor
- Texture method calls auto-create and use the default sampler internally

### 6.2 Implicit IO inference for unmarked `self` fields

If a shader field has no explicit marker, the compiler applies these rules:

- Unmarked scalar `f32`, `f16`, `u32`, `i32` -> implicit `instance`
- Unmarked `vec*` or `mat*` -> implicit `uniform`
- Unmarked `bool`, `struct`, `array`, and other non-scalar/non-vec/non-mat types are **not** inferred

If inference fails, the compiler reports:

```text
shader field `<name>` needs explicit IO marker (uniform/instance/varying) or implicit scalar/vec/mat type
```

So these are valid:

```rust
time: 0.0           // implicit instance (f32)
color: vec4(1.0)    // implicit uniform (vec4f)
```

But these need explicit markers:

```rust
enabled: uniform(true)
buf: uniform_buffer(MyUniformStruct)
tex: texture_2d(float)
```

### 6.3 Read/write rules by stage

Readable from shader code:

- Instance fields
- Uniforms
- Uniform buffers
- Vertex buffers
- Varyings
- Textures
- Scope uniforms/buffers/textures

Writable in `vertex`:

- `self.<varying>`
- `self.<vertex_position>`

Writable in `fragment`:

- `self.<fragment_outputN>`

Not writable from shader code:

- Uniforms
- Instance inputs
- Uniform buffers
- Vertex buffers
- Textures

## 7. Member Access, Swizzles, and Indexing

### 7.1 Struct field access

If a POD struct has fields, access them with:

```rust
self.draw_pass.time
self.draw_list.view_shift
my_struct.some_field
```

### 7.2 Vector component names and swizzles

Supported component letters:

- geometric: `x y z w`
- color aliases: `r g b a`

Rules:

- Length 1 swizzle returns a scalar
- Length 2 returns a `vec2*`
- Length 3 returns a `vec3*`
- Length 4 returns a `vec4*`
- Reordering and repetition are allowed
- Every referenced component must exist in the source vector dimension

Examples:

```rust
v.x
v.xy
v.xyz
v.wzyx
color.rgb
color.rgba
uv.xx
```

### 7.3 Indexing

Arrays, vectors, and matrices are indexable.

Allowed index types:

- abstract integer literal
- `i32`
- `u32`

Examples:

```rust
let arr = array(1.0, 2.0, 3.0, 4.0)
let x = arr[0]

var v = vec4(1.0, 2.0, 3.0, 4.0)
let y = v[2]
v[3] = 10.0
```

Matrix indexing returns a column vector:

- `mat2x2f[i] -> vec2f`
- `mat3x3f[i] -> vec3f`
- `mat4x4f[i] -> vec4f`
- similarly for the non-square matrix families

## 8. Operators

### 8.1 Unary operators

| Operator | Supported types | Result |
| --- | --- | --- |
| `-x` | `f32`, `f16`, `i32`, `vec*f`, `vec*h`, `vec*i`, abstract numeric | Same type |
| `!x` | `bool`, `i32`, `u32` | `bool` logical-not, or integer bitwise-not |

`!` is not a generic numeric operator in shaders. It only works on:

- `bool` -> logical negation
- `i32`, `u32` -> bitwise negation

### 8.2 Floating-point arithmetic family

The `+ - * /` operators use the float-arithmetic type tables.

Supported families include:

- scalar/scalar float math
- scalar/vector broadcast for float and half vectors
- vector/vector element-wise math for matching vector families
- scalar/matrix and matrix/scalar float multiplication/division/add/sub
- vector/matrix and matrix/vector multiplication for float matrices
- matrix/matrix multiplication for matching float matrix dimensions

Important practical cases:

- `f32 op vecNf -> vecNf`
- `vecNf op f32 -> vecNf`
- `vecNf op vecNf -> vecNf`
- `mat * vec -> vec` for supported dimensions
- `vec * mat -> vec` for supported dimensions
- `mat * mat -> mat` for the supported float matrix families

### 8.3 Integer arithmetic family

The integer-only operators are:

- `%`
- `<<`
- `>>`
- `&`
- `|`
- `^`

Supported types:

- `i32`
- `u32`
- matching int vectors (`vec2i/3i/4i`, `vec2u/3u/4u`)

These integer-only operators do **not** use float-style scalar broadcast on int vectors. Use matching scalar types or matching int-vector types.

### 8.4 Comparisons

Supported comparison operators:

- `==`
- `!=`
- `<`
- `<=`
- `>`
- `>=`

The comparison compatibility table covers:

- scalar floats/ints/bools -> scalar `bool`
- vector float/int families compared with matching vector families -> `vec2b`, `vec3b`, or `vec4b`

Examples:

```rust
let a = 1.0 < 2.0
let b = vec2(1.0) == vec2(2.0)
```

### 8.5 Logical operators

Supported:

- `&&`
- `||`

Supported type:

- `bool` only

### 8.6 Assignment

Supported forms:

- `x = expr`
- `x += expr`
- `x -= expr`
- `x *= expr`
- `x /= expr`
- `x %= expr`
- `x &= expr`
- `x |= expr`
- `x ^= expr`
- `x <<= expr`
- `x >>= expr`

These work on:

- mutable locals (`var`)
- writable struct fields
- writable vector components
- writable array/vector indices
- writable shader outputs (`varying`, `vertex_position`, fragment outputs) in the correct stage

## 9. Control Flow

The shader compiler supports:

- `if`
- `else if`
- `else`
- expression-form `if`
- `match`
- `for i in start..end`
- `while condition { ... }`
- `loop { ... }`
- `break`
- `continue`
- `return`
- `discard()`

### 9.1 `if` statements

```rust
if x > 0.0 {
    return 1.0
} else if x == 0.0 {
    return 0.0
} else {
    return -1.0
}
```

### 9.2 Expression-form `if`

```rust
let sign = if x < 0.0 { -1.0 } else { 1.0 }
```

Both branches must resolve to the same type, or to compatible numeric scalar types that the compiler can unify.

### 9.3 `match`

Used in the compiler tests:

```rust
let result = match self.enum_test {
    ShaderEnum.Test1 => 1.0
    ShaderEnum.Test2 => 2.0
    _ => 0.0
}
```

### 9.4 `for`

```rust
for i in 0..4 {
    sum += 1.0
}
```

Shader-specific rule:

- Shader `for` loops are currently lowered as `u32` loops
- If the range is `i32`, it is coerced to `u32`

### 9.5 `while`, `loop`, `break`, `continue`

```rust
while count < 10.0 {
    count += 1.0
}

loop {
    i += 1.0
    if i > 10.0 { break }
    if i == 3.0 { continue }
}
```

### 9.6 `return` and `discard`

`return` is fully supported inside nested `if`s and loops.

`discard()` is the built-in fragment-discard function:

```rust
if alpha < 0.01 {
    discard()
}
```

Signature:

- `discard() -> void`

## 10. Built-in Constants

These are injected into `mod.math` and are available in shader code when that module is in scope.

| Name | Type | Value |
| --- | --- | --- |
| `PI` | `f32` | `3.141592653589793` |
| `E` | `f32` | `2.718281828459045` |
| `LN2` | `f32` | `0.6931471805599453` |
| `LN10` | `f32` | `2.302585092994046` |
| `LOG2E` | `f32` | `1.4426950408889634` |
| `LOG10E` | `f32` | `0.4342944819032518` |
| `SQRT1_2` | `f32` | `0.70710678118654757` |
| `TORAD` | `f32` | `0.017453292519943295` |
| `GOLDEN` | `f32` | `1.618033988749895` |

## 11. Built-in Functions

### 11.1 Unary float / float-vector built-ins

For the signatures below, let:

- `T in { f32, f16, vec2f, vec3f, vec4f, vec2h, vec3h, vec4h }`

Then the following are all:

- `name(x: T) -> T`

Functions:

- `acos`
- `acosh`
- `asin`
- `asinh`
- `atan`
- `atanh`
- `ceil`
- `cos`
- `cosh`
- `degrees`
- `exp`
- `exp2`
- `floor`
- `fract`
- `inverseSqrt`
- `log`
- `log2`
- `radians`
- `round`
- `sin`
- `sinh`
- `sqrt`
- `tan`
- `tanh`
- `trunc`
- `dFdx`
- `dFdy`

Examples:

```rust
let a = sin(PI * 0.5)
let b = floor(vec2(1.2, 2.8))
let c = dFdx(self.pos)
```

### 11.2 Unary float/int and vector float/int built-ins

For the signatures below, let:

- `T in { f32, f16, u32, i32, vec2f, vec3f, vec4f, vec2h, vec3h, vec4h, vec2u, vec3u, vec4u, vec2i, vec3i, vec4i }`

Then:

- `abs(x: T) -> T`
- `sign(x: T) -> T`

### 11.3 `length`

Supported overloads:

- `length(x: f32) -> f32`
- `length(x: f16) -> f16`
- `length(x: vec2f) -> f32`
- `length(x: vec3f) -> f32`
- `length(x: vec4f) -> f32`
- `length(x: vec2h) -> f16`
- `length(x: vec3h) -> f16`
- `length(x: vec4h) -> f16`

### 11.4 `normalize`

Supported overloads:

- `normalize(v: vec2f) -> vec2f`
- `normalize(v: vec3f) -> vec3f`
- `normalize(v: vec4f) -> vec4f`
- `normalize(v: vec2h) -> vec2h`
- `normalize(v: vec3h) -> vec3h`
- `normalize(v: vec4h) -> vec4h`

### 11.5 Matching 2-argument float built-ins

For:

- `T in { f32, f16, vec2f, vec3f, vec4f, vec2h, vec3h, vec4h }`

The following require both arguments to have the same `T`:

- `atan2(y: T, x: T) -> T`
- `pow(x: T, y: T) -> T`
- `modf(x: T, y: T) -> T`

Notes:

- In this DSL, `modf(x, y)` means floating remainder / modulo
- On WGSL backends the compiler emits the equivalent remainder expression directly because WGSL's builtin `modf` has a different meaning

### 11.6 `step`

Supported overloads:

- `step(edge: f32, x: f32) -> f32`
- `step(edge: f16, x: f16) -> f16`
- `step(edge: vec2f, x: vec2f) -> vec2f`
- `step(edge: vec3f, x: vec3f) -> vec3f`
- `step(edge: vec4f, x: vec4f) -> vec4f`
- `step(edge: vec2h, x: vec2h) -> vec2h`
- `step(edge: vec3h, x: vec3h) -> vec3h`
- `step(edge: vec4h, x: vec4h) -> vec4h`
- `step(edge: f32, x: vec2f) -> vec2f`
- `step(edge: f32, x: vec3f) -> vec3f`
- `step(edge: f32, x: vec4f) -> vec4f`
- `step(edge: f16, x: vec2h) -> vec2h`
- `step(edge: f16, x: vec3h) -> vec3h`
- `step(edge: f16, x: vec4h) -> vec4h`

### 11.7 `distance` and `dot`

Supported overloads:

- `distance(a: f32, b: f32) -> f32`
- `distance(a: f16, b: f16) -> f16`
- `distance(a: vec2f, b: vec2f) -> f32`
- `distance(a: vec3f, b: vec3f) -> f32`
- `distance(a: vec4f, b: vec4f) -> f32`
- `distance(a: vec2h, b: vec2h) -> f16`
- `distance(a: vec3h, b: vec3h) -> f16`
- `distance(a: vec4h, b: vec4h) -> f16`

- `dot(a: f32, b: f32) -> f32`
- `dot(a: f16, b: f16) -> f16`
- `dot(a: vec2f, b: vec2f) -> f32`
- `dot(a: vec3f, b: vec3f) -> f32`
- `dot(a: vec4f, b: vec4f) -> f32`
- `dot(a: vec2h, b: vec2h) -> f16`
- `dot(a: vec3h, b: vec3h) -> f16`
- `dot(a: vec4h, b: vec4h) -> f16`

### 11.8 `cross`

Supported overloads:

- `cross(a: vec3f, b: vec3f) -> vec3f`
- `cross(a: vec3h, b: vec3h) -> vec3h`

### 11.9 `max` and `min`

For:

- `T in { f32, f16, u32, i32, vec2f, vec3f, vec4f, vec2h, vec3h, vec4h, vec2u, vec3u, vec4u, vec2i, vec3i, vec4i }`

Supported overloads:

- `max(a: T, b: T) -> T`
- `min(a: T, b: T) -> T`

### 11.10 `mix`

Core overload:

- `mix(x: T, y: T, a: T) -> T`

where:

- `T in { f32, f16, vec2f, vec3f, vec4f, vec2h, vec3h, vec4h }`

Extra convenience overloads implemented by the type table:

- `mix(x: vec2f, y: vec2f, a: f32|f16|i32|u32) -> vec2f`
- `mix(x: vec3f, y: vec3f, a: f32|f16|i32|u32) -> vec3f`
- `mix(x: vec4f, y: vec4f, a: f32|f16|i32|u32) -> vec4f`

Note:

- That scalar-alpha convenience is implemented for `vec*f`
- It is **not** implemented as a generic vector-half convenience overload in the current type table

### 11.11 `smoothstep` and `fma`

For:

- `T in { f32, f16, vec2f, vec3f, vec4f, vec2h, vec3h, vec4h }`

Supported overloads:

- `smoothstep(edge0: T, edge1: T, x: T) -> T`
- `fma(a: T, b: T, c: T) -> T`

### 11.12 `clamp`

For:

- `T in { f32, f16, u32, i32, vec2f, vec3f, vec4f, vec2h, vec3h, vec4h, vec2u, vec3u, vec4u, vec2i, vec3i, vec4i }`

Supported overload:

- `clamp(value: T, min: T, max: T) -> T`

### 11.13 Built-in method sugar on POD values

The compiler recognizes these method names on POD values and lowers them to the matching free built-in, using the receiver as argument 0:

- `x.mix(y, a)` -> `mix(x, y, a)`
- `x.clamp(lo, hi)` -> `clamp(x, lo, hi)`
- `edge0.smoothstep(edge1, x)` -> `smoothstep(edge0, edge1, x)`
- `edge.step(x)` -> `step(edge, x)`
- `a.min(b)` -> `min(a, b)`
- `a.max(b)` -> `max(a, b)`

These are mechanical rewrites. They do not represent a separate runtime method table.

## 12. Texture Methods

All texture handles support the following built-in methods.

### 12.1 `size`

- `texture.size() -> vec2f`

Returns the texture dimensions as floats.

### 12.2 `sample`

- `texture.sample(coord) -> vec4f`

Shipped usage patterns:

- 2D/2D-array/depth/video textures are sampled with `vec2f`
- Cube textures are sampled with `vec3f`

### 12.3 `sample_as_bgra`

- `texture.sample_as_bgra(coord) -> vec4f`

Semantics:

- Same as `sample`, except the GLSL/WebGL path uses a BGRA-aware sampler helper
- On the other backends it is effectively the same as `sample`

### 12.4 `sample_lod`

- `texture.sample_lod(coord, lod: f32) -> vec4f`

### 12.5 `sample_video`

- `texture.sample_video(coord) -> vec4f`

Used for external/video textures. The GLSL path has special handling for OES/external texture sampling.

### 12.6 Sampler behavior

Texture built-ins use an auto-generated default sampler with:

- normalized coordinates
- linear filtering
- clamp-to-edge behavior

There is currently no public manual sampler constructor in `mod.shader`.

## 13. Standard Shader Helper Modules from `mod.sdf`

When the widget prelude is used, these helper objects are imported from `draw/src/shader/sdf.rs`.

### 13.1 `GaussShadow`

All methods currently defined on `GaussShadow`:

| Name | Signature | Notes |
| --- | --- | --- |
| `gaussian` | `gaussian(x: float, sigma: float) -> float` | Internal helper |
| `erf_vec2` | `erf_vec2(x0: vec2) -> vec2` | Internal helper |
| `erf_vec4` | `erf_vec4(x0: vec4) -> vec4` | Internal helper |
| `rounded_box_shadow_x` | `rounded_box_shadow_x(x: float, y: float, sigma: float, corner: float, half_size: vec2) -> float` | Internal helper |
| `rounded_box_shadow` | `rounded_box_shadow(lower: vec2, upper: vec2, point: vec2, sigma: float, corner: float) -> float` | Rounded rectangle shadow |
| `box_shadow` | `box_shadow(lower: vec2, upper: vec2, point: vec2, sigma: float) -> float` | Box shadow |

### 13.2 `Math`

| Name | Signature | Notes |
| --- | --- | --- |
| `rotate_2d` | `rotate_2d(v: vec2, a: float) -> vec2` | Rotate vector `v` by angle `a` |
| `random_2d` | `random_2d(v: vec2) -> float` | Hash-style pseudo-random |

### 13.3 `Pal`

| Name | Signature |
| --- | --- |
| `premul` | `premul(v: vec4) -> vec4` |
| `iq` | `iq(t: float, a: vec3, b: vec3, c: vec3, d: vec3) -> vec3` |
| `iq0` | `iq0(t: float) -> vec3` |
| `iq1` | `iq1(t: float) -> vec3` |
| `iq2` | `iq2(t: float) -> vec3` |
| `iq3` | `iq3(t: float) -> vec3` |
| `iq4` | `iq4(t: float) -> vec3` |
| `iq5` | `iq5(t: float) -> vec3` |
| `iq6` | `iq6(t: float) -> vec3` |
| `iq7` | `iq7(t: float) -> vec3` |
| `hsv2rgb` | `hsv2rgb(c: vec4) -> vec4` |
| `rgb2hsv` | `rgb2hsv(c: vec4) -> vec4` |

### 13.4 `Sdf2d`

#### Fields

| Field | Type | Meaning |
| --- | --- | --- |
| `pos` | `vec2` | Current sample position |
| `result` | `vec4` | Accumulated premultiplied result |
| `last_pos` | `vec2` | Path helper state |
| `start_pos` | `vec2` | Path helper state |
| `shape` | `float` | Current shape distance |
| `clip` | `float` | Clip distance |
| `has_clip` | `float` | Clip enabled flag |
| `old_shape` | `float` | Previous shape distance |
| `blur` | `float` | Blur width |
| `aa` | `float` | Anti-aliasing width |
| `scale_factor` | `float` | Current transform scale |
| `dist` | `float` | Most recently computed primitive distance |

#### Methods

| Name | Signature | Notes |
| --- | --- | --- |
| `antialias` | `antialias(p: vec2) -> float` | Helper used by `viewport` |
| `viewport` | `viewport(pos: vec2) -> Self` | Constructor-like entry point |
| `translate` | `translate(x: float, y: float) -> vec2` | Mutates `self.pos` |
| `rotate` | `rotate(a: float, x: float, y: float)` | Rotates current position around point |
| `scale` | `scale(f: float, x: float, y: float)` | Scales current position around point |
| `clear` | `clear(color: vec4)` | Clears/accumulates premultiplied color |
| `calc_blur` | `calc_blur(w: float) -> float` | Internal blur helper |
| `fill_keep_premul` | `fill_keep_premul(source: vec4) -> vec4` | Fill without resetting shape state |
| `fill_premul` | `fill_premul(color: vec4) -> vec4` | Fill premultiplied and reset state |
| `fill_keep` | `fill_keep(color: vec4) -> vec4` | Non-premul helper |
| `fill` | `fill(color: vec4) -> vec4` | Most common fill entry point |
| `stroke_keep` | `stroke_keep(color: vec4, width: float) -> vec4` | Stroke without resetting state |
| `stroke` | `stroke(color: vec4, width: float) -> vec4` | Stroke and reset state |
| `glow_keep` | `glow_keep(color: vec4, width: float) -> vec4` | Glow without resetting state |
| `glow` | `glow(color: vec4, width: float) -> vec4` | Glow and reset state |
| `union` | `union()` | Boolean union using `dist`/`old_shape` |
| `intersect` | `intersect()` | Boolean intersection |
| `subtract` | `subtract()` | Boolean subtraction |
| `gloop` | `gloop(k: float)` | Smooth union |
| `blend` | `blend(k: float)` | Linear blend between distances |
| `circle` | `circle(x: float, y: float, r: float)` | Circle primitive |
| `arc_round_caps` | `arc_round_caps(x: float, y: float, radius: float, start_angle: float, end_angle: float, thickness: float)` | Arc with round caps |
| `arc_flat_caps` | `arc_flat_caps(x: float, y: float, radius: float, start_angle: float, end_angle: float, thickness: float)` | Arc with flat caps |
| `arc2` | `arc2(x: float, y: float, r: float, s: float, e: float) -> vec4` | Legacy/debug helper |
| `hline` | `hline(y: float, h: float)` | Horizontal line band |
| `box` | `box(x: float, y: float, w: float, h: float, r: float)` | Rounded box |
| `box_y` | `box_y(x: float, y: float, w: float, h: float, r_top: float, r_bottom: float)` | Different top/bottom corner radii |
| `box_x` | `box_x(x: float, y: float, w: float, h: float, r_left: float, r_right: float)` | Different left/right corner radii |
| `box_all` | `box_all(x: float, y: float, w: float, h: float, r_left_top: float, r_right_top: float, r_right_bottom: float, r_left_bottom: float)` | Per-corner radii |
| `rect` | `rect(x: float, y: float, w: float, h: float)` | Rectangle |
| `hexagon` | `hexagon(x: float, y: float, r: float)` | Hexagon primitive |
| `move_to` | `move_to(x: float, y: float)` | Path start |
| `line_to` | `line_to(x: float, y: float)` | Path segment |
| `close_path` | `close_path()` | Close current path |

Most shipped widget shaders start with:

```rust
let sdf = Sdf2d.viewport(self.pos * self.rect_size)
```

and then build shapes followed by `fill`, `stroke`, or `glow`.

## 14. Common Built-in Geometry PODs

These are the shipped vertex PODs from `draw/src/geometry/geometry_gen.rs`.

### 14.1 `geom.QuadVertex`

| Field | Type |
| --- | --- |
| `pos` | `vec2f` |

### 14.2 `geom.VectorVertex`

| Field | Type |
| --- | --- |
| `x` | `f32` |
| `y` | `f32` |
| `u` | `f32` |
| `v` | `f32` |
| `color_r` | `f32` |
| `color_g` | `f32` |
| `color_b` | `f32` |
| `color_a` | `f32` |
| `stroke_mult` | `f32` |
| `stroke_dist` | `f32` |
| `shape_id` | `f32` |
| `param0` | `f32` |
| `param1` | `f32` |
| `param2` | `f32` |
| `param3` | `f32` |
| `param4` | `f32` |
| `param5` | `f32` |
| `clip_radius` | `f32` |
| `zbias` | `f32` |

### 14.3 `geom.PbrVertex`

| Field | Type | Meaning |
| --- | --- | --- |
| `pos_nx` | `vec4f` | xyz + nx |
| `ny_nz_uv` | `vec4f` | ny nz u v |
| `color` | `vec4f` | rgba |
| `tangent` | `vec4f` | tangent xyz + handedness |

### 14.4 `geom.CubeVertex`

| Field | Type |
| --- | --- |
| `geom_pos` | `vec3f` |
| `geom_id` | `f32` |
| `geom_normal` | `vec3f` |
| `geom_pad` | `f32` |
| `geom_uv` | `vec2f` |
| `geom_tail_pad_0` | `f32` |
| `geom_tail_pad_1` | `f32` |

## 15. `mod.draw.DrawQuad`: Common Fields You Usually Have

If your shader inherits:

```rust
..mod.draw.DrawQuad
```

then these fields and helpers are part of `self`.

### 15.1 DrawQuad shader IO fields

| Field | Type | Kind |
| --- | --- | --- |
| `vertex_pos` | `vec4f` | `vertex_position` |
| `fb0` | `vec4f` | `fragment_output(0, vec4f)` |
| `draw_call` | `draw.DrawCallUniforms` | `uniform_buffer` |
| `draw_pass` | `draw.DrawPassUniforms` | `uniform_buffer` |
| `draw_list` | `draw.DrawListUniforms` | `uniform_buffer` |
| `geom` | `geom.QuadVertex` | `vertex_buffer` |
| `pos` | `vec2f` | `varying` |
| `world` | `vec4f` | `varying` |

### 15.2 DrawQuad live/runtime fields commonly read in shaders

| Field | Type |
| --- | --- |
| `rect_pos` | `vec2f` |
| `rect_size` | `vec2f` |
| `draw_clip` | `vec4f` |
| `depth_clip` | `f32` |
| `draw_depth` | `f32` |
| `pad1` | `f32` |
| `pad2` | `f32` |

### 15.3 DrawQuad helper methods

| Name | Signature | Notes |
| --- | --- | --- |
| `clip_and_transform_vertex` | `clip_and_transform_vertex(rect_pos: vec2, rect_size: vec2) -> vec4` | Clips quad vertex and builds clip-space position |
| `transform_vertex` | `transform_vertex(rect_pos: vec2, rect_size: vec2) -> vec4` | Same idea, without clip rectangle path |
| `vertex` | `vertex()` | Default entry point writes `vertex_pos` |
| `fragment` | `fragment()` | Default entry point writes `fb0 = pixel()` |
| `pixel` | `pixel() -> vec4` | Default returns transparent color |

### 15.4 Nested uniform-buffer field layouts

#### `draw.DrawCallUniforms`

| Field | Type |
| --- | --- |
| `zbias` | `f32` |
| `pad1` | `f32` |
| `pad2` | `f32` |
| `pad3` | `f32` |

#### `draw.DrawPassUniforms`

| Field | Type |
| --- | --- |
| `camera_projection` | `mat4x4f` |
| `camera_projection_r` | `mat4x4f` |
| `camera_view` | `mat4x4f` |
| `camera_view_r` | `mat4x4f` |
| `depth_projection` | `mat4x4f` |
| `depth_projection_r` | `mat4x4f` |
| `depth_view` | `mat4x4f` |
| `depth_view_r` | `mat4x4f` |
| `camera_inv` | `mat4x4f` |
| `dpi_factor` | `f32` |
| `dpi_dilate` | `f32` |
| `time` | `f32` |
| `pad2` | `f32` |

#### `draw.DrawListUniforms`

| Field | Type |
| --- | --- |
| `view_transform` | `mat4x4f` |
| `view_clip` | `vec4f` |
| `view_shift` | `vec2f` |
| `pad1` | `f32` |
| `pad2` | `f32` |

Practical examples:

```rust
let uv = self.pos
let world = self.world
let time = self.draw_pass.time
let z = self.draw_call.zbias
let shift = self.draw_list.view_shift
let geom_pos = self.geom.pos
```

## 16. `mod.draw.DrawText`: Common Extra Fields

If you inherit `mod.draw.DrawText`, these are the extra shader-visible pieces added on top of the general draw-stack behavior.

### 16.1 Extra shader object fields

| Field | Type | Kind |
| --- | --- | --- |
| `color` | `vec4f` | implicit uniform/live field |
| `sdf_sharpness` | `f32` | implicit instance/live field |
| `sdf_luma_bias` | `f32` | implicit instance/live field |
| `pos` | `vec2f` | varying |
| `t` | `vec2f` | varying |
| `world` | `vec4f` | varying |
| `radius` | `float` | uniform |
| `cutoff` | `float` | uniform |
| `total_chars` | `f32` | instance |
| `grayscale_texture` | `Texture2d` | texture |
| `color_texture` | `Texture2d` | texture |
| `msdf_texture` | `Texture2d` | texture |

### 16.2 Common extra per-instance/live fields used by the shipped shader

| Field | Type |
| --- | --- |
| `glyph_depth` | `f32` |
| `texture_index` | `f32` |
| `char_index` | `f32` |
| `t_min` | `vec2f` |
| `t_max` | `vec2f` |
| `atlas_plane` | `f32` |

### 16.3 Common helper methods

| Name | Signature |
| --- | --- |
| `sdf` | `sdf(scale: float, p: vec2, color: vec4) -> float` |
| `msdf` | `msdf(scale: float, p: vec2, color: vec4) -> float` |
| `get_color` | `get_color() -> vec4` |
| `sample_text_pixel` | `sample_text_pixel() -> vec4` |
| `vertex` | `vertex()` |
| `fragment` | `fragment()` |
| `pixel` | `pixel() -> vec4` |

## 17. Practical `pixel: fn()` Examples

Again: `pixel` is just a helper by convention. These examples assume your `fragment` function writes `self.fb0 = self.pixel()`, either because you inherited `DrawQuad` or because you wrote that yourself.

### 17.1 Minimal custom shader with explicit `fragment`

```rust
set_type_default() do #(MyShader::script_shader(vm)) {
    vertex_pos: vertex_position(vec4f)
    fb0: fragment_output(0, vec4f)
    geom: vertex_buffer(geom.QuadVertex, geom.QuadGeom)
    pos: varying(vec2f)

    vertex: fn() {
        self.pos = self.geom.pos
        self.vertex_pos = vec4(self.geom.pos * 2.0 - 1.0, 0.0, 1.0)
    }

    fragment: fn() {
        self.fb0 = self.pixel()
    }

    pixel: fn() -> vec4 {
        return vec4(self.pos.x, self.pos.y, 0.0, 1.0)
    }
}
```

### 17.2 Fullscreen-style procedural pixel shader

```rust
set_type_default() do #(DrawFullscreenShader::script_shader(vm)) {
    ..mod.draw.DrawQuad
    time: 0.0

    pixel: fn() -> vec4 {
        let uv = self.pos * 2.0 - vec2(1.0, 1.0)
        let aspect = self.rect_size.x / max(self.rect_size.y, 0.0001)
        let p = vec2(uv.x * aspect, uv.y)
        let t = self.time * 1.35
        let angle = atan2(p.y, p.x)
        let radius = length(p)
        let ripple = sin(radius * 12.0 - t * 6.0 + angle * 5.0)
        let bands = 0.5 + 0.5 * sin(angle * 6.0 - t * 4.0 + radius * 10.0)
        let target = 0.42 + 0.07 * sin(t + angle * 3.0 + ripple * 0.4)
        let ring = clamp(1.0 - abs(radius - target) * 18.0, 0.0, 1.0)
        let vignette = clamp(1.15 - radius * 0.75, 0.0, 1.0)
        let base = vec3(0.02, 0.04, 0.09)
            .mix(vec3(0.08, 0.34, 0.92), bands)
            .mix(vec3(1.05, 0.42, 0.16), ring * 0.9)
        return vec4(base * vignette + vec3(1.2, 0.5, 0.18) * ring, 1.0)
    }
}
```

This example uses:

- implicit `self.time`
- `self.pos` and `self.rect_size` from `DrawQuad`
- built-ins `atan2`, `length`, `sin`, `abs`, `clamp`
- method-sugar `vec3(...).mix(...)`

### 17.3 Rounded-rectangle SDF pixel shader

```rust
set_type_default() do #(MyPanelShader::script_shader(vm)) {
    ..mod.draw.DrawQuad
    radius: uniform(12.0)
    fill_color: instance(#x2ecc71)
    border_color: instance(#163621)
    border_width: uniform(2.0)

    pixel: fn() -> vec4 {
        let sdf = Sdf2d.viewport(self.pos * self.rect_size)
        sdf.box(0.0, 0.0, self.rect_size.x, self.rect_size.y, self.radius)
        sdf.fill_keep(self.fill_color)
        sdf.stroke(self.border_color, self.border_width)
        return sdf.result
    }
}
```

This example uses:

- `Sdf2d.viewport`
- `Sdf2d.box`
- `Sdf2d.fill_keep`
- `Sdf2d.stroke`
- explicit `uniform` and `instance` fields

### 17.4 Texture sampling pixel shader

```rust
set_type_default() do #(MyImageShader::script_shader(vm)) {
    ..mod.draw.DrawQuad
    tex: texture_2d(float)

    pixel: fn() -> vec4 {
        let tex_size = self.tex.size()
        let half_texel = vec2(0.5 / tex_size.x, 0.5 / tex_size.y)
        let uv = clamp(self.pos, half_texel, vec2(1.0, 1.0) - half_texel)
        let c = self.tex.sample_as_bgra(uv)
        return vec4(c.rgb * c.a, c.a)
    }
}
```

This example uses:

- texture field declaration
- `size()`
- `sample_as_bgra()`
- premultiplied-alpha output

## 18. Quick Rules of Thumb

- `vertex` and `fragment` are the only real entry points
- `pixel` is just a helper by convention
- Unmarked scalar fields become instances
- Unmarked vec/mat fields become uniforms
- `bool` fields are not inferred; mark them explicitly
- `self` can read your fields, inherited fields, scope uniforms, textures, and uniform buffers
- Only varyings / vertex position are writable in `vertex`
- Only fragment outputs are writable in `fragment`
- Vector swizzles support `xyzw` and `rgba`
- Arrays, vectors, and matrices are indexable with integer indices
- If you inherit `..mod.draw.DrawQuad`, you almost always use:
  - `self.pos`
  - `self.rect_pos`
  - `self.rect_size`
  - `self.draw_call.zbias`
  - `self.draw_pass.*`
  - `self.draw_list.*`
  - `self.geom.pos`
