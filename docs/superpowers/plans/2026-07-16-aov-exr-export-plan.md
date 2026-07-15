# AOV-to-EXR Export Application Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `luisa-render-aov` executable that renders a scene with all AOV components enabled and exports each as a separate EXR file.

**Architecture:** Single new file `src/apps/aov_export.cpp` following the pattern of `src/apps/cli.cpp`. The app loads the scene JSON, overrides the integrator to `"AOV"` with all components and `dump: "final"`, writes a temp JSON, then runs the standard parse→scene→pipeline→render flow. The existing `AuxiliaryBufferPathTracing` integrator handles AOV accumulation and EXR output via `save_image()`.

**Tech Stack:** C++20, nlohmann/json, cxxopts, luisa::render, tinyexr (indirectly via save_image)

**Spec:** `docs/superpowers/specs/2026-07-16-aov-exr-export-design.md`

---

## File Map

| File | Action | Purpose |
|------|--------|---------|
| `src/apps/aov_export.cpp` | Create | Main application source |
| `src/apps/CMakeLists.txt` | Modify | Register new executable |

---

### Task 1: Register the new application in CMake

**Files:**
- Modify: `src/apps/CMakeLists.txt`

- [ ] **Step 1: Add the executable registration**

Edit `src/apps/CMakeLists.txt` — add one line after the existing `luisa-render-export` entry:

```cmake
luisa_render_add_application(luisa-render-aov SOURCES aov_export.cpp)
```

Full file after edit:

```cmake
function(luisa_render_add_application name)
    cmake_parse_arguments(APP "" "" "SOURCES" ${ARGN})
    add_executable(${name} ${APP_SOURCES})
    target_link_libraries(${name} PRIVATE luisa::render)
    install(TARGETS ${name}
            LIBRARY DESTINATION ${CMAKE_INSTALL_BINDIR}
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endfunction()

luisa_render_add_application(luisa-render-cli SOURCES cli.cpp)
luisa_render_add_application(luisa-render-export SOURCES export.cpp)
luisa_render_add_application(luisa-render-aov SOURCES aov_export.cpp)
```

- [ ] **Step 2: Verify CMake configuration**

```bash
cmake --build build --target help 2>&1 | Select-String "luisa-render-aov"
```

Expected: `luisa-render-aov` appears in the target list.

- [ ] **Step 3: Commit**

```bash
git add src/apps/CMakeLists.txt
git commit -m "build: register luisa-render-aov executable"
```

---

### Task 2: Create the application source file

**Files:**
- Create: `src/apps/aov_export.cpp`

- [ ] **Step 1: Write the complete source file**

Create `src/apps/aov_export.cpp`:

