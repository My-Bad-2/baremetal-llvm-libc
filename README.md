# Baremetal LLVM-Libc Prebuilts

This repository provides prebuilt, static libraries of **llvm-libc**, **Clang-RT builtins**, and **CRT (C Runtime) object files** specifically configured for bare-metal environment.

For those interested in building the library from scratch, this repository also includes the necessary patch files and build instructions.

## Usage (Prebuilt)

To use the prebuilt library in your project, add this repository as a subdirectory in your `CMakeLists.txt`. You must specify the target architecture using the `LIBC_ARCHITECTURE` variable.

### Directory Structure Assumption
Ensure this repository is cloned into your project (e.g., under `libs/llvm-libc`).

### CMake Configuration

Add the following to your kernel's `CMakeLists.txt`:

```cmake
# 1. Specify the architecture (e.g., x86_64, aarch64)
set(LIBC_ARCHITECTURE "x86_64")

# 2. Add this repository
add_subdirectory(libs/llvm-libc)

# 3. Link the library to your kernel/executable
# Replace 'your-kernel-target' with the name of your executable
target_link_libraries(your-kernel-target PRIVATE llvm-libc ${CRT_LIBRARY_PATH}/libclang_rt.builtins.a)
```
*Note: The available options for `LIBC_ARCHITECTURE` depend on the folders provided in the `lib/` directory of this repo.*

## **Building from Source**
If you prefer to build the library yourself, or if you need to modify the source configurations, follow the steps below.

1. **Prerequisites**
- Clang/LLVM (Version 16+ recommended)
- CMake
- Ninja (optional, but recommended)
- Git

2. **Prepare the LLVM Source**
Clone the LLVM project and apply the provided patch file included in this repository (`x86_64-baremetal-libc.patch`). This patch adds the `x86_64` baremetal target to the source code. 

```Bash
git clone [https://github.com/llvm/llvm-project.git](https://github.com/llvm/llvm-project.git)
cd llvm-project
git apply /path/to/this/repo/patches/x86_64-baremetal-libc.patch
```
3. **Build Command (x86_64)**
Run the following CMake command to configure and build the library for an x86_64 bare-metal target.

```Bash
export PREFIX="/path/to/install/dir"
cmake -G Ninja -B build -S runtimes \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DLLVM_ENABLE_RUNTIMES="libc" \
  -DLLVM_LIBC_FULL_BUILD=ON \
  -DLIBC_TARGET_OS=baremetal \
  -DLIBC_TARGET_TRIPLE=x86_64-unknown-none-elf \
  -DCMAKE_BUILD_TYPE=Release \
  -DLIBC_CPU_FEATURES="" \
  -DCMAKE_CXX_FLAGS="-mno-sse -mno-sse2 -mno-avx -msoft-float" \
  -DCMAKE_C_FLAGS="-mno-sse -mno-sse2 -mno-avx -msoft-float" \
  -DCMAKE_INSTALL_PREFIX=$PREFIX

# Build the library
ninja -C build libc
```
Once built, the static archive will be located in `build/libc/lib/libc.a`.

## Integration & Kernel Hooks
**IMPORTANT:** While this library provides standard C functions (like `memcpy`, `strcpy`, `printf`), it operates in a freestanding environment.

To enable full functionality (especially for I/O and memory allocation functions), you must implement specific hooks within your kernel. The library relies on these symbols to interact with your hardware or memory manager.

### Required Hooks
You must implement the following functions in your kernel and link them against this library:

| Hook | Type (Function/Variable) | Function Signature/Variable Type | Note |
| :--- | :--- | :--- | :--- |
| __llvm_libc_stdio_write | Function | size_t __llvm_libc_stdio_write(void* cookie, const char* data, size_t size) | Pass the data to Kernel UART/console and return size |
| __llvm_libc_stdout_cookie | Variable | void* | A Dummy Variable `(void*)1` |

Failure to implement these hooks may result in linker errors or undefined behavior during runtime.

### C Runtime Compilation Configuration
The C Runtime (Compiler-RT) was configured with

```bash
export PREFIX="/path/to/install/dir"
cmake compiler-rt -B build-rt \
    -G Ninja \
    -DCMAKE_C_COMPILER=clang \
    -DCMAKE_CXX_COMPILER=clang++ \
    -DCMAKE_C_COMPILER_TARGET="x86_64-unknown-none-elf" \
    -DCMAKE_ASM_COMPILER_TARGET="x86_64-unknown-none-elf" \
    -DCMAKE_AR=$(which llvm-ar) \
    -DCMAKE_NM=$(which llvm-nm) \
    -DCMAKE_RANLIB=$(which llvm-ranlib) \
    -DCOMPILER_RT_BUILD_BUILTINS=ON \
    -DCOMPILER_RT_BUILD_SANITIZERS=OFF \
    -DCOMPILER_RT_BUILD_XRAY=OFF \
    -DCOMPILER_RT_BUILD_LIBFUZZER=OFF \
    -DCOMPILER_RT_BUILD_PROFILE=OFF \
    -DCOMPILER_RT_BUILD_MEMPROF=OFF \
    -DCOMPILER_RT_BUILD_ORC=OFF \
    -DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON \
    -DCOMPILER_RT_BAREMETAL_BUILD=ON \
    -DCOMPILER_RT_OS_DIR="baremetal" \
    -DCMAKE_TRY_COMPILE_TARGET_TYPE=STATIC_LIBRARY \
    -DCMAKE_C_FLAGS="-ffreestanding -O2 -nostdlib" \
    -DCMAKE_CXX_FLAGS="-ffreestanding -O2 -nostdlib" \
    -DCMAKE_INSTALL_PREFIX=$PREFIX
```

## Credits
LLVM Project: This repository is based on the LLVM's libc, compiler-rt project. Massive credit goes to the LLVM developers and contributors.

[License](https://github.com/llvm/llvm-project/blob/main/LICENSE.TXT)

[LLVM-Project](https://github.com/llvm/llvm-project.git)