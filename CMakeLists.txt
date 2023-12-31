cmake_minimum_required(VERSION 3.22)
project(transfers)

if (MSVC)
    # PDB debug information is not supported by buildcache.
    # Store debug info in the object files.
    option(TRANSFERS_DEBUG_SYMBOLS "generate debug symbols (debug builds)" ON)
    if (TRANSFERS_DEBUG_SYMBOLS)
        set(TRANSFERS_MSVC_DEBUG_FLAGS "/Z7")
    else()
        set(TRANSFERS_MSVC_DEBUG_FLAGS "")
    endif()
    string(REPLACE "/Zi" "${TRANSFERS_MSVC_DEBUG_FLAGS}" CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG}")
    string(REPLACE "/Zi" "${TRANSFERS_MSVC_DEBUG_FLAGS}" CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
    string(REPLACE "/Zi" "${TRANSFERS_MSVC_DEBUG_FLAGS}" CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELWITHDEBINFO}")
    string(REPLACE "/Zi" "${TRANSFERS_MSVC_DEBUG_FLAGS}" CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO}")
endif()

if(APPLE)
    # prevents ar from invoking ranlib, let CMake do it
    set(CMAKE_C_ARCHIVE_CREATE "<CMAKE_AR> Scr <TARGET> <LINK_FLAGS> <OBJECTS>")
    set(CMAKE_CXX_ARCHIVE_CREATE "<CMAKE_AR> Scr <TARGET> <LINK_FLAGS> <OBJECTS>")

    # tell ranlib to ignore empty compilation units
    set(CMAKE_C_ARCHIVE_FINISH "<CMAKE_RANLIB> -no_warning_for_no_symbols -c <TARGET>")
    set(CMAKE_CXX_ARCHIVE_FINISH "<CMAKE_RANLIB> -no_warning_for_no_symbols -c <TARGET>")
endif()

if (NOT DEFINED CMAKE_MSVC_RUNTIME_LIBRARY)
    if (TRANSFERS_MIMALLOC)
        set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
    else ()
        set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
    endif ()
endif ()

include(cmake/buildcache.cmake)
include(cmake/pkg.cmake)

# --- LINT ---
option(TRANSFERS_LINT "Run clang-tidy with the compiler." OFF)
if (TRANSFERS_LINT)
    # clang-tidy will be run on all targets defined hereafter
    include(cmake/clang-tidy.cmake)
endif ()

if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
    set(transfers-compile-options
            -Weverything
            -Wno-c++98-compat
            -Wno-c++98-compat-pedantic
            -Wno-newline-eof
            -Wno-missing-prototypes
            -Wno-padded
            -Wno-double-promotion
            -Wno-undef
            -Wno-undefined-reinterpret-cast
            -Wno-float-conversion
            -Wno-global-constructors
            -Wno-exit-time-destructors
            -Wno-switch-enum
            -Wno-c99-designator
            -Wno-zero-as-null-pointer-constant
            -Wno-missing-noreturn
            -Wno-undefined-func-template
            -Wno-unsafe-buffer-usage
            -Wno-c++20-compat
            -Werror)
elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "AppleClang")
    set(transfers-compile-options -Wall -Wextra -Werror)
elseif (MSVC)
    set(transfers-compile-options /WX)
else ()
    set(transfers-compile-options
            -Wall
            -Wextra
            -Wno-maybe-uninitialized)
    if (NOT CMAKE_CROSSCOMPILING)
        set(transfers-compile-options ${transfers-compile-options} -Werror)
    endif ()
endif ()

# --- LIB ---
file(GLOB_RECURSE transfers-files src/*.cc)
add_library(transfers ${transfers-files})
target_include_directories(transfers PUBLIC include)
target_link_libraries(transfers PUBLIC cista geo lmdb nigiri osmium ppr-routing utl)
target_compile_features(transfers PUBLIC cxx_std_23)
target_compile_options(transfers PRIVATE ${transfers_compile-options})

# --- TEST ---
file(GLOB_RECURSE transfers-test-files test/*.cc)
add_executable(transfers-test ${transfers-test-files})
target_link_libraries(transfers-test PUBLIC gtest transfers)
target_compile_options(transfers-test PRIVATE ${transfers-compile-options})