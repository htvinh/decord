# Decord — Agent Guide

## Build

```sh
# submodules required
git submodule update --init --recursive

mkdir -p build && cd build
cmake .. -DUSE_CUDA=0          # add -DFFMPEG_DIR=/path for custom FFmpeg
make -j$(nproc)

# Python bindings (after C++ build)
pip install -e ./python        # not repo root
```

- CMake minimum 3.8.2, C++11.
- `USE_CUDA=ON` enables NVDEC hardware decoder; `-DFFMPEG_DIR` for custom path.
- Official build scripts: `tools/build_macos_10_9.sh`, `tools/build_manylinux2010.sh`.

## Apple Silicon (ARM64 / M-series)

- Requires Homebrew FFmpeg. pkg-config auto-detects `/opt/homebrew/` paths.
- The codebase has been patched for FFmpeg 8.1 API compatibility. Key changes if updating FFmpeg:
  - `src/video/ffmpeg/ffmpeg_common.h`: added `#include <libavcodec/bsf.h>` (`AVBSFContext` moved out of `avcodec.h`)
  - `src/video/video_reader.cc`: `av_find_best_stream` 5th arg is now `const AVCodec**`; `av_stream_get_side_data` replaced with `av_packet_side_data_get(st->codecpar->coded_side_data, ...)`
  - `src/audio/audio_reader.cc`: `ch_layout` replaces `channels`/`channel_layout`; `avcodec_close` removed; `av_opt_set_chlayout` replaces `av_opt_set_channel_layout`
  - `src/video/ffmpeg/filter_graph.cc`: `av_opt_set_int_list` for `pix_fmts` removed; format enforced via `,format=rgb24` in filter description instead.

## CI / Sanity

- CI: `.github/workflows/ccpp.yml` — `mkdir build && cd build && cmake .. -DUSE_CUDA=0 && make && pip install -e ./python && python3 -c "import decord; print(decord.__version__)"`
- No linter, formatter, typechecker, or pre-commit config in repo.

## Tests

```sh
# Python (from repo root, after pip install -e ./python)
python -m pytest tests/python/unittests/
# single test file:
python -m pytest tests/python/unittests/test_video_reader.py

# C++ (requires gtest, build with cmake, then):
make cpptest
```

- Test data lives in `tests/test_data/` (`.mov`, `.mp4` files).
- Benchmarks in `tests/benchmark/`.

## Testing quirks

- Python tests require the C library built (`libdecord`) and `pip install -e ./python` first.
- C++ tests use gtest (optional cmake target).

## Architecture

- **C++ core** → **PackedFunc registry** (`DECORD_REGISTER_GLOBAL`) → **Python FFI** (`python/decord/_ffi/`).
- Pattern adapted from TVM/DGL: C++ registers functions by string name, Python looks them up via `get_global_func`.
- Version is duplicated in 3 places — update all with `tools/update_version.py`:
  - `python/decord/_ffi/libinfo.py`
  - `include/decord/runtime/c_runtime_api.h`
  - `src/runtime/file_util.cc`
- Main entrypoints: `python/decord/__init__.py` → C API bindings in `src/video/video_interface.cc`, `src/audio/audio_interface.cc`, `src/video/video_loader.cc`.
- Bridges to PyTorch, MXNet, TF, TVM in `python/decord/bridge/` (via DLPack, zero-copy).
- `python/decord/data/` contains MXNet-based dataloaders (not the primary API).

## Key conventions

- `DECORD_REGISTER_GLOBAL("name")` in C++ registers a `PackedFunc` callable from Python as `get_global_func("name")()`.
- `#if DECORD_USE_CUDA` guards CUDA code paths.
- All `import decord` symbols come from `python/decord/__init__.py`.
