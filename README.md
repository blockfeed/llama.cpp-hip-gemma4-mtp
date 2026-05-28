# llama.cpp-hip-gemma4-mtp

Arch Linux PKGBUILD for llama.cpp with AMD ROCm/HIP optimizations,
patched with [PR #23398](https://github.com/ggml-org/llama.cpp/pull/23398)
for Gemma 4 MTP support. Pinned to tag `b9378`.

Adds the `GEMMA4_ASSISTANT` model architecture for multi-token prediction
speculative decoding, including KV cache sharing between trunk and draft
contexts, pre/post projection layers, and automatic device placement
validation. The draft model reads trunk KV read-only and requires no
dedicated KV cache of its own.

Based on AUR [llama.cpp-hip](https://aur.archlinux.org/packages/llama.cpp-hip)
by Orion-zhen. Not built or tested yet.

**Arch Linux only.**

**Dependencies:** `hip-runtime-amd`, `hipblas`, `rocblas`, `rocm-hip-sdk`.

**CI targets:** gfx906, gfx1010, gfx1030, gfx1031, gfx1100, gfx1101, gfx1102,
gfx1151, gfx1200, gfx1201. Local builds auto-detect the host GPU.

**Systemd:** includes `llama.cpp.service` and `/etc/conf.d/llama.cpp` config.
