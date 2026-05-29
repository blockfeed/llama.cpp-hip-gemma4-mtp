# Maintainer: blockfeed
# Based on AUR llama.cpp-hip by Orion-zhen
# Adds PR #23398: Gemma 4 MTP support (github.com/ggml-org/llama.cpp/pull/23398)
#
# Binary-only addon for llama.cpp-hip. Installs llama-server + runtime libs
# to /opt/llama.cpp-gemma4-mtp/. Use as llama-swap's llama_server_path for
# Gemma 4 MTP model slots. Coexists with main llama.cpp-hip package.
#
# KNOWN BUG: Qwen models with --spec-type draft-mtp (no --model-draft) crash
# at GGML_ASSERT(sched) in memory_breakdown(). Only Gemma 4 models work.

pkgname=llama.cpp-hip-gemma4-mtp
_srcname="llama.cpp"
pkgver=b9378
pkgrel=2
pkgdesc="Gemma 4 MTP llama-server addon for llama.cpp-hip (PR #23398) — llama-swap companion"
arch=(x86_64)
url='https://github.com/ggml-org/llama.cpp'
license=('GPL3')
depends=(
  curl
  gcc-libs
  glibc
  hip-runtime-amd
  hipblas
  openmp
  rocblas
)
makedepends=(
  cmake
  git
  rocm-hip-sdk
)
options=(lto !debug)
source=(
  "${pkgname}-${pkgver}.tar.gz::https://github.com/ggml-org/llama.cpp/archive/refs/tags/${pkgver}.tar.gz"
  "gemma4-mtp-23398.patch"
)
sha256sums=('9e426b02100e0f473c59165bb154fac06b0a67e4718279d94ebddeccd5eaab2d'
            '85f306c876883e26f3be4ad6b8ef911540bc70e3160798a8d8bebcf346181eff')

prepare() {
  ln -sf "${_srcname}-${pkgver}" llama.cpp

  # Tailwind v4's oxide scanner walks up looking for the nearest .git to anchor
  # .gitignore lookup. In AUR helpers the parent .git is the AUR clone, whose
  # .gitignore is `*` + an allowlist for PKGBUILD/.SRCINFO. That rule ignores
  # tools/ui/src/lib/**/*.svelte, so Tailwind scans zero files and the built
  # bundle.css ships without any utility classes. An empty .git in the extracted
  # source tree stops the upward walk before that point.
  mkdir -p "${_srcname}-${pkgver}/.git"

  # Apply PR #23398: Gemma 4 MTP support
  # Adds GEMMA4_ASSISTANT architecture, MTP speculative decoding,
  # KV cache sharing, and draft model support.
  cd "${_srcname}-${pkgver}"
  patch -p1 -i "${srcdir}/gemma4-mtp-23398.patch"
}

build() {
  # Source ROCm profile if not already set
  if [[ -z "${ROCM_PATH}" ]]; then
    source /etc/profile
  fi
  export HIP_PATH="$(hipconfig -R)"
  export HIPCXX="$(hipconfig -l)/clang"
  export HIP_PLATFORM=amd

  local _cmake_options=(
    -B build
    -S "${_srcname}-${pkgver}"
    -DCMAKE_BUILD_TYPE=Release
    -DCMAKE_INSTALL_PREFIX='/opt/llama.cpp-gemma4-mtp'
    -DCMAKE_INSTALL_RPATH='/opt/llama.cpp-gemma4-mtp/lib'
    -DCMAKE_HIP_FLAGS="-mllvm --amdgpu-unroll-threshold-local=600"
    -DBUILD_SHARED_LIBS=ON
    -DLLAMA_BUILD_TESTS=OFF
    -DLLAMA_USE_SYSTEM_GGML=OFF
    -DLLAMA_BUILD_UI=OFF
    -DGGML_ALL_WARNINGS=OFF
    -DGGML_ALL_WARNINGS_3RD_PARTY=OFF
    -DGGML_BUILD_EXAMPLES=OFF
    -DGGML_BUILD_TESTS=OFF
    -DGGML_LTO=ON
    -DGGML_RPC=ON
    -DGGML_HIP=ON
    -DGGML_HIP_GRAPHS=ON
    -DHIP_PLATFORM=amd
    -DGGML_CUDA_FA_ALL_QUANTS=ON
    -DLLAMA_BUILD_NUMBER="${pkgver#b}"
    -Wno-dev
  )

  if [ -n "$CI" ] && [ "$CI" != 0 ]; then
    msg2 "CI = $CI detected, building universal package"
    _cmake_options+=(
      -DGGML_BACKEND_DL=ON
      -DGGML_CPU_ALL_VARIANTS=ON
      -DGGML_NATIVE=OFF
      # https://llvm.org/docs/AMDGPUUsage.html
      # gfx906: MI 50/60, Radeon VII
      # gfx101x: RX 5000 Series
      # gfx103x: RX 6000 Series
      # gfx110x: RX 7000 Series
      # gfx1151: Strix Halo
      # gfx120x: RX 9000 Series
      -DAMDGPU_TARGETS="gfx906;gfx1010;gfx1030;gfx1031;gfx1100;gfx1101;gfx1102;gfx1151;gfx1200;gfx1201"
    )
  else
    # Local build: optimize for host GPU
    _cmake_options+=(
      -DGGML_NATIVE=ON
    )
  fi

  # Allow user-specified additional CMake flags via env
  if [[ -n "$LLAMA_BUILD_EXTRA_ARGS" ]]; then
    msg2 "Applied custom CMake build args: $LLAMA_BUILD_EXTRA_ARGS"
    _cmake_options+=($LLAMA_BUILD_EXTRA_ARGS)
  fi

  cmake "${_cmake_options[@]}"
  cmake --build build -- -j $(nproc)
}

package() {
  DESTDIR="${pkgdir}" cmake --install build

  cd "${pkgdir}/opt/llama.cpp-gemma4-mtp"

  # Remove dev files (headers, cmake configs, pkgconfig)
  rm -rf include
  rm -rf share

  # Remove all binaries except llama-server
  find bin -type f ! -name 'llama-server' -delete
  find bin -type l -delete

  # Remove impl libs except the one llama-server loads at runtime
  find lib -name '*-impl.so' ! -name 'libllama-server-impl.so' -delete

  # Remove unversioned .so symlinks — keep only .so.X soname symlinks + versioned files
  find lib -name '*.so' -type l ! -name '*.so.*' -delete

  # Remove cmake and pkgconfig dirs (now empty)
  rm -rf lib/cmake
  rm -rf lib/pkgconfig
}
