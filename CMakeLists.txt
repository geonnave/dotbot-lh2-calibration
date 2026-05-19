# dotbot-lh2-calibration — opt-in CMake build (additive, SES projects unchanged).
#
# Quick start:
#
#   # Use the bundled dotbot-libs submodule (default — what CI does):
#   cmake -B build -DCMAKE_TOOLCHAIN_FILE=cmake/arm-none-eabi.cmake -G Ninja
#   cmake --build build
#
#   # Use a local dotbot-libs checkout for cross-repo iteration:
#   cmake -B build -DCMAKE_TOOLCHAIN_FILE=cmake/arm-none-eabi.cmake \
#         -DDOTBOT_LIBS=/path/to/local/DotBot-libs -G Ninja
#
# Mirrors the DotBot-firmware CMake build path (toolchain file, linker
# script, FetchContent of nrfx + CMSIS_5, dotbot_libs INTERFACE
# consumption). Existing SES .emProject files are untouched.

cmake_minimum_required(VERSION 3.20)

project(dotbot-lh2-calibration
    LANGUAGES C ASM
    VERSION 0.0.1
)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Cortex-M33 hard-float flags shared by every nRF5340 app-core target.
set(CPU_FLAGS
    -mcpu=cortex-m33
    -mthumb
    -mfloat-abi=hard
    -mfpu=fpv5-sp-d16
)

set(COMMON_FLAGS
    ${CPU_FLAGS}
    -ffunction-sections
    -fdata-sections
    -Wall
    -Wextra
    -Wpedantic
    # Silence dotbot-libs's nRF/System/*_cpu.c weak-alias declarations not
    # matching dummy_handler's inferred `noreturn`. Tracked separately.
    -Wno-missing-attributes
    -O0
    -g3
)

add_compile_options(${COMMON_FLAGS})
add_link_options(
    ${CPU_FLAGS}
    -Wl,--gc-sections
    -Wl,--print-memory-usage
    --specs=nosys.specs
    -nostartfiles
)

# ── Nordic MDK + ARM CMSIS-Core via FetchContent ─────────────────────
include(FetchContent)

FetchContent_Declare(nrfx
    GIT_REPOSITORY https://github.com/NordicSemiconductor/nrfx.git
    GIT_TAG        v3.5.0
    GIT_SHALLOW    TRUE
)
FetchContent_Declare(cmsis_core
    GIT_REPOSITORY https://github.com/ARM-software/CMSIS_5.git
    GIT_TAG        5.9.0
    GIT_SHALLOW    TRUE
)
FetchContent_MakeAvailable(nrfx cmsis_core)

set(NRFX_MDK_INCLUDE   ${nrfx_SOURCE_DIR}/mdk                       CACHE INTERNAL "Nordic MDK headers")
set(CMSIS_CORE_INCLUDE ${cmsis_core_SOURCE_DIR}/CMSIS/Core/Include  CACHE INTERNAL "ARM CMSIS-Core headers")

# ── dotbot-libs ──────────────────────────────────────────────────────
# Default: the bundled git submodule. Override with -DDOTBOT_LIBS=...
# for cross-repo iteration.
set(DOTBOT_LIBS "${CMAKE_CURRENT_SOURCE_DIR}/dotbot-libs"
    CACHE PATH "Path to dotbot-libs. Defaults to the bundled submodule.")

if(NOT EXISTS "${DOTBOT_LIBS}/CMakeLists.txt")
    message(FATAL_ERROR
        "DOTBOT_LIBS=${DOTBOT_LIBS} has no CMakeLists.txt. "
        "Either initialize the bundled submodule "
        "(git submodule update --init --recursive) or override "
        "-DDOTBOT_LIBS=/path/to/your/DotBot-libs checkout.")
endif()

add_subdirectory(${DOTBOT_LIBS} dotbot-libs-build EXCLUDE_FROM_ALL)

# ── Apps ─────────────────────────────────────────────────────────────

add_subdirectory(calibration)
