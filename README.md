<!--
© 2024 Carl Åstholm
SPDX-License-Identifier: MIT
-->

# zigglgen

Fork for Linux that removes the ProcTable initialisation and instead just define the OpenGL functions as extern.

IMPORTANT difference is that you need to link libGL.so in build.so see the example below.

exe.root_module.linkSystemLibrary("GL", .{});

The only Zig OpenGL binding generator you need.

## Installation and usage

zigglgen currently supports the following versions of the Zig compiler:

- `0.14.0`
- `0.15.0-dev` (master)

Older or more recent versions of the compiler are not guaranteed to be compatible.

1\. Run `zig fetch` to add the zigglgen package to your `build.zig.zon` manifest:

```sh
zig fetch --save git+https://github.com/castholm/zigglgen.git
```

2\. Generate a set of OpenGL bindings in your `build.zig` build script:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const exe_mod = b.createModule(...);

    // Choose the OpenGL API, version, profile and extensions you want to generate bindings for.
    const gl_bindings = @import("zigglgen").generateBindingsModule(b, .{
        .api = .gl,
        .version = .@"4.1",
        .profile = .core,
        .extensions = &.{ .ARB_clip_control, .NV_scissor_exclusive },
    });

    // Import the generated module.
    exe_mod.addImport("gl", gl_bindings);
}
```

3\. Initialize OpenGL and start issuing commands:

```zig
const windowing = @import(...);
const gl = @import("gl");

pub fn main() !void {
    // Create an OpenGL context using a windowing system of your choice.
    const context = windowing.createContext(...);
    defer context.destroy();

    // Make the OpenGL context current on the calling thread.
    windowing.makeContextCurrent(context);
    defer windowing.makeContextCurrent(null);

    // Issue OpenGL commands to your heart's content!
    const alpha: gl.float = 1;
    gl.ClearColor(1, 1, 1, alpha);
    gl.Clear(gl.COLOR_BUFFER_BIT);
 }
```

See [castholm/zig-examples](https://github.com/castholm/zig-examples) for example projects.

## API

See [this gist](https://gist.github.com/castholm/a99b650cdd523244b456c7597e321e6e) for a preview of what the generated
code might look like.

### OpenGL symbols

zigglgen generates declarations for OpenGL functions, constants, types and extensions using the original names as
defined in the various OpenGL specifications (as opposed to the prefixed names used in C).

|           | C                     | Zig                |
|-----------|:----------------------|:-------------------|
| Command   | `glClearColor`        | `ClearColor`       |
| Constant  | `GL_TRIANGLES`        | `TRIANGLES`        |
| Type      | `GLfloat`             | `float`            |
| Extension | `GL_ARB_clip_control` | `ARB_clip_control` |

### `info`

```zig
pub const info = struct {};
```

Contains information about the generated set of OpenGL bindings, such as the OpenGL API, version and profile the
bindings were generated for.

## FAQ

### Which OpenGL APIs are supported?

Any APIs, versions, profiles and extensions included in Khronos's [OpenGL XML API
Registry](https://github.com/KhronosGroup/OpenGL-Registry/tree/main/xml) are supported. These include:

- OpenGL 1.0 through 3.1
- OpenGL 3.2 through 4.6 (Compatibility/Core profile)
- OpenGL ES 1.1 (Common/Common-Lite profile)
- OpenGL ES 2.0 through 3.2
- OpenGL SC 2.0

The [`updateApiRegistry.ps1`](updateApiRegistry.ps1) PowerShell script is used to fetch the API registry and convert it
to a set of Zig source files that are committed to revision control and used by zigglgen.

### Why aren't OpenGL constants represented as Zig enums?

The short answer is that it's simply not possible to represent groups of OpenGL constants as Zig enums in a
satisfying manner:

- The API registry currently specifies some of these groups, but far from all of them, and the groups are not guaranteed
  to be complete. Groups can be extended by extensions, so Zig enums would need to be defined as non-exhaustive, and
  using constants not specified as part of a group would require casting.
- Some commands like *GetIntegerv* that can return constants will return them as plain integers. Comparing the returned
  values against Zig enum fields would require casting.
- Some constants in the same group are aliases for the same value, which makes them impossible to represent as
  Zig enums.

### Why did calling a supported extension function result in a null pointer dereference?

Certain OpenGL extension add features that are only conditionally available under certain OpenGL versions/profiles or
when certain other extensions are also supported; for example, the *VertexWeighthNV* command from the *NV_half_float*
extension is only available when the *EXT_vertex_weighting* extension is also supported. Unfortunately, the API registry
does not specify these interactions in a consistent manner, so it's not possible for zigglgen to generate code that
ensures that calls to supported extension functions are always safe.

If you use OpenGL extensions it is your responsibility to read the extension specifications carefully and understand
under which conditions their features are available.

## Contributing

If you have any issues or suggestions, please open an issue or a pull request.

### Help us define overrides for function parameters and return types!

Due to the nature of the API Registry being designed for C, zigglgen currently generates most pointers types as `[*c]`
pointers, which is less than ideal. A long-term goal for zigglgen is for every single pointer type to be correctly
annotated. There are approximately 3300 commands defined in the API registry and if we work together, we can achieve
that goal sooner. Even fixing up just a few commands would mean a lot!

Overriding parameters/return types is very easy; all you need to do is add additional entries to the
`paramOverride`/`returnTypeOverride` functions in [`zigglgen.zig`](zigglgen.zig), then open a pull request with your
changes (bonus points if you also reference relevant OpenGL references page or specifications in the description of your
pull request).

## License

This repository is [REUSE-compliant](https://reuse.software/). The effective SPDX license expression for the repository as a whole is:

```
Apache-2.0 AND MIT
```

Copyright notices and license texts have been reproduced in [`LICENSE.txt`](LICENSE.txt), for your convenience.
