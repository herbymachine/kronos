cmake_minimum_required(VERSION 3.20)

project(chdr VERSION 0.2 LANGUAGES C)

option(BUILD_LTO "Compile libchdr with link-time optimization if supported" OFF)
if(BUILD_LTO)
  include(CheckIPOSupported)
  check_ipo_supported(RESULT HAVE_IPO)
  if(HAVE_IPO)
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
  endif()
endif()

option(BUILD_FUZZER "Build instrumented binary for fuzzing with libfuzzer, requires clang")
if(BUILD_FUZZER)
  # Override CFLAGS early for instrumentation. Disable shared libs for instrumentation.
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=address,fuzzer-no-link")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address,fuzzer-no-link")
endif()

include(GNUInstallDirs)

#--------------------------------------------------
# dependencies
#--------------------------------------------------


# lzma
if(NOT TARGET lzma)
  add_subdirectory(deps/lzma-25.01 EXCLUDE_FROM_ALL)
endif()
list(APPEND CHDR_LIBS lzma)
list(APPEND CHDR_INCLUDES lzma)

# zlib
if(NOT TARGET zlibstatic)
option(ZLIB_BUILD_EXAMPLES "Enable Zlib Examples" OFF)
add_subdirectory(deps/zlib-1.3.1 EXCLUDE_FROM_ALL)
set_target_properties(zlibstatic PROPERTIES POSITION_INDEPENDENT_CODE ON)
endif()
list(APPEND CHDR_LIBS zlibstatic)

# zstd
if(NOT TARGET libzstd_static)
option(ZSTD_BUILD_SHARED "BUILD SHARED LIBRARIES" OFF)
option(ZSTD_BUILD_PROGRAMS "BUILD PROGRAMS" OFF)
option(ZSTD_LEGACY_SUPPORT "LEGACY SUPPORT" OFF)
add_subdirectory(deps/zstd-1.5.7/build/cmake EXCLUDE_FROM_ALL)
endif()
list(APPEND CHDR_LIBS libzstd_static)

#--------------------------------------------------
# chdr
#--------------------------------------------------

set(CHDR_SOURCES
  src/libchdr_bitstream.c
  src/libchdr_cdrom.c
  src/libchdr_chd.c
  src/libchdr_flac.c
  src/libchdr_huffman.c
)

list(APPEND CHDR_INCLUDES ${CMAKE_CURRENT_BINARY_DIR}/include)

add_library(chdr-static STATIC ${CHDR_SOURCES})
target_include_directories(chdr-static PRIVATE ${CHDR_INCLUDES} PUBLIC include)
target_link_libraries(chdr-static PRIVATE ${CHDR_LIBS} ${PLATFORM_LIBS})

if(MSVC)
  target_compile_definitions(chdr-static PRIVATE _CRT_SECURE_NO_WARNINGS)
endif()
