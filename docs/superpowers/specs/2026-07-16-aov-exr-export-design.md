# Design: AOV-to-EXR Export Application

## Purpose

New standalone executable `luisa-render-aov` that renders a 3D scene with all AOV (Arbitrary Output Variables) enabled and exports each AOV channel as a separate EXR file.

## Scope

Single new file `src/apps/aov_export.cpp` plus one-line addition to `src/apps/CMakeLists.txt`. No modifications to any other directory.

## Architecture

Follows the exact same pattern as `src/apps/cli.cpp`:

```
Parse CLI args → Create Context → Create Device → Create Stream
→ Load scene JSON → Override integrator to "AOV"
→ Write temp JSON → SceneParser::parse(temp) → Scene::create
→ Pipeline::create → pipeline->render(stream) → stream.synchronize()
```

The AOV export happens automatically during rendering because the overridden integrator (`AuxiliaryBufferPathTracing`) is configured with `dump: "final"` and all AOV components enabled. The integrator's built-in dump mechanism calls `save_image()` which writes EXR files via tinyexr.

## CLI Interface

```
luisa-render-aov -b <backend> [-d <device_index>] [-o <output_dir>] <scene_file> [-D <defines>...] [-v]
```

| Flag | Description | Default |
|------|-------------|---------|
| `-b/--backend` | Compute backend name (required) | — |
| `-d/--device` | Device index | `0` |
| `-o/--output` | Output directory for EXR files | Same directory as scene file |
| `-D/--define` | Scene macro definitions (`key=value`) | — |
| `-v/--verbose` | Enable verbose logging | `false` |
| `-h/--help` | Display help | — |
| positional | Path to scene description file (required) | — |

## Integrator Override Strategy

Before `SceneParser::parse()`, the app loads the scene JSON file, replaces/overrides the `render.integrator` section with:

```json
{
  "impl": "AOV",
  "prop": {
    "components": ["all"],
    "dump": "final"
  }
}
```

This ensures every scene (regardless of its original integrator) produces AOV exports.

The modified JSON is written to a temporary file (`{scene_name}.aov_temp.json`) in the same directory, which is then passed to `SceneParser`. The temp file is cleaned up after parsing.

## AOV Components Exported

All 10 consumer-facing components (out of 11 total):

| Component | Channels | Description |
|-----------|----------|-------------|
| `sample` | 3 (RGB) | Final beauty radiance |
| `diffuse` | 3 (RGB) | Diffuse component |
| `specular` | 3 (RGB) | Specular component |
| `normal` | 3 (XYZ) | World-space shading normal |
| `albedo` | 3 (RGB) | Surface albedo at first non-specular bounce |
| `depth` | 1 | Distance from camera to first hit |
| `roughness` | 2 (X,Y) | Surface roughness |
| `ndc` | 3 (XYZ) | Normalized device coordinates |
| `mask` | 1 | Pixel coverage mask |
| `variance` | 3 (RGB) | Per-pixel radiance variance |

`radiance_2` (sum of squared radiance, used internally for variance computation) is skipped.

## Output Files

Each AOV component → one EXR file per camera:

```
{output_dir}/{camera_filename_stem}_{component}.exr
```

Example for a camera with `file: "render.exr"`:
```
output/
  render_sample.exr
  render_diffuse.exr
  render_specular.exr
  render_normal.exr
  render_albedo.exr
  render_depth.exr
  render_roughness.exr
  render_ndc.exr
  render_mask.exr
  render_variance.exr
```

## Data Flow

```
Scene JSON (input, e.g. scene.json)
  │
  ├─ Read with nlohmann::json
  ├─ Modify render.integrator → AOV
  └─ Write temp file (scene.aov_temp.json)
         │
         ▼
    SceneParser::parse(temp_path, macros)
         │
         ▼
      SceneDesc
         │
         ▼
    Scene::create(context, scene_desc)
         │
         ▼
    Pipeline::create(device, stream, scene)
         │
         ▼
    pipeline->render(stream)
         │
         ▼
    AuxiliaryBufferPathTracing::Instance::render()
         ├─ For each camera:
         │    ├─ Accumulate AOV samples on GPU
         │    └─ On FINAL dump: save_image() → EXR
         │
         ▼
    stream.synchronize()
         │
         ▼
    EXR files written to output directory
```

## Build Integration

Add to `src/apps/CMakeLists.txt`:
```cmake
luisa_render_add_application(luisa-render-aov SOURCES aov_export.cpp)
```

## Limitations

- **Motion blur**: Not supported (same constraint as `AuxiliaryBufferPathTracing`)
- **Multi-camera**: All cameras in the scene are rendered; AOVs are named per-camera
- **Output naming**: Uses the camera's `file` stem from the scene description; if absent, falls back to `"render"`
- **Overwrites**: If EXR files already exist, they are silently overwritten
- **Temp file**: A temporary JSON file is created; guaranteed cleanup via `atexit` or scope guard

## Error Handling

- Invalid backend → exit with error message and help text
- Missing scene file → exit with error
- Scene parse failure → propagated from `SceneParser`
- Device creation failure → propagated from `Context`
- EXR write failure → logged as warning (non-fatal, from `save_image`)
