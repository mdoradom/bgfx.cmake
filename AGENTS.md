# AGENTS.md

## Scope and intent
- This repo is a CMake packaging/build layer for upstream `bgfx`, `bimg`, and `bx` (git submodules), not a replacement for upstream GENie workflows.
- Most project-specific logic lives under `cmake/`; code under `bgfx/`, `bimg/`, and `bx/` is vendored upstream source.
- Prefer changing `cmake/*.cmake` and top-level `CMakeLists.txt` unless the task explicitly targets upstream code.

## Big picture architecture
- Root orchestration is in `CMakeLists.txt`: options/caches are defined once, then delegated via `add_subdirectory(cmake/bx)`, `add_subdirectory(cmake/bimg)`, `add_subdirectory(cmake/bgfx)`.
- Core target layering:
  - `bx` (`cmake/bx/bx.cmake`) -> base utilities/platform glue.
  - `bimg` (`cmake/bimg/bimg.cmake`) -> image library, links `bx`.
  - `bgfx` (`cmake/bgfx/bgfx.cmake`) -> renderer library, links `bx` + `bimg`.
- Tool and helper targets are in `cmake/bgfx/*.cmake` (for example `shaderc.cmake`, `geometryc.cmake`, `texturev.cmake`) and optionally grouped under custom target `tools`.
- `cmake/bgfx/shared.cmake` generates interface sources (`bgfx-vertexlayout`, `bgfx-shader`) using `configure_file` templates.

## Critical workflows (verified from README + CI)
- Initial setup (submodule-based):
  - `git submodule init && git submodule update`
  - `cmake -S . -B build -GNinja -DCMAKE_BUILD_TYPE=Release`
  - `cmake --build build`
- Examples are usually not in default build graph; build explicitly with:
  - `cmake --build build --target examples`
- Install/export package artifacts with:
  - `cmake -B build -DCMAKE_INSTALL_PREFIX="$PWD/install" -DBGFX_INSTALL=ON`
  - `cmake --build build --target install`
- Linux CI baseline dependencies: `libgl1-mesa-dev`, `libwayland-dev`, `libwayland-egl-backend-dev` (see `.github/workflows/ci.yml`).

## Project-specific conventions and patterns
- CMake file discovery intentionally uses `file(GLOB/GLOB_RECURSE ...)`; after adding/removing matching files, re-run CMake configure.
- Keep option wiring centralized in root `CMakeLists.txt` (for example `BGFX_BUILD_TOOLS_*`, `BGFX_BUILD_EXAMPLES`, `BGFX_CONFIG_*`), then consume in module files.
- Preserve exported target names and namespace (`bgfx::`) used by install/package config (`cmake/Config.cmake.in`).
- Formatting is enforced by `cmake-format` with `.cmake-format.py` (tabs enabled, width 120, canonical command/keyword case).

## Integration points and cross-component behavior
- External source locations are configurable via cache vars `BX_DIR`, `BIMG_DIR`, `BGFX_DIR`; non-absolute values are normalized to absolute paths in root `CMakeLists.txt`.
- Shader/asset build helpers exposed to downstream users live in `cmake/bgfxToolUtils.cmake`:
  - `bgfx_compile_shaders(...)`
  - `bgfx_compile_texture(...)`
  - `bgfx_compile_binary_to_header(...)`
- Installed package config (`cmake/Config.cmake.in`) computes `BGFX_SHADER_INCLUDE_PATH` and can import host tools when cross-compiling via `_bgfx_crosscompile_use_host_tool(...)`.
- Versioning is derived from upstream bgfx metadata and git history in `cmake/version.cmake` (API from `bgfx/scripts/bgfx.idl`, revision count from `git rev-list`).

## High-signal files to read before editing
- `CMakeLists.txt`
- `cmake/bgfx/CMakeLists.txt`
- `cmake/bgfx/bgfx.cmake`
- `cmake/bgfx/examples.cmake`
- `cmake/bgfxToolUtils.cmake`
- `.github/workflows/ci.yml`

