cmake_minimum_required(VERSION 3.15)
project(klibc)

if(NOT LIBC_ARCHITECTURE)
    set(LIBC_ARCHITECTURE "x86_64")
endif()

if(LIBC_ARCHITECTURE STREQUAL "x86_64")
    set(LIBC_DIR "x86_64-unknown-none-elf")
endif()

set(LIB_NAME "c") 

add_library(llvm-libc STATIC IMPORTED GLOBAL)
set_property(
    TARGET llvm-libc PROPERTY 
    IMPORTED_LOCATION "${CMAKE_CURRENT_SOURCE_DIR}/lib/${LIBC_DIR}/lib${LIB_NAME}.a"
)

set(CRT_LIBRARY_PATH "${CMAKE_CURRENT_SOURCE_DIR}/lib/${LIBC_DIR}" CACHE INTERNAL "")

target_include_directories(llvm-libc INTERFACE
    "${CMAKE_CURRENT_SOURCE_DIR}/include"
)
