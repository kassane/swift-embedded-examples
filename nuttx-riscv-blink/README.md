# Swift 6 on NuttX RTOS using CMake

## Description

Run blink example on NuttX RTOS.

## Requirements

- [NuttX](https://github.com/apache/nuttx) & [NuttX-apps](https://github.com/apache/nuttx-apps)
- [CMake](https://cmake.org/download/)
- [Swift 6](https://swift.org/download/)

## How to build

- **Get nuttx and nuttx-apps**
```bash
git clone --depth=1 --recursive https://github.com/apache/nuttx -b nuttx-12.5.1
git clone --depth=1 --recursive https://github.com/apache/nuttx-apps -b nuttx-12.5.1 apps
# optional (kconfig-frontends) build or use your distro pkg-manager
git clone  --depth=1 https://bitbucket.org/nuttx/tools.git
```

## How to run

- **Output**
```bash
qemu-system-riscv32 \
    -semihosting \
    -M virt,aclint=on \
    -cpu rv32 -smp 8 \
    -bios none \
    -kernel nuttx -nographic
nsh> blink
# <blink>
```

## References

- [Nuttx - Compiling with CMake](https://nuttx.apache.org/docs/latest/quickstart/compiling_cmake.html)
- [NuttX - C++ Example using CMake](https://nuttx.apache.org/docs/latest/guides/cpp_cmake.html)