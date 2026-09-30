# Entity Imposter // WebGPU

Octahedral imposters for any 3D model or scene in WebGPU. Every model is baked once into an atlas of
N x N views; far instances are drawn as one camera-facing quad that rebuilds the model from those views,
with correct depth. A compute pass picks mesh or imposter per instance each frame.

The bake is **light agnostic**: it stores only the model's surface (a small G-buffer: albedo, normal,
depth, ambient occlusion, specular, gloss, translucency, emission), never lit colour. Imposters are lit
per frame by the same code as the meshes, so any lighting scenario, including one changed live, uses the
same atlases. Nothing about lighting is part of what a bake depends on (`ModelLibrary.load`'s key), and
the HUD counts atlas bakes after the first relight to show it stays at 0.

Everything is in one static file, `index.html`. Open it in a WebGPU browser (Chrome/Edge 113+, or Brave
with WebGPU enabled), straight from disk or served over HTTP:

```bash
python -m http.server 8767
```

The demo world ("Imposter Valley": ~15k trees, rocks and homesteads in a 3.2 km valley ringed by snowy
mountains) is a JSON block inside
`index.html` (`<script id="scenario" type="application/json">`). **Load** (or drop) a scenario `.json` to
replace it, and **Export scenario JSON** to get the current one as a file to edit.

Drop a **.glb**, **.gltf** (with its `.bin` and textures, or with embedded data) or **.obj** (with its
`.mtl` and textures) onto the page to bake that model: it replaces the mesh / imposter comparison pair
and a field of copies is scattered around the valley (see `drop` below).

## How the imposter works

**Bake** (`ImposterBaker`), once per model:

1. The model's bounding sphere (centre, radius R) is the imposter's volume.
2. For each frame (i, j) of an N x N grid, the direction is `octDecode((i, j) / (N - 1) * 2 - 1)`:
   - **hemi**: the upper hemisphere, the octahedron's top half rotated 45 degrees to fill the square
     (ground objects).
   - **full**: the whole sphere, the lower half folded into the corners (objects seen from any side, or
     rotated freely).
3. An orthographic camera looks at the sphere along the frame direction. It writes the unlit surface,
   one atlas per layer:

   | Layer | Format | Channels |
   |---|---|---|
   | albedo | `rgba8unorm-srgb` | albedo, coverage |
   | normal | `rgba8unorm` | object-space normal, depth along the frame direction |
   | surface | `rgba8unorm` | ambient occlusion, specular, gloss, translucency (wrap) |
   | emissive | `rgba8unorm-srgb` | emission / 8; only for models with emissive materials |

   All are 4x MSAA targets cleared to 0. Resolving them gives values premultiplied by coverage, so
   bilinear filtering and mips never bleed the background into the edges.
4. Each frame is resolved into a small cell texture and copied into its atlas cell. A box-filter mip
   chain follows, with cells kept at 8 px or more.

**Ambient occlusion** (`GpuMesh.bakeAO`, `WGSL_AO`) is geometry only, so it bakes with the mesh:

1. Render the model's depth from 48 directions around it.
2. A compute pass tests every vertex against those views, cosine-weighted around its normal.
3. The result is a per-vertex AO buffer. Meshes read it as a vertex attribute and the imposter bake
   writes it into the surface layer, so both darken the same creases and canopy interiors.

**Runtime** (`WGSL_IMPOSTER`), per instance:

1. **Vertex**:
   - A quad faces the camera and is sized to the silhouette cone of the bounding sphere, so it covers
     the object even up close.
   - The view direction in the instance's own space (so any rotation works) is octahedral-encoded
     into grid coordinates.
   - The three frames of the surrounding grid triangle and their barycentric weights are passed flat.
2. **Fragment**, for each of the three frames:
   1. Intersect the pixel's view ray with the frame's plane through the centre, giving a uv in that
      frame.
   2. **Parallax**: read the baked depth there, move along the ray to that depth and reproject.
   3. Sample albedo and normal / depth.

   Then blend the three frames by weight (the samples are premultiplied, so divide by the blended
   coverage) and alpha-test at 0.5 with sharpened alpha to coverage.
3. **Light**: the blended surface (normal rotated into world space, AO, specular, gloss, translucency,
   emission) goes through `shade()`, the same function the meshes use, with the current lighting:
   - sun direction, colour and intensity
   - sky / ground ambient
   - fog and exposure
   - emission scale ("lights": windows glow at night)
4. **Pixel depth offset**: the blended hit points give the surface position. Its depth is written, so
   imposters intersect the terrain and each other correctly.

**LOD and culling** (`WGSL_CULL`), one compute dispatch per model:

1. Pick mesh or imposter by camera distance. The switch distance is `lod.distance` per 5 m of bounding
   radius, times the model's `lodBias`.
2. If the camera frustum holds the instance's bounding sphere, append it to the camera's mesh list
   and/or imposter list. Inside the fade band it goes to both, and the two dither complementary pixels
   (a cross-fade without blending).
3. For each shadow cascade whose light box holds the sphere, append it to that cascade's caster list.
   This happens whether or not the camera sees it. The same LOD applies, with a hard switch at the
   middle of the fade band.
4. Count straight into the indirect draw arguments with atomics: one `drawIndirect` for the imposters and
   one `drawIndexedIndirect` per submesh. The first submesh's `instanceCount` counts the list, and the
   others follow it with `atomicMax`. The args are reset from a template by one queue write per frame, so
   there are no per-list copies or clears, and the CPU never touches per-instance data.

**Static archetypes** (terrain chunks: one forced-mesh instance that never moves) skip all of that. Their
bounding spheres are culled on the CPU against the camera frustum and each cascade box, and the visible
ones are drawn with plain `drawIndexed`. Browsers validate every indirect draw on the CPU, and 64 chunks ×
5 lists of indirect draws cost more than the whole GPU frame.

**Cascaded shadow maps** (`Renderer.writeShadows`, `shadowAt()`):

1. **Split**: the camera range up to `shadows.distance` is split into 4 cascades (a log / uniform mix
   set by `lambda`).
2. **Fit**: each slice's bounding sphere gets an orthographic light box, in one `depth32float` texture
   array layer. The box is snapped to whole texels so shadows do not shimmer as the camera moves, and
   stretched 800 m back toward the sun so casters outside the slice still land in it.
3. **Casters**: each cascade draws its own culled lists. Meshes use slope depth bias: cutout materials
   (leaves) go through an alpha-test fragment shader, opaque ones (terrain, bark, stone) through a
   depth-only pipeline. The terrain chunks cast too, so mountains throw their shadows across the valley. **Imposters cast their real shape**: from the cascade's light view the same vertex and
   `reconstruct()` code rebuilds the surface (3 frames, parallax), and `fsShadow` writes its depth. A
   tree's shadow therefore has its canopy's holes, not a quad. In the far cascades (from `CHEAP_CASCADE`,
   texels of 0.3 m and up) `fsShadowCheap` tests one frame's coverage at the quad's depth. Writing no
   depth there keeps early-z.
4. **Receivers**: meshes and imposters use the same `shade()`.
   - It picks the cascade by view depth.
   - It offsets the lookup along the normal and toward the light, scaled by that cascade's texel size.
   - It takes 3 x 3 PCF (comparison sampler) and fades out at the shadow distance.

   Imposters receive at their reconstructed surface points, so they also self-shadow.

The shadow maps are rebuilt every frame from the current sun, so they follow every lighting change
without any rebake.

## Layout (sections of the script in `index.html`)

```
config, math, noise, Oct          constants, vectors / matrices / quaternions, value noise, octahedral mapping
textures, Material                procedural Canvas 2D patterns, sRGB mip chains that keep alpha-test coverage
Geo, SHAPES, MeshBuilder          part shapes (box, cylinder, cone, sphere, knot, lathe, roof, foliage), merge by material
MeshData, ModelAsset, ModelLibrary  models -> meshes (nested models, `array` repeats), imports, asset lifetime
GltfLoader, ObjLoader             dropped files -> MeshData
WGSL_*                            sky, mesh, bake, mip, cull, imposter, atlas overlay
GpuMesh, ImposterAtlas, ImposterBaker, Renderer
Archetype                         instances of one model in a world: buffers, culling, indirect draws
Entity, Terrain, Prop, Compare, Scatter, ENTITY_TYPES, World
FlyCamera, InputSystem, Hud, ControlPanel, Game, main
```

## Scenario file

| Key | Contents |
|---|---|
| `name`, `camera` | title; start `pos` (y above the ground), `yaw`, `pitch` (degrees), `fov`, `speed` |
| `lighting` | named lighting scenarios: `sun` (`azimuth`, `elevation` in degrees, or `dir`; `color`, `intensity`), `sky` (`top`, `horizon`), `ambient` (`sky`, `ground`, `strength`), `fog` (`color`, `density`), `exposure`, `emission` (scale of emissive materials) |
| `environment` | the starting `lighting` scenario by name, or a scenario object |
| `lod` | `mode` (`auto`, `mesh`, `imposter`), `distance` (metres per 5 m of radius), `fade` (band as a fraction of the distance), `far` |
| `shadows` | `enabled`, `distance` (metres the cascades cover), `resolution` (per cascade), `lambda` (split: 0 uniform .. 1 logarithmic) |
| `imposter` | default bake settings: `grid` (frames per side), `res` (px per frame), `mode` (`hemi`, `full`), `ao` (strength 0..1); runtime switches `blend`, `parallax`, `depthOffset` |
| `materials` | procedural `pattern` (`flat`, `checker`, `grass`, `bark`, `birch`, `needles`, `leaves`, `rock`, `plaster`, `bricks`, `tiles`, `planks`, `window`, `metal`) drawn with `colors`, `seed`; `tint`, `uvScale`, `alphaCutoff`, `doubleSided`, `backfaceFlip`, `wrap` (soft terminator for foliage), `spec`, `gloss`, `emissive` { `color`, `strength` } |
| `models` | `parts` list; `imposter` (bake settings incl. `ao`, or `false` for none); `lodBias`; `castShadows` (default true; the terrain only receives) |
| `entities` | `{ type, id, ... }`, where `type` maps to a class in `ENTITY_TYPES` (see below) |
| `drop` | for dropped models: `fit` (largest side in metres), `imposter` (bake settings), `replace` (entity ids whose model becomes the dropped one), `field` (an extra entity using it) |

### Model parts

Every part has `shape`, optional `pos`, `rot` (degrees; a number is a yaw, `[x, y, z]` is applied as
Y·X·Z), `scale` (number or `[x, y, z]`), `mat`, `color` (tint), and `array` `{ count, pos, rot, scale }`.
With `array`, copy k is placed by that step applied k times, e.g. branches around a trunk or shrinking
fir tiers.

| `shape` | Parameters |
|---|---|
| `box` | `size` [x, y, z], centred |
| `cylinder` | `r` or `r0` / `r1`, `h` (from y = 0), `seg`, `caps`, `jag` (alternating bottom radius) |
| `cone` | `r`, `h`, `seg`, `jag` |
| `sphere` | `r`, `subdiv`, `noise`, `freq`, `seed` (displaced icosphere: rocks) |
| `knot` | `p`, `q`, `R`, `tube`, `seg`, `rad` (torus knot) |
| `lathe` | `profile` [[r, y], ...] from the bottom up, `seg` |
| `roof` | `size` [width, height, depth], gable ridge along x |
| `foliage` | `radius` [x, y, z], `count`, `card` (size), `seed`: leaf cards in an ellipsoid, normals bent outward |
| `model` | `model`: another model (scenario or imported) nested in place |

`model` parts are how a **scene** becomes one imposter: `homestead` is a cottage, two trees, a shed, a
fence and a rock, baked together.

### Entities

| `type` | Keys |
|---|---|
| `terrain` | `size`, `res` (grid cells per side), `chunks` (per side), `height` + `frequency` (fbm hills), `seed`, `flat` [cx, cz, r0, r1] (flattened valley), `mountains` { `center`, `inner`, `outer`, `height`, `frequency` } (ridged ring), `peaks` [[x, z, radius, height], ...], `layers` { `grass`, `dry`, `rock`, `snow` colours, `rockSlope` [from, to] (1 - normal.y), `snowLine` [from, to] (height) }, `mat` (a neutral detail texture the layer colours tint), `castShadows` |
| `prop` | `model`, `pos` (y above the ground unless `absolute`), `rot`, `scale`, `lod` (`mesh` / `imposter` forces it), `spin` { `axis`, `speed` deg/s }, `footprint` |
| `compare` | `model`, `pos`, `gap`: the model twice, mesh on the left and imposter on the right |
| `scatter` | `model`, `count`, `center` [x, z], `inner`, `outer`, `spacing` (radius), `scale` [min, max], `tilt`, `sink`, `maxSlope` (1 - normal.y), `maxHeight` (treeline), `seed` |

To add a kind of entity: subclass `Entity`, add instances in `spawn()` with
`world.archetype(model).add(pos, quat, scale, force)`, move them in `update(dt, t)` with
`archetype.set(...)`, and register the class in `ENTITY_TYPES`.

## Controls

| Input | Action |
|---|---|
| drag / WASD / Q E / Shift / wheel | look / move / down, up / fast / speed |
| `1` `2` `3` | LOD auto / all mesh / all imposter |
| `[` `]` | LOD distance |
| `B` `P` `O` `T` | frame blending, depth parallax, pixel depth offset, LOD tint + billboard outlines |
| `G` | shadows on / off |
| `K` | next lighting scenario (animated; never rebakes) |
| `V` `N` | atlas overlay (albedo / normal / depth / ao / surface / emissive, the three frames in use outlined), next model |
| `R` `L` `H` | reset camera, load a file, hide the UI |

The panel's **Shadows** section toggles shadows, sets their distance and tints the image by cascade.
Its **Lighting** section switches scenarios and moves the sun and the emission scale live.
Its **Bake** section re-bakes the selected model with another grid, frame resolution or mode.
The HUD lists instances, visible meshes / imposters, shadow casters (over all cascades), triangles and
atlas memory per model. Where the browser allows `timestamp-query`, it also shows GPU time per pass: cull,
each cascade, and the main pass.

## Limits

- glTF: triangle meshes, base colour factor / texture, metallic / roughness factors (as specular /
  gloss), emissive factor (+ `KHR_materials_emissive_strength`), alpha modes, vertex colours and the
  node hierarchy. No Draco / meshopt compression, skins or animation.
- Imposters are static: a baked model can move and rotate as a whole, but not animate.
- One shadow-casting light (the sun or moon). Ambient light is the sky / ground hemisphere, occluded by
  the baked AO. Emissive materials glow but do not light their surroundings.
- Within the LOD fade band an instance casts as mesh or imposter (hard switch), not dithered.
- The layers cost memory: 3 (or 4 with emission) RGBA8 atlases per model.
- Views between frames are blended, so close up a coarse grid ghosts. Raise `grid` for models that are
  seen close.
