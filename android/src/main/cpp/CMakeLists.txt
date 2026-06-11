cmake_minimum_required(VERSION 3.4.1)

project(llamafu)

# Set the path to the llama.cpp directory
if(NOT DEFINED LLAMA_CPP_DIR)
    if(DEFINED ENV{LLAMA_CPP_DIR})
        set(LLAMA_CPP_DIR $ENV{LLAMA_CPP_DIR})
    else()
        # Use the git submodule by default
        set(LLAMA_CPP_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../../../../../llama.cpp")
    endif()
endif()

# Verify that llama.cpp directory exists
if(NOT EXISTS "${LLAMA_CPP_DIR}/CMakeLists.txt")
    message(FATAL_ERROR "llama.cpp not found at ${LLAMA_CPP_DIR}. Please run 'git submodule update --init --recursive' to initialize the llama.cpp submodule.")
endif()

# Add the llama.cpp include directories
include_directories(
    ${LLAMA_CPP_DIR}
    ${LLAMA_CPP_DIR}/include
    ${LLAMA_CPP_DIR}/common
    ${LLAMA_CPP_DIR}/ggml/include
    ${LLAMA_CPP_DIR}/tools/mtmd
)

# Build llama.cpp as part of our build process.
# Force static archives so they embed into libllamafu.so (single APK .so).
# BUILD_SHARED_LIBS must be set BEFORE add_subdirectory — option() respects existing value.
set(BUILD_SHARED_LIBS OFF CACHE BOOL "Build all llama.cpp/ggml libs as static" FORCE)
set(GGML_STATIC ON)
set(LLAMA_STATIC ON)

# === Vulkan GPU backend (Adreno/Mali/Xclipse offload) ===
# llama.cpp 의 GGML_VULKAN 백엔드 활성. 단말 GPU 로 layer offload → EXAONE 2.4B
# 같은 모델의 추론 시간 2-3x 단축 (Galaxy S21 기준 120s → 40-60s 예상).
# Dart 측은 Llamafu.init(useGpu: true) 로 활성화하며, Vulkan 미지원 단말은
# 자동으로 CPU fallback (llama.cpp 내부 처리).
#
# 빌드 전제 (PC 측, 빌드 시점만):
#  - CMake 3.22+ (Android SDK Manager → CMake 3.22.1)
#  - host 에 ninja (brew 또는 SDK cmake/3.22.1/bin)
#  - host 에 glslc (NDK 내 shader-tools 의 glslc 를 PATH 에)
#  - vulkan-headers — NDK sysroot 의 vulkan.h 는 C 헤더만, ggml-vulkan 은 vulkan.hpp (C++) 필요.
#    `brew install vulkan-headers` 로 해결. 헤더는 architecture-independent.
# 단말 호환성 — Mali GPU (Exynos S21 등) 는 ggml-vulkan 의 일부 extension 미지원으로
# ggml_backend_sched_reserve 시 null deref crash. 검증된 Vulkan 호환 단말
# (Snapdragon 8 Gen 2+ Adreno 740+) 확보 시 ON 으로 재전환.
set(GGML_VULKAN OFF CACHE BOOL "Enable Vulkan backend" FORCE)

# vulkan.hpp 검색 — host 측 brew 또는 시스템 경로.
find_path(VULKAN_HPP_INCLUDE_DIR
    NAMES vulkan/vulkan.hpp
    PATHS /opt/homebrew/include /usr/local/include /usr/include
    NO_CMAKE_FIND_ROOT_PATH  # cross-compile sysroot 무시, host path 검색.
)
if(VULKAN_HPP_INCLUDE_DIR)
    message(STATUS "Found vulkan.hpp at: ${VULKAN_HPP_INCLUDE_DIR}")
    include_directories(BEFORE ${VULKAN_HPP_INCLUDE_DIR})
else()
    message(FATAL_ERROR "vulkan.hpp not found. Run: brew install vulkan-headers")
endif()

# debug 빌드여도 native code 는 무조건 -O3 + NDEBUG 강제.
# Android NDK default 가 debug 시 -O0 적용 → llama.cpp 추론 30배+ 느려짐.
# dart 측은 debug (디버깅 가능), native 만 release 최적화 = 최적 조합.
set(CMAKE_C_FLAGS_DEBUG "-O3 -DNDEBUG")
set(CMAKE_CXX_FLAGS_DEBUG "-O3 -DNDEBUG")
set(CMAKE_C_FLAGS_RELEASE "-O3 -DNDEBUG")
set(CMAKE_CXX_FLAGS_RELEASE "-O3 -DNDEBUG")
add_compile_options(-O3 -DNDEBUG)
set(LLAMA_BUILD_TESTS OFF)
set(LLAMA_BUILD_EXAMPLES OFF)
set(LLAMA_BUILD_SERVER OFF)

# Add llama.cpp subdirectory
add_subdirectory(${LLAMA_CPP_DIR} llama.cpp EXCLUDE_FROM_ALL)

# Find the built libraries
find_library(libllama llama ${LLAMA_CPP_DIR}/build)
find_library(libggml ggml ${LLAMA_CPP_DIR}/build)

# MTMD (multimodal: vision/audio) sources — required because llamafu.cpp references
# clip_* and mtmd_* symbols. Mirrors host CMakeLists.txt (lines 127-150).
set(MTMD_SOURCES
    ${LLAMA_CPP_DIR}/tools/mtmd/mtmd.cpp
    ${LLAMA_CPP_DIR}/tools/mtmd/clip.cpp
    ${LLAMA_CPP_DIR}/tools/mtmd/mtmd-audio.cpp
    ${LLAMA_CPP_DIR}/tools/mtmd/mtmd-helper.cpp
    ${LLAMA_CPP_DIR}/tools/mtmd/models/cogvlm.cpp
    ${LLAMA_CPP_DIR}/tools/mtmd/models/conformer.cpp
    ${LLAMA_CPP_DIR}/tools/mtmd/models/glm4v.cpp
    ${LLAMA_CPP_DIR}/tools/mtmd/models/internvl.cpp
    ${LLAMA_CPP_DIR}/tools/mtmd/models/kimivl.cpp
    ${LLAMA_CPP_DIR}/tools/mtmd/models/llama4.cpp
    ${LLAMA_CPP_DIR}/tools/mtmd/models/llava.cpp
    ${LLAMA_CPP_DIR}/tools/mtmd/models/minicpmv.cpp
    ${LLAMA_CPP_DIR}/tools/mtmd/models/pixtral.cpp
    ${LLAMA_CPP_DIR}/tools/mtmd/models/qwen2vl.cpp
    ${LLAMA_CPP_DIR}/tools/mtmd/models/qwen3vl.cpp
    ${LLAMA_CPP_DIR}/tools/mtmd/models/siglip.cpp
    ${LLAMA_CPP_DIR}/tools/mtmd/models/whisper-enc.cpp
)

# Create the llamafu library
add_library(
    llamafu
    SHARED
    llamafu.cpp
    ${MTMD_SOURCES}
)

# clip.cpp 등이 stb_image 등을 쓰는 경우 vendor 헤더 필요
target_include_directories(llamafu PRIVATE
    ${LLAMA_CPP_DIR}/vendor
)

# For Android, we need to link android and log libraries
# For Linux host, we skip these
if (ANDROID)
    target_link_libraries(
        llamafu
        llama
        ggml
        android
        log
    )
else()
    target_link_libraries(
        llamafu
        llama
        ggml
    )
endif()

# Create a test program
add_executable(
    test_llamafu
    test_llamafu.cpp
)

target_link_libraries(
    test_llamafu
    llamafu
    llama
    ggml
)