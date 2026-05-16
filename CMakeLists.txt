cmake_minimum_required(VERSION 3.20)
project(klibc VERSION 0.1 LANGUAGES C)

set(LIBC_ARCHITECTURE "x86_64" CACHE STRING "Target architecture for the prebuilt libc")
set_property(CACHE LIBC_ARCHITECTURE PROPERTY STRINGS x86_64 aarch64)

set(LIBC_RELEASE_URL "https://github.com/My-Bad-2/baremetal-llvm-libc/releases/download/Libc/baremetal-libc.tar.xz" CACHE STRING "GitHub release asset URL (.tar.xz) containing libs/")
set(LIBC_RELEASE_SHA256 "4b5dc549007d35cc613241fb027a8bdd1bd9379af1f822b88d58c5d9f871de7f" CACHE STRING "SHA256 of the GitHub release asset")
set(LIBC_DOWNLOAD_DIR "${CMAKE_BINARY_DIR}/_deps/llvm-libc" CACHE PATH "Download/extract directory")
set(LIBC_LIB_NAME "c" CACHE STRING "Base name of the static library (without lib prefix)")
set(LIBC_LIB_DIR_NAME "" CACHE STRING "Override libs/<dir> name from the release asset")

if(LIBC_ARCHITECTURE STREQUAL "x86_64")
    set(_libc_arch_dir "x86_64")
elseif(LIBC_ARCHITECTURE STREQUAL "aarch64")
    set(_libc_arch_dir "aarch64")
else()
    message(FATAL_ERROR "Unsupported LIBC_ARCHITECTURE: ${LIBC_ARCHITECTURE}")
endif()

if(LIBC_RELEASE_URL STREQUAL "")
    message(FATAL_ERROR "LIBC_RELEASE_URL must be set to a GitHub release .tar.xz asset URL")
endif()
if(LIBC_RELEASE_SHA256 STREQUAL "")
    message(FATAL_ERROR "LIBC_RELEASE_SHA256 must be set to verify the release asset")
endif()

if(LIBC_LIB_DIR_NAME STREQUAL "")
    set(_libc_lib_subdir "${_libc_arch_dir}")
else()
    set(_libc_lib_subdir "${LIBC_LIB_DIR_NAME}")
endif()

set(_libc_archive "${LIBC_DOWNLOAD_DIR}/llvm-libc-prebuilt.tar.xz")
set(_libc_extract_dir "${LIBC_DOWNLOAD_DIR}/src")
set(_libc_extract_stamp "${_libc_extract_dir}/.extract.stamp")

file(MAKE_DIRECTORY "${LIBC_DOWNLOAD_DIR}")
if(EXISTS "${_libc_archive}")
    file(SHA256 "${_libc_archive}" _libc_archive_hash)
    if(NOT _libc_archive_hash STREQUAL LIBC_RELEASE_SHA256)
        file(REMOVE "${_libc_archive}")
    endif()
endif()

if(NOT EXISTS "${_libc_archive}")
    file(DOWNLOAD "${LIBC_RELEASE_URL}" "${_libc_archive}"
        EXPECTED_HASH "SHA256=${LIBC_RELEASE_SHA256}"
        SHOW_PROGRESS
    )
endif()

if(EXISTS "${_libc_extract_stamp}")
    file(READ "${_libc_extract_stamp}" _libc_extract_hash)
endif()

if(NOT EXISTS "${_libc_extract_stamp}" OR NOT _libc_extract_hash STREQUAL LIBC_RELEASE_SHA256)
    file(REMOVE_RECURSE "${_libc_extract_dir}")
    file(MAKE_DIRECTORY "${_libc_extract_dir}")
    file(ARCHIVE_EXTRACT INPUT "${_libc_archive}" DESTINATION "${_libc_extract_dir}")
    file(WRITE "${_libc_extract_stamp}" "${LIBC_RELEASE_SHA256}")
endif()

set(_libc_lib_dir "${_libc_extract_dir}/libs/${_libc_lib_subdir}")
set(_libc_lib_path "${_libc_lib_dir}/lib${LIBC_LIB_NAME}.a")

if(NOT EXISTS "${_libc_lib_path}")
    message(FATAL_ERROR "Expected prebuilt library not found: ${_libc_lib_path}")
endif()

add_library(llvm-libc STATIC IMPORTED GLOBAL)
set_target_properties(llvm-libc PROPERTIES
    IMPORTED_LOCATION "${_libc_lib_path}"
    INTERFACE_INCLUDE_DIRECTORIES "${PROJECT_SOURCE_DIR}/include/${_libc_arch_dir}"
)
add_library(llvm-libc::llvm-libc ALIAS llvm-libc)

set(CRT_LIBRARY_PATH "${_libc_lib_dir}" CACHE INTERNAL "")
