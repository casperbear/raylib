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
   Claude's build (2026-10-08): `MSBuild.exe projects\VS2022\raylib\raylib.vcxproj /p:Configuration=Release
   /p:Platform=x64` (VS2019 MSBuild); built alone, the project writes to `projects\VS2022\raylib\build\raylib\bin\x64\Release`
   (gitignored; the human's .sln build uses `projects\VS2022\build`). Runtime: /MT (LIBCMT), as before.

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

## Fork-only API (kerning, 2026-10-08)

`Font.kerningCount` / `Font.kernings` (`KerningPair { first, second, amount }`: glyph indices into `Font.glyphs`, whole
pixels at `baseSize`, sorted by first then second): the LAST fields of `Font`, so positional initializers keep working.
Filled in `LoadFontFromMemory` (TTF/OTF only) by static `LoadFontKerningData` (rtext.c): glyphCount^2 lookups through
`stbtt_GetGlyphKernAdvance` (~1 ms for ASCII; skipped above `FONT_KERNING_MAX_GLYPHS` 1024 or without kern / GPOS
tables), amounts rounded, zero pairs dropped. `GetGlyphKerning(font, codepoint, nextCodepoint)` (public) and static
`GetKerningByIndex` (binary search). Not exported by `ExportFontAsCode`; BMFont / image / BDF / default fonts have none.

## Fork changes inside upstream functions / settings

- Kerning (rtext.c unless noted): `LoadFontFromMemory` (calls `LoadFontKerningData`, its TRACELOG line shows the pair
  count), `UnloadFont` (frees `kernings`), `DrawTextEx`, `DrawTextCodepoints`, `MeasureTextEx`, `MeasureTextCodepoints`
  (a `previousIndex` per line adds the pair amount before each glyph, reset at '\n'), `ImageTextEx` (rtextures.c, via
  `GetGlyphKerning`, must stay in step with `MeasureTextEx`, which sizes its image). After an upstream merge, check
  these functions for upstream changes and for new text loops over `advanceX` that should kern too.
- `LoadFontData` (rtext.c): glyph `advanceX` rounded to the nearest pixel (`+ 0.5f`, both branches) instead of upstream's
  truncation (2026-10-08; upstream master still truncates). Measured on a 44-character pangram: truncation made it 18-30
  px narrower than the exact sum (7-12% at sizes 16-20), rounding within +-5 px.

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
