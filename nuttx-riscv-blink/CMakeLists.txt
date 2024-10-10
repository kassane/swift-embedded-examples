cmake_minimum_required(VERSION 3.14...3.30)

project(blink
    VERSION 1.0
    DESCRIPTION "Blink on NuttX"
)
enable_language(Swift)

set(BOARD_CONFIG rv-virt:nsh)

if(POLICY CMP0169)
    cmake_policy(SET CMP0169 OLD)
endif()

include(FetchContent)
FetchContent_Declare(
    apps
    GIT_REPOSITORY https://github.com/apache/nuttx-apps.git
    GIT_TAG master
    SOURCE_DIR ${CMAKE_BINARY_DIR}/apps
    FIND_PACKAGE_ARGS
)
FetchContent_GetProperties(apps)
if(NOT apps_POPULATED)
    FetchContent_Populate(apps)
endif()

FetchContent_Declare(
    nuttx
    GIT_REPOSITORY https://github.com/apache/nuttx.git
    GIT_TAG master
    SOURCE_DIR ${CMAKE_BINARY_DIR}/nuttx
    FIND_PACKAGE_ARGS
)
FetchContent_GetProperties(nuttx)
if(NOT nuttx_POPULATED)
    FetchContent_Populate(nuttx)
endif()

if(NOT "${nuttx_SOURCE_DIR}" STREQUAL "")
    list(APPEND CMAKE_MODULE_PATH ${nuttx_SOURCE_DIR}/cmake)
endif()

if(CMAKE_HOST_SYSTEM_NAME STREQUAL "Windows")
    set(SCRIPT_SUFFIX .bat)
else()
    set(SCRIPT_SUFFIX .sh)
endif()

execute_process(
    COMMAND ${CMAKE_COMMAND} -E chdir ${nuttx_SOURCE_DIR}
    ${CMAKE_COMMAND} -E env PATH=${nuttx_SOURCE_DIR}/tools:$ENV{PATH}
    cmake -S ${nuttx_SOURCE_DIR} -B ${nuttx_BINARY_DIR} -G${CMAKE_GENERATOR} -DBOARD_CONFIG=${BOARD_CONFIG}
    RESULT_VARIABLE result
)
if(result)
    message(FATAL_ERROR "Failed to run configure")
endif()

execute_process(
    COMMAND ${CMAKE_COMMAND} -E chdir ${nuttx_SOURCE_DIR}
    ${CMAKE_COMMAND} -E env PATH=${nuttx_SOURCE_DIR}/tools:$ENV{PATH}
    cmake --build ${nuttx_BINARY_DIR}
    RESULT_VARIABLE result
)
if(result)
    message(FATAL_ERROR "Failed to run cmake menuconfig")
endif()


set(NUTTX_PATH "${nuttx_BINARY_DIR}")

include(cmake/riscv32.cmake)

set(AC_COMMON_FLAGS "${AC_COMMON_FLAGS} -Wall -Wshadow -Wundef -fno-strict-aliasing -Os")
set(AC_COMMON_FLAGS "${AC_COMMON_FLAGS} -D_DEBUG -D_LIBCPP_BUILD_STATIC -D_LIBCPP_NO_EXCEPTIONS ")
set(AC_COMMON_FLAGS "${AC_COMMON_FLAGS} -fno-exceptions -fcheck-new -fno-rtti -pedantic ")
set(AC_COMMON_FLAGS "${AC_COMMON_FLAGS} -nostdinc++")

set(AC_DEFINES "${AC_DEFINES} -DCONFIG_WCHAR_BUILTIN")

include_directories(
    src
    ${NUTTX_PATH}/include
    ${NUTTX_PATH}/include/libcxx
    ${NUTTX_PATH}/arch/chip
)

# Compute -Xcc flags to set up the C and C++ header search paths for Swift (for bridging header).
set(SWIFT_INCLUDES)
foreach(dir ${CMAKE_C_IMPLICIT_INCLUDE_DIRECTORIES})
    string(CONCAT SWIFT_INCLUDES ${SWIFT_INCLUDES} "-Xcc ")
    string(CONCAT SWIFT_INCLUDES ${SWIFT_INCLUDES} "-I${dir} ")
endforeach()
foreach(dir ${CMAKE_CXX_IMPLICIT_INCLUDE_DIRECTORIES})
    string(CONCAT SWIFT_INCLUDES ${SWIFT_INCLUDES} "-Xcc ")
    string(CONCAT SWIFT_INCLUDES ${SWIFT_INCLUDES} "-I${dir} ")
endforeach()

# Swift compiler flags to build in Embedded Swift mode, optimize for size, choose the right ISA, ABI, etc.
set(march_flag "rv32imafdc_zicsr_zifencei")
set(mabi_flag "ilp32d")
target_compile_options(${PROJECT_NAME} PUBLIC "$<$<COMPILE_LANGUAGE:Swift>:SHELL:
        -target riscv32-none-none-eabi
        -Xfrontend -function-sections -enable-experimental-feature Embedded -wmo -parse-as-library -Osize
        -Xcc -march=${march_flag} -Xcc -mabi=${mabi_flag}

        -pch-output-dir /tmp
        -Xfrontend -enable-single-module-llvm-emission

        ${SWIFT_INCLUDES}

        -import-bridging-header ${CMAKE_CURRENT_LIST_DIR}/src/BridgingHeader.h
    >")

# Enable Swift support in CMake, force Whole Module builds (required by Embedded Swift), and use "CMAKE_Swift_COMPILER_WORKS" to
# skip the trial compilations which don't (yet) correctly work when cross-compiling.
set(CMAKE_Swift_COMPILER_WORKS YES)
set(CMAKE_Swift_COMPILATION_MODE_DEFAULT wholemodule)
set(CMAKE_Swift_COMPILATION_MODE wholemodule)

# List of Swift source files to build.
target_sources(${PROJECT_NAME}
    PRIVATE
    src/Main.swift
)


link_directories(${NUTTX_PATH}/libs)

# add_executable(src/main.cpp)

# add_custom_command(
#     TARGET ${EXE_NAME}
#     POST_BUILD
#     COMMAND ${CMAKE_OBJCOPY} ARGS -S -O binary ${CMAKE_BINARY_DIR}/${EXE_NAME}.elf ${CMAKE_BINARY_DIR}/${EXE_NAME}.bin
# )

# target_link_libraries(${EXE_NAME} --start-group)

# target_link_libraries(${EXE_NAME} sched)
# target_link_libraries(${EXE_NAME} drivers)
# target_link_libraries(${EXE_NAME} boards)
# target_link_libraries(${EXE_NAME} c)
# target_link_libraries(${EXE_NAME} mm)
# target_link_libraries(${EXE_NAME} arch)
# target_link_libraries(${EXE_NAME} xx)
# target_link_libraries(${EXE_NAME} apps)
# target_link_libraries(${EXE_NAME} fs)
# target_link_libraries(${EXE_NAME} binfmt)
# target_link_libraries(${EXE_NAME} board)
# target_link_libraries(${EXE_NAME} gcc)
# target_link_libraries(${EXE_NAME} supc++)

# target_link_libraries(${EXE_NAME} --end-group)
