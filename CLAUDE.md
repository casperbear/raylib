# casperbear/raylib fork

Fork of raysan5/raylib (`upstream` remote) used by MixMage (`D:\backup\projects\MixMage`).
Upstream is merged periodically via GitHub "Merge branch 'raysan5:master' into master".
After a merge, `src/raylib.h`, `src/raymath.h`, `src/rlgl.h` and the built `raylib.lib` are copied
into `MixMage/RayLib/include` and `MixMage/RayLib/lib`.

To see everything the fork changes: `git diff upstream/master master -- src`.

## Checklist after merging upstream

1. Check whether upstream changed the originals of the fork's copied functions (table below).
   `git diff <previous merged upstream commit> <new upstream commit> -- src/rlgl.h src/rtextures.c src/rmodels.c`
2. Grep for new positional `Texture` initializers, e.g. `(Texture2D){ id, 1, 1, 1, FORMAT }`.
   `slices` is deliberately the LAST field of `Texture` so such initializers keep working (slices = 0).
   Never move it before `format`.
3. Rebuild `raylib.lib`, copy the three headers and the lib into MixMage.

## Fork-only API (texture arrays and mipmap control)

`Texture.slices`: 0 = ordinary 2D texture, > 0 = `GL_TEXTURE_2D_ARRAY` with that many layers.
Requires GL 3.3 / ES 3; the functions warn and return 0 on GL 1.1 / 2.1.

| Fork function | File | Copied/derived from upstream |
|---|---|---|
| `rlEnableTextureArray(id)` / `rlDisableTextureArray()` | rlgl.h | `rlEnableTexture` / `rlDisableTexture` |
| `rlTextureArrayParameters(id, param, value)` | rlgl.h | `rlTextureParameters` |
| `rlLoadTextureArray(data[], w, h, format, slices)` | rlgl.h | `rlLoadTexture` (uncompressed only, level 0 only, NEAREST + REPEAT) |
| `rlLoadTextureArrayFromAtlas(data, w, h, format, columns, rows)` | rlgl.h | `rlLoadTexture`; slices read row by row, left to right, via `GL_UNPACK_ROW_LENGTH/SKIP_*` |
| `rlGenTextureArrayMipmaps(id, w, h, format, *mipmaps)` | rlgl.h | `rlGenTextureMipmaps` |
| `rlGenTextureMipmapsEx(..., mipmapsDesired)` | rlgl.h | `rlGenTextureMipmaps`, caps `GL_TEXTURE_MAX_LEVEL` |
| `rlGenTextureArrayMipmapsEx(..., mipmapsDesired)` | rlgl.h | `rlGenTextureMipmaps`, caps `GL_TEXTURE_MAX_LEVEL` |
| `ImageMipmapsEx(image, mipmapsDesired)` | rtextures.c | `ImageMipmaps`; first half of levels use `ImageResize`, the rest `ImageResizeNN` |
| `LoadTextureArrayFromImages(images, count)` | rtextures.c | `LoadTextureFromImage`; all images must match size/format/mipmaps |
| `LoadTextureArrayFromAtlasImage(atlas, columns, rows)` | rtextures.c | `LoadTextureFromImage`; width/height = one slice |
| `GenTextureArrayMipmaps`, `GenTextureMipmapsEx`, `GenTextureArrayMipmapsEx` | rtextures.c | `GenTextureMipmaps` |
| `SetTextureArrayFilter(texture, filter)` | rtextures.c | `SetTextureFilter` |
| `SetTextureArrayWrap(texture, wrap)` | rtextures.c | `SetTextureWrap` |

## Fork changes inside upstream functions / settings

- `DrawMesh`, `DrawMeshInstanced` (rmodels.c): bind a material map with `slices > 0` as a texture array.
  Convention: per-vertex slice in `texcoords2.x` (DrawMesh), per-instance slice in `transforms[i].m3` (DrawMeshInstanced).
- `CameraMoveToTarget` (rcamera.h): early return when `delta == 0`.
- `rlEnd` (rlgl.h): depth increment `1/10000` instead of `1/20000`.
- raymath.h: `RAYMATH_USE_SIMD_INTRINSICS` defaults to 1.
- config.h: `RL_CULL_DISTANCE_NEAR 0.125`, `RL_CULL_DISTANCE_FAR 2000.0`, `SUPPORT_SCREEN_CAPTURE 0` (no F12 screenshots).
- rlgl.h: commented-out `// #define RLGL_IMPLEMENTATION` for IntelliSense; must stay commented when building.
- external/stb_image.h: commented-out "dirty alpha" cleanup experiments in the PNG parser (inactive).
- raylib.h: `InitWindow` comment notes that 0 width/height opens fullscreen.

## Known quirks in fork code (not yet fixed)

- `rlEnableTextureArray` / `rlDisableTextureArray` contain `return 0;` in `void` functions (only in the GL 1.1/2.1 branch).
- rmodels.c calls `rlDisableTextureArray(id)` with an argument although it takes `void` (MSVC warning only).
- `ImageMipmapsEx` calls `TRACELOG` without a log level (upstream `ImageMipmaps` uses `TRACELOGD`).
- `rlTextureArrayParameters`: `RL_TEXTURE_MIPMAP_BIAS_RATIO` case has no `break` (falls into `default: break`, harmless).
- `LoadTextureArrayFromImages` ignores source mipmaps; uploads level 0 only.
