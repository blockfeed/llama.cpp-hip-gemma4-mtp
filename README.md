# llama.cpp-hip-gemma4-mtp

Arch Linux PKGBUILD for llama.cpp with AMD ROCm/HIP optimizations,
patched with [PR #23398](https://github.com/ggml-org/llama.cpp/pull/23398)
for Gemma 4 MTP support. Pinned to tag `b9378`.

**Binary-only addon** for the main `llama.cpp-hip` package. Installs only
`llama-server` + runtime shared libs to `/opt/llama.cpp-gemma4-mtp/`.
Intended as a `llama_server_path` target in llama-swap for Gemma 4 MTP
model slots. Coexists with `llama.cpp-hip` — no path conflicts.

Adds the `GEMMA4_ASSISTANT` model architecture for multi-token prediction
speculative decoding, including KV cache sharing between trunk and draft
contexts, pre/post projection layers, and automatic device placement
validation. The draft model reads trunk KV read-only and requires no
dedicated KV cache of its own.

Based on AUR [llama.cpp-hip](https://aur.archlinux.org/packages/llama.cpp-hip)
by Orion-zhen.

**Arch Linux only.**

**Dependencies:** `hip-runtime-amd`, `hipblas`, `rocblas`, `rocm-hip-sdk`.

**CI targets:** gfx906, gfx1010, gfx1030, gfx1031, gfx1100, gfx1101, gfx1102,
gfx1151, gfx1200, gfx1201. Local builds auto-detect the host GPU.

**llama-swap config:**
```yaml
# Gemma 4 MTP model slot
- name: gemma4-mtp
  llama_server_path: /opt/llama.cpp-gemma4-mtp/bin/llama-server
  model: /path/to/gemma-4-26b-a4b-mtp.gguf
  model_draft: /path/to/gemma-4-draft.gguf
  ...
```

## Known Bug

**Qwen MTP models crash at startup.** The patched binary added device-memory
reporting that calls `memory_breakdown()` during startup. For Qwen's
internal-heads MTP (`--spec-type draft-mtp` without `--model-draft`), the
draft context's backend scheduler is null at that point — assertion fires,
process aborts:

```
ggml-backend.cpp:1945: GGML_ASSERT(sched) failed
  → ggml_backend_sched_get_buffer_type()
  → llama_context::memory_breakdown()
  → common_get_device_memory_data()
```

Gemma 4 works because it provides `--model-draft`, which takes a different
initialization path.

**Only Gemma 4 MTP models work with this build.** Use the main `llama.cpp-hip`
package for all other models. Awaiting upstream fix.