```cpp
//
// AOV-to-EXR exporter for LuisaRender.
// Renders a scene with all AOV components enabled
// and exports each as a separate EXR file.
//

#include <span>
#include <iostream>
#include <fstream>
#include <cstdio>

#include <cxxopts.hpp>
#include <nlohmann/json.hpp>

#include <core/stl/format.h>
#include <core/logging.h>
#include <sdl/scene_desc.h>
#include <sdl/scene_parser.h>
#include <base/scene.h>
#include <base/pipeline.h>

#if defined(LUISA_PLATFORM_WINDOWS)
#include <windows.h>
[[nodiscard]] auto get_current_exe_path() noexcept {
    constexpr auto max_path_length = std::max<size_t>(MAX_PATH, 4096);
    std::filesystem::path::value_type path[max_path_length] = {};
    auto nchar = GetModuleFileNameW(nullptr, path, max_path_length);
    if (nchar == 0 ||
        (nchar == MAX_PATH &&
         ((GetLastError() == ERROR_INSUFFICIENT_BUFFER) ||
          (path[MAX_PATH - 1] != 0)))) {
        LUISA_ERROR_WITH_LOCATION("Failed to get current executable path.");
    }
    return std::filesystem::canonical(path).string();
}
#elif defined(LUISA_PLATFORM_APPLE)
#include <libproc.h>
#include <unistd.h>
[[nodiscard]] auto get_current_exe_path() noexcept {
    char pathbuf[PROC_PIDPATHINFO_MAXSIZE] = {};
    auto pid = getpid();
    if (auto size = proc_pidpath(pid, pathbuf, sizeof(pathbuf)); size > 0) {
        luisa::string_view path{pathbuf, static_cast<size_t>(size)};
        return std::filesystem::canonical(path).string();
    }
    LUISA_ERROR_WITH_LOCATION(
        "Failed to get current executable path (PID = {}): {}.",
        pid, strerror(errno));
}
#else
#include <unistd.h>
[[nodiscard]] auto get_current_exe_path() noexcept {
    char pathbuf[PATH_MAX] = {};
    for (auto p : {"/proc/self/exe", "/proc/curproc/file", "/proc/self/path/a.out"}) {
        if (auto size = readlink(p, pathbuf, sizeof(pathbuf)); size > 0) {
            luisa::string_view path{pathbuf, static_cast<size_t>(size)};
            return std::filesystem::canonical(path).string();
        }
    }
    LUISA_ERROR_WITH_LOCATION(
        "Failed to get current executable path.");
}
#endif

using namespace luisa;
using namespace luisa::compute;
using namespace luisa::render;
using nlohmann::json;

namespace {

[[nodiscard]] auto parse_cli_options(int argc, const char *const *argv,
                                     const std::string &backend_hints) noexcept {
    cxxopts::Options cli{"luisa-render-aov"};
    cli.add_option("", "b", "backend", backend_hints,
                   cxxopts::value<luisa::string>(), "<backend>");
    cli.add_option("", "d", "device", "Compute device index",
                   cxxopts::value<std::size_t>()->default_value("0"), "<index>");
    cli.add_option("", "o", "output", "Output directory for EXR files",
                   cxxopts::value<std::filesystem::path>(), "<dir>");
    cli.add_option("", "", "scene", "Path to scene description file",
                   cxxopts::value<std::filesystem::path>(), "<file>");
    cli.add_option("", "D", "define",
                   "Parameter definitions to override scene description macros.",
                   cxxopts::value<std::vector<luisa::string>>()->default_value("<none>"),
                   "<key>=<value>");
    cli.add_option("", "h", "help", "Display this help message",
                   cxxopts::value<bool>()->default_value("false"), "");
    cli.add_option("", "v", "verbose", "Enable verbose logging",
                   cxxopts::value<bool>()->default_value("false"), "");
    cli.allow_unrecognised_options();
    cli.positional_help("<file>");
    cli.parse_positional("scene");
    auto options = [&] {
        try {
            return cli.parse(argc, argv);
        } catch (const std::exception &e) {
            LUISA_WARNING_WITH_LOCATION(
                "Failed to parse command line arguments: {}.",
                e.what());
            std::cout << cli.help() << std::endl;
            exit(-1);
        }
    }();
    if (options["help"].as<bool>()) {
        std::cout << cli.help() << std::endl;
        exit(0);
    }
    if (options["backend"].count() == 0) [[unlikely]] {
        LUISA_WARNING_WITH_LOCATION("Backend not specified.");
        std::cout << cli.help() << std::endl;
        exit(-1);
    }
    if (options["scene"].count() == 0u) [[unlikely]] {
        LUISA_WARNING_WITH_LOCATION("Scene file not specified.");
        std::cout << cli.help() << std::endl;
        exit(-1);
    }
    if (auto unknown = options.unmatched(); !unknown.empty()) [[unlikely]] {
        luisa::string opts{unknown.front()};
        for (auto &&u : luisa::span{unknown}.subspan(1)) {
            opts.append("; ").append(u);
        }
        LUISA_WARNING_WITH_LOCATION(
            "Unrecognized options: {}", opts);
    }
    return options;
}

[[nodiscard]] auto parse_cli_macros(int &argc, char *argv[]) {
    SceneParser::MacroMap macros;
    auto parse_macro = [&macros](luisa::string_view d) noexcept {
        if (auto p = d.find('='); p == luisa::string::npos) [[unlikely]] {
            LUISA_WARNING_WITH_LOCATION(
                "Invalid definition: {}", d);
        } else {
            auto key = d.substr(0, p);
            auto value = d.substr(p + 1);
            LUISA_VERBOSE_WITH_LOCATION("Parameter definition: {} = '{}'", key, value);
            if (auto iter = macros.find(key); iter != macros.end()) {
                LUISA_WARNING_WITH_LOCATION(
                    "Duplicate definition: {} = '{}'. "
                    "Ignoring the previous one: {} = '{}'.",
                    key, value, key, iter->second);
                iter->second = value;
            } else {
                macros.emplace(key, value);
            }
        }
    };
    for (int i = 1; i < argc; i++) {
        auto arg = luisa::string_view{argv[i]};
        if (arg == "-D" || arg == "--define") {
            if (i + 1 == argc) {
                LUISA_WARNING_WITH_LOCATION(
                    "Missing definition after {}.", arg);
                argv[i] = nullptr;
            } else {
                parse_macro(argv[i + 1]);
                argv[i] = nullptr;
                argv[++i] = nullptr;
            }
        } else if (arg.starts_with("-D")) {
            parse_macro(arg.substr(2));
            argv[i] = nullptr;
        }
    }
    auto new_end = std::remove(argv, argv + argc, nullptr);
    argc = static_cast<int>(new_end - argv);
    return macros;
}

// RAII temp file cleanup
struct TempFile {
    std::filesystem::path path;
    ~TempFile() noexcept {
        std::error_code ec;
        std::filesystem::remove(path, ec);
    }
};

// Load scene JSON, override integrator to AOV with all components,
// and optionally redirect camera output paths.
[[nodiscard]] auto prepare_scene_json(
    const std::filesystem::path &scene_path,
    const std::filesystem::path *output_dir) -> TempFile {

    // Read original scene JSON
    std::ifstream input{scene_path};
    LUISA_ASSERT(input.is_open(),
                 "Failed to open scene file '{}'.",
                 scene_path.string());
    json scene_json;
    input >> scene_json;
    input.close();

    // Override integrator in the render section
    if (!scene_json.contains("render")) {
        scene_json["render"] = json::object();
    }
    scene_json["render"]["integrator"] = {
        {"impl", "AOV"},
        {"prop", {
            {"components", {"all"}},
            {"dump", "final"}
        }}
    };
    LUISA_INFO("Injected AOV integrator with all components and dump=final.");

    // If output directory is specified, redirect all camera file paths
    if (output_dir != nullptr) {
        std::filesystem::create_directories(*output_dir);
        for (auto &[key, node] : scene_json.items()) {
            if (node.is_object() && node.value("type", "") == "Camera") {
                if (node.contains("prop") && node["prop"].contains("file")) {
                    auto original_file = node["prop"]["file"].get<std::string>();
                    std::filesystem::path cam_path{original_file};
                    auto new_path = *output_dir / cam_path.filename();
                    node["prop"]["file"] = new_path.string();
                    LUISA_INFO("Redirected camera '{}' output: '{}' -> '{}'.",
                               key, original_file, new_path.string());
                }
            }
        }
    }

    // Write temp file next to the original scene
    auto temp_path = scene_path.parent_path() /
                     (scene_path.stem().string() + ".aov_temp.json");
    std::ofstream output{temp_path};
    LUISA_ASSERT(output.is_open(),
                 "Failed to write temp scene file '{}'.",
                 temp_path.string());
    output << scene_json.dump(4);
    output.close();
    LUISA_INFO("Wrote AOV-configured scene to '{}'.", temp_path.string());

    return TempFile{temp_path};
}

} // anonymous namespace

int main(int argc, char *argv[]) {

    log_level_info();

    auto exe_path = get_current_exe_path();
    luisa::compute::Context context{exe_path};
    auto macros = parse_cli_macros(argc, argv);
    for (auto &&[k, v] : macros) {
        LUISA_INFO("Found CLI Macro: {} = {}", k, v);
    }

    auto backend_hints = fmt::format(
        "Compute backend name (possible values: {})",
        fmt::join(context.installed_backends(), ", "));
    auto options = parse_cli_options(argc, argv, backend_hints);
    if (options["verbose"].as<bool>()) { log_level_verbose(); }

    auto backend = options["backend"].as<luisa::string>();
    auto index = options["device"].as<std::size_t>();
    auto scene_path = options["scene"].as<std::filesystem::path>();

    // Resolve output directory
    luisa::optional<std::filesystem::path> output_dir;
    if (options["output"].count() > 0u) {
        output_dir = std::filesystem::absolute(options["output"].as<std::filesystem::path>());
        LUISA_INFO("Output directory: '{}'.", output_dir->string());
    }

    // Prepare scene JSON with AOV integrator
    auto temp_file = prepare_scene_json(
        scene_path, output_dir ? &*output_dir : nullptr);

    // Standard render pipeline (same as cli.cpp)
    compute::DeviceConfig config{
        .device_index = index,
        .inqueue_buffer_limit = false
    };
    auto device = context.create_device(backend, &config);

    Clock clock;
    auto scene_desc = SceneParser::parse(temp_file.path, macros);
    auto parse_time = clock.toc();

    LUISA_INFO("Parsed scene description file '{}' in {} ms.",
               scene_path.string(), parse_time);
    auto scene = Scene::create(context, scene_desc.get());
    auto stream = device.create_stream(StreamTag::GRAPHICS);
    auto pipeline = Pipeline::create(device, stream, *scene);
    pipeline->render(stream);
    stream.synchronize();

    LUISA_INFO("AOV export complete. EXR files written.");
}
```

- [ ] **Step 2: Build the application**

```bash
cmake --build build --target luisa-render-aov --config Release
```

Expected: Build succeeds with no errors.

- [ ] **Step 3: Verify the executable exists**

```bash
Get-ChildItem build -Recurse -Filter "luisa-render-aov*" | Select-Object FullName
```

Expected: The executable file is found.

- [ ] **Step 4: Test help output**

```bash
./build/bin/luisa-render-aov --help
```

Expected: Help text shows options including `-b`, `-d`, `-o`, `-D`, `-v`, `-h`, and positional `<file>`.

- [ ] **Step 5: Commit**

```bash
git add src/apps/aov_export.cpp
git commit -m "feat: add luisa-render-aov application for AOV-to-EXR export"
```

---

## Verification (Manual Integration Test)

After both tasks are complete, optionally verify end-to-end with a scene file:

```bash
./build/bin/luisa-render-aov -b cuda -o ./aov_output ./path/to/scene.json
```

Expected:
- Scene renders to completion
- `./aov_output/` contains EXR files like `{name}_sample.exr`, `{name}_diffuse.exr`, etc.
- Each EXR file is valid and viewable
