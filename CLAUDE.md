# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **OpenXR SDK Source** repository by the Khronos Group — the canonical source tree for the OpenXR loader, API layers, samples, and tests. The consumer-facing `OpenXR-SDK` repo is a stripped-down derivative with pre-generated files.

- **Language**: C++17 (some C for headers and utilities)
- **Build system**: CMake 3.16+
- **Test framework**: Catch2 (vendored in `src/external/catch2`)

## Build Commands

### Configure and build (Windows / Visual Studio)

```cmd
mkdir build\win64 && cd build\win64
cmake -G "Visual Studio 17" -A x64 ..\..
cmake --build . --config RelWithDebInfo --parallel
```

For a DLL loader (default is static on Windows): add `-DDYNAMIC_LOADER=ON`.

### Configure and build (Linux)

```sh
mkdir -p build/linux_debug && cd build/linux_debug
cmake -DCMAKE_BUILD_TYPE=Debug ../..
make -j$(nproc)
```

### Run tests

```sh
ctest --test-dir build/linux_debug
# Run a single test:
ctest --test-dir build/linux_debug -R loader_test
```

### Run the hello_xr sample

The binary is at `<build_dir>/src/tests/hello_xr/hello_xr`. An OpenXR runtime must be installed, or point `XR_RUNTIME_JSON` to a runtime manifest JSON.

## Code Generation

Python 3.6+ with Jinja2 is required to generate C++ source from `specification/registry/xr.xml`. Many target repos ship pre-generated files so Python is optional unless you set `-DBUILD_FORCE_GENERATION=ON`.

The main code generation entry point is `src/scripts/src_genxr.py`, which uses template scripts in `src/scripts/` (e.g., `loader_source_generator.py`, `utility_source_generator.py`, `validation_layer_generator.py`).

## Key CMake Options

| Option | Default | Purpose |
|--------|---------|---------|
| `BUILD_LOADER` | ON | Build the OpenXR loader |
| `BUILD_API_LAYERS` | ON | Build API layers (api_dump, core_validation) |
| `BUILD_TESTS` | ON | Build test targets |
| `BUILD_SDK_TESTS` | ON | Build hello_xr and other SDK samples |
| `DYNAMIC_LOADER` | OFF on Win, ON elsewhere | Build loader as shared library |
| `BUILD_ALL_EXTENSIONS` | OFF | Fail if any graphics API is not found |
| `BUILD_FORCE_GENERATION` | OFF | Regenerate sources from xr.xml even if pre-generated exist |

## Architecture

### Loader (`src/loader/`)
The core runtime discovery and API dispatch layer. Key files:

- **`loader_instance.cpp`** — `xrCreateInstance` implementation, runtime negotiation
- **`loader_core.cpp`** — Core loader logic, manages runtime and layer bookkeeping
- **`manifest_file.cpp`** — Parses JSON runtime and API layer manifest files
- **`runtime_interface.cpp`** — Dynamic loading of runtime shared libraries
- **`api_layer_interface.cpp`** — Manages API layer loading and dispatch chain construction
- **`loader_logger.cpp`** / **`loader_logger_recorders.cpp`** — Internal logging infrastructure

### API Layers (`src/api_layers/`)
- **api_dump** — Logs every OpenXR API call and its parameters
- **core_validation** — Validates API usage, parameter correctness, and object lifetimes
- **best_practices** — Warns about suboptimal API usage patterns

Each layer is built as a MODULE library and loaded by the loader at runtime. Layers must export specific entry points and are described by JSON manifest files.

### Common (`src/common/`)
Shared utilities: filesystem helpers (`filesystem_utils.cpp`), object lifecycle tracking (`object_info`), hex handle formatting (`hex_and_handles.h`), graphics wrapper (`gfxwrapper_opengl.c`).

### Tests (`src/tests/`)
- **hello_xr** — Primary sample application, shows a colored cube in VR using multiple graphics backends (Vulkan, OpenGL, OpenGL ES, D3D11, D3D12, Metal)
- **loader_test** — Catch2-based tests for loader behavior (requires API layers to be built)
- **c_compile_test** — Verifies OpenXR headers are compilable from C
- **test_runtimes** — Stub runtime implementations used by loader tests

### Headers (`include/openxr/`)
The public OpenXR API headers. These are the canonical interface. `openxr_platform.h` provides platform-specific types.

### Specification (`specification/registry/xr.xml`)
The single source of truth for the entire OpenXR API — types, functions, enums, extensions. All code generation flows from this file.

## Code Quality

- **Formatting**: clang-format (prefer v14, accept v11–20). Run `./runClangFormat.sh`.
- **Spell check**: `./checkCodespell`
- **Python linting**: Uses `tox` with flake8/pycodestyle (see `tox.ini` for ignored rules)

The format-and-spell CI job runs in a `khronosgroup/docker-images:openxr.*` container.

## Platform Defines

Code compiles with platform-specific preprocessor defines set by the build system:
- `XR_OS_WINDOWS`, `XR_OS_LINUX`, `XR_OS_ANDROID`, `XR_OS_APPLE`
- `XR_USE_PLATFORM_WIN32`, `XR_USE_PLATFORM_XLIB`, `XR_USE_PLATFORM_XCB`, `XR_USE_PLATFORM_WAYLAND`, `XR_USE_PLATFORM_ANDROID`, `XR_USE_PLATFORM_MACOS`
- `XR_USE_GRAPHICS_API_VULKAN`, `XR_USE_GRAPHICS_API_OPENGL`, `XR_USE_GRAPHICS_API_OPENGL_ES`, `XR_USE_GRAPHICS_API_D3D11`, `XR_USE_GRAPHICS_API_D3D12`, `XR_USE_GRAPHICS_API_METAL`

## Versioning

API version is parsed from `xr.xml` (or `include/openxr/openxr.h` as fallback) at CMake time. An SDK hotfix version is read from the `HOTFIX` file if present.

## git blame

This repo tracks bulk-reformat commits in `.git-blame-ignore-revs`. Use `git blame --ignore-revs-file=.git-blame-ignore-revs` to get meaningful blame output.
