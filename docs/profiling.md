# Profiling a frame and finding leaks

Where a frame's time goes, stage by stage, and whether a scene holds more each time the
player enters and leaves a part of the game. The API is in `<scene/stats>` (imported by
`<scene>`): [API reference](api.md#scenestats).

## Frame stats

Profiling starts off and costs nothing then. `Scene_profile(true)` turns it on; close each
frame with `Scene_frame_end` and read it with `Scene_frame_stats`:

```eskiu
scene.profile(true);
...each frame:
scene.stage_begin(SCENE_STAGE_APP);
update_game(&game, dt);                              // your simulation
scene.stage_end(SCENE_STAGE_APP);
anims.update(&scene, now);                           // timed as ANIMATION
physics.update(&scene, now);                         // timed as PHYSICS
emitter.update(&scene, now)?;                        // timed as PARTICLES
scene.stage_begin(SCENE_STAGE_ACQUIRE);
GpuFrame frame = surface.frame()?;
scene.stage_end(SCENE_STAGE_ACQUIRE);
scene.render(frame.view, surface.width, surface.height, surface.format)?;
scene.stage_begin(SCENE_STAGE_PRESENT);
surface.present(&frame);
scene.stage_end(SCENE_STAGE_PRESENT);
scene.frame_end();

SceneFrameStats s = scene.frame_stats();             // the frame just closed
SceneStatsSummary sum = scene.stats_summary();       // average and max of the last 120
String report;
scene.stats_report(&report);                         // the summary as a table
```

`SceneFrameStats.cpu_ms[stage]` is the stage's own time in milliseconds. A stage begun
inside another is taken out of the outer one (the outer one pauses), so the stages add up
to `cpu_total_ms` without counting anything twice: wrap your whole update in
`SCENE_STAGE_APP` and the animation and physics calls inside it still show on their own.
`frame_ms` is the time from one `Scene_frame_end` to the next, the frame interval: 16.7 at
60 frames a second, 8.3 at 120.

| Stage | Timed by | What it includes |
|---|---|---|
| `SCENE_STAGE_APP` | you | your game code between `stage_begin` and `stage_end` |
| `SCENE_STAGE_ANIMATION` | `SceneAnimations_update` | property animations advanced and written to nodes |
| `SCENE_STAGE_PHYSICS` | `ScenePhysicsWorld_update` | world matrices, fixed physics steps, collisions, the bodies' nodes moved |
| `SCENE_STAGE_PARTICLES` | `SceneParticleEmitter_update` | spawning, moving and aging particles and rebuilding their quads (all emitters together) |
| `SCENE_STAGE_TRANSFORMS` | `Scene_render` | every node's world matrix (`Scene_update`), billboards |
| `SCENE_STAGE_CULL` | `Scene_render` | the camera, the draw list with frustum culling, the sort (opaque front to back, blended back to front), lights picked |
| `SCENE_STAGE_PREPARE` | `Scene_render` | shadow targets, the light and frame uniforms, each drawn node's uniforms (one buffer write per node), per-node and per-material GPU objects made or updated, pipelines made |
| `SCENE_STAGE_ENCODE` | `Scene_render` | the shadow, scene, portal, bloom and post-processing passes recorded |
| `SCENE_STAGE_SUBMIT` | `Scene_render` | the commands finished, validated and sent to the queue |
| `SCENE_STAGE_ACQUIRE` | you | `surface.frame()`: the wait for the next drawable |
| `SCENE_STAGE_PRESENT` | you | `surface.present()` |

With `Scene_render_in` (your own encoder, an interface drawn over the scene) time your
`enc.submit()` with `SCENE_STAGE_SUBMIT` yourself. wgpu-native validates the commands and
turns them into Metal, Vulkan or DirectX 12 commands as passes end and at submit, so part
of the driver's work shows in `encode` and most of it in `submit`.

The counters next to the times: `draws` (draw calls of meshes, shadow maps included),
`culled` (meshes outside the camera's view, not drawn in the scene pass), `shadow_maps`
(shadow passes) and `pipelines_made`. A pipeline is made, and its shader compiled, the
first time a material variant is drawn: a frame with `pipelines_made > 0` that is slow is a
shader compile, not steady-state cost. Draw everything once behind a loading screen to
make them early.

`SceneProfiler.window` sets how many frames the summary covers (120; at most 1000).
`scene_clock_ms()` is the clock the stages use, for your own measurements.

## GPU time

When the device has timestamp queries (`Gpu.timestamps`; wgpu-native has them on Metal,
Vulkan and DirectX 12), `Scene_profile(true)` also makes a `GpuTimer` and each
`Scene_render` has the GPU write its clock at the start and end of every pass
([gpu's guide](https://github.com/eskiu-packages/gpu/blob/develop/docs/profiling.md)).
`gpu_ms` is the time from the start of the frame's first pass to the end of its last.
It comes back without waiting for the GPU, so it belongs to an earlier frame (`gpu_frame`
counts the timed renders; one to three frames back). Without timestamp queries (WebGL2, a
driver that lacks them) `gpu_ms` is -1 and `SceneProfiler.gpu_note` says why: the GPU time
is reported as unavailable, never estimated. Rendering twice in a frame (two eyes) leaves
the last render's time in `gpu_ms`.

Reading the numbers together:

- `frame_ms` near the display's interval and `acquire` large: the frame waits for the
  display or for the GPU. If `gpu_ms` is near the interval too, the GPU is the limit;
  if it is well below, the app is simply as fast as the display (vsync).
- `cpu_total_ms` near `frame_ms` and above the interval: the CPU is the limit; the
  largest stage says where.
- `gpu_ms` grows with the drawable's size: fill rate (resolution, bloom, MSAA, overdraw of
  blended surfaces). It grows with `draws`: vertex work or many small draws.

## The reference scene

[`tests/profiling.esk`](../tests/profiling.esk) is a fixed workload: 400 boxes and spheres
(eight materials, a quarter of them spinning), 40 balls falling in the physics world, a
particle fountain at 100 particles a second, a sun with a shadow map, bloom and Hable tone
mapping, 1280 x 720, offscreen (so `acquire` and `present` are 0), with about 0.16 ms of
simulated app work. It renders 30 frames to warm up and prints the summary of the next
120. On an Apple M4 Pro (macOS, Metal):

```
120 frames          avg ms    max ms
  app               0.159     0.231
  animation         0.022     0.039
  physics           1.018     1.298
  particles         0.384     0.554
  transforms        0.099     0.125
  cull              0.214     0.303
  prepare           2.111     2.315
  encode            0.159     0.258
  submit            1.841     2.242
  acquire           0.000     0.000
  present           0.000     0.000
  cpu total         6.007     7.010
  frame interval    6.091     7.020
  gpu               1.938     4.419  (120 frames timed)
  draws 859 (max 859), culled 24, shadow maps 1, pipelines made 0 (max 0 in a frame)
```

What it shows: for about 440 drawn nodes, `prepare` (one uniform write per node and per
material each frame) and `submit` (validation and the Metal commands for some 860 draws)
cost more than the GPU's own work. Fewer, larger meshes and `Scene_set_instances` for
repeated objects cut both. Physics at 1 ms for 40 bodies is the next largest. Run it with
`eskiuc test profiling` and read `build/tests/profiling.log`.

## The cost of measuring

Off, each stage call is one test of a flag. On, each stage reads the clock twice (about 20
reads a frame), `Scene_frame_end` adds a few
dozen arithmetic operations and keeps 120 frames of history (about 20 KB), and the GPU
timer adds one resolve and one 16-bytes-per-pass copy per frame, a buffer map, a query
set and four small buffers. In the reference scene, 60 frames with profiling off took 5.4
ms a frame and 60 with it on 4.9 ms: the difference is within the run-to-run variation, so
the cost is below what this measures.

## Counting what a scene holds

`Scene_counts` returns a `SceneCounts`: the live nodes, meshes with GPU buffers (and their
bytes), materials with GPU objects, textures (and their bytes), lights, shaders, the
pipeline cache, the nodes with their own GPU uniforms, and `gpu`, the process's live GPU
objects from `gpu_counts()` (buffers, textures, pipelines, bind groups, samplers, with
bytes; the renderer's own targets and your app's objects included).

The `*_slots` fields are the ids handed out. Ids are never reused: a removed node keeps its
id, and a freed mesh, material or texture its id, so these grow each time something is
added, by design. Each costs a few hundred bytes of CPU memory and nothing on the GPU.

### Detecting growth

Take a baseline once the thing under test has run once (caches such as pipelines and
samplers fill the first time), repeat it, and compare:

```eskiu
enter_and_leave(&scene)?;                 // warm up the caches
SceneCounts base = scene.counts();
for (i in 0..10) enter_and_leave(&scene)?;
SceneCounts now = scene.counts();
String grew;
int n = now.growth(&base, &grew);          // the counts that went up, one line each
if (n > 0) printf("%s", grew.data);
```

Anything but the `*_slots` growing by the same amount each round is a leak: what grows
names it. Read the counts between frames, not in the middle of one. The reference test
does this for a sub-scene made as a separate `Scene` (made, drawn, released: nothing may
grow) and for a subtree of the main scene with its own particle emitter (added, drawn,
removed: only node, mesh and material slots may grow). It fails when anything else grows.

### Releasing what a part of the game added

`Scene_remove` removes nodes and releases their GPU uniforms, but meshes, materials and
textures are the scene's and stay until released:

- `Scene_free_mesh(id)` releases a mesh's buffers (a node still holding it draws nothing),
- `Scene_free_material(id)` releases a material's GPU objects (they are made again if a
  node draws with it),
- `Scene_free_texture(id)` releases a texture (materials using it show white),
- `SceneParticleEmitter_free` releases its emitter's mesh and material.

For a glTF model added with `Scene_add_gltf`, free each id in its `SceneGltf.meshes`,
`materials` and `textures` (skipping 0), `Scene_remove` its `root`, then `SceneGltf_free`.
A sub-scene that is a `Scene` of its own releases everything with `Scene_release`.

## On an iPhone

The stats above work the same on iOS. These Xcode tools show what they cannot. None of
this was verified on a device for this guide: the numbers here come from a Mac.

- **Time Profiler** (Instruments): samples the CPU. Eskiu functions keep their names
  (`Scene_render_in`, `_scene_collect`, `ScenePhysicsWorld_update`) unless the binary is
  stripped; without debug information expect function names but no source lines.
  Use it when one stage is large and you need the function inside it.
- **Metal System Trace** (Instruments): the GPU's timeline (vertex and fragment work per
  command buffer), the drawable's life (how long `acquire` waited and why), the CPU threads
  and the display's refresh. It shows whether a frame missed its deadline on the CPU or the
  GPU.
- **GPU frame capture** (Xcode: Debug > Capture GPU Workload while the app runs from
  Xcode): every pass and draw of one frame, its shaders, textures and buffers, the
  GPU time per pass and the memory each resource takes. It checks `texture_bytes` against
  what Metal really allocated (the driver pads and compresses).
- **Memory**: Xcode's memory gauge and the Memory Graph for the process's footprint;
  `ui agent ios perf` (eskiu-packages/ui) reports it from a running app.

## Telling causes apart

- **Your code**: `app` (and `animation`, `physics`, `particles` when you drive them).
  Time Profiler inside `app` when it is large.
- **Assets**: `texture_bytes` and `mesh_bytes` in `Scene_counts`, `draws`,
  `pipelines_made` spikes (shader compiles on first use), and a `gpu_ms` that drops when
  you render at a smaller size or turn bloom and MSAA off (fill rate).
- **The SDK**: `transforms`, `cull`, `prepare`, `encode`. They grow with the number of
  live nodes and drawn meshes, not with the screen's size.
- **The driver**: `submit`, `acquire` and `present`, and what Metal System Trace shows
  between the command buffer's commit and the GPU starting it.
- **Thermal throttling**: run a fixed workload (a fixed camera path, a fixed number of
  enemies) for several minutes and log `stats_summary()` every 120 frames. With the same
  work, stages that grow over time while the counts stay flat mean the device slowed down:
  the CPU stages rise when the CPU is throttled, `gpu_ms` when the GPU is. Instruments'
  thermal state (`ProcessInfo.thermalState` in a host app, or the thermal track
  Instruments can record) confirms it. Compare measurements only at the same thermal
  state, and let the device cool between runs.
