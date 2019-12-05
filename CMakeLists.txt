# Copyright (c) Microsoft Corporation.
# Licensed under the MIT License.

cmake_minimum_required(VERSION 3.5 FATAL_ERROR)

message(STATUS "Source Dir: ${CMAKE_SOURCE_DIR}")
message(STATUS "Host System name: ${CMAKE_HOST_SYSTEM_NAME}")
if ("${CMAKE_HOST_SYSTEM_NAME}" STREQUAL "Windows")
    set(CMAKE_SYSTEM_VERSION 10.0.18362.0 CACHE STRING INTERNAL FORCE)
    set(CMAKE_VS_WINDOWS_TARGET_PLATFORM_VERSION 10.0.18362.0 CACHE STRING INTERNAL FORCE)
endif()

project(msquic)

message(STATUS "System name: ${CMAKE_SYSTEM_NAME}")
message(STATUS "System version: ${CMAKE_SYSTEM_VERSION}")
message(STATUS "Platform version: ${CMAKE_VS_WINDOWS_TARGET_PLATFORM_VERSION}")

enable_testing()

# Set the default TLS method for each platform.
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
    set(QUIC_TLS "schannel" CACHE STRING "TLS Library to use")
else()
    set(QUIC_TLS "openssl" CACHE STRING "TLS Library to use")
endif()

option(QUIC_BUILD_TOOLS "Builds the tools code" ON)
option(QUIC_BUILD_TEST "Builds the test code" ON)
option(QUIC_ENABLE_LOGGING "Enables logging" ON)
option(QUIC_SANITIZE_ADDRESS "Enables address sanitizer" OFF)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/bld/obj)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/artifacts/bin)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/artifacts/bin)
file(MAKE_DIRECTORY ${CMAKE_SOURCE_DIR}/artifacts/logs)

if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
    # Generate the MsQuicEtw header file.
    file(MAKE_DIRECTORY ${CMAKE_SOURCE_DIR}/bld/inc)
    file(WRITE ${CMAKE_SOURCE_DIR}/bld/inc/MsQuicEtw.rc)
    include_directories(${CMAKE_SOURCE_DIR}/bld/inc)
    add_custom_command(
        OUTPUT ${CMAKE_SOURCE_DIR}/bld/inc/MsQuicEtw.h
        OUTPUT ${CMAKE_SOURCE_DIR}/bld/inc/MsQuicEtw.rc
        COMMAND mc.exe -um -h ${CMAKE_SOURCE_DIR}/bld/inc -r ${CMAKE_SOURCE_DIR}/bld/inc ${CMAKE_SOURCE_DIR}/manifest/MsQuicEtw.man)
    add_custom_target(MsQuicEtw
        DEPENDS ${CMAKE_SOURCE_DIR}/bld/inc/MsQuicEtw.h
        DEPENDS ${CMAKE_SOURCE_DIR}/bld/inc/MsQuicEtw.rc)

    # Custom build flags.
    set(QUIC_COMMON_FLAGS "-DWIN32_LEAN_AND_MEAN -DSECURITY_WIN32 /W4 /wd4100 /wd4101 /wd4189 /wd4458 /wd4459 /wd4702 /wd26451 /wd26812 /sdl /MP")
    if(QUIC_ENABLE_LOGGING)
        message(STATUS "Configuring for manifested ETW events and logging")
        set(QUIC_COMMON_FLAGS "${QUIC_COMMON_FLAGS} -DQUIC_EVENTS_MANIFEST_ETW -DQUIC_LOGS_MANIFEST_ETW")
    else()
        message(STATUS "Disabling events and logging")
        set(QUIC_COMMON_FLAGS "${QUIC_COMMON_FLAGS} -DQUIC_EVENTS_STUB -DQUIC_LOGS_STUB")
    endif()

    if(QUIC_TLS STREQUAL "openssl" OR QUIC_TLS STREQUAL "schannel")
        # OpenSSL and SChannel don't support 0-RTT yet.
        message(STATUS "Disabling 0-RTT support")
        set(QUIC_COMMON_FLAGS "${QUIC_COMMON_FLAGS} -DQUIC_0RTT_UNSUPPORTED")
    endif()

    if(QUIC_SANITIZE_ADDRESS)
        message(STATUS "Address sanitizer unsupported on this platform.")
    endif()

    set(QUIC_C_FLAGS "${QUIC_COMMON_FLAGS}")
    set(QUIC_CXX_FLAGS "${QUIC_COMMON_FLAGS} /std:c++17")

    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /Zi")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /Zi")
    set(CMAKE_SHARED_LINKER_FLAGS_RELEASE "${CMAKE_SHARED_LINKER_FLAGS_RELEASE} /DEBUG /OPT:REF /OPT:ICF")
else()
    # Custom build flags.
    set(QUIC_COMMON_FLAGS "-DQUIC_PLATFORM_LINUX -fms-extensions -fPIC -Wall -Wno-unknown-pragmas -Wno-unused-variable -Wno-unused-value -pthread")
    if(QUIC_ENABLE_LOGGING)
        message(STATUS "Configuring for syslog events and logging")
        set(QUIC_COMMON_FLAGS "${QUIC_COMMON_FLAGS} -DQUIC_EVENTS_SYSLOG -DQUIC_LOGS_SYSLOG")
    else()
        message(STATUS "Disabling events and logging")
        set(QUIC_COMMON_FLAGS "${QUIC_COMMON_FLAGS} -DQUIC_EVENTS_STUB -DQUIC_LOGS_STUB")
    endif()

    if(QUIC_TLS STREQUAL "openssl")
        # OpenSSL doesn't support 0-RTT yet.
        message(STATUS "Disabling 0-RTT support")
        set(QUIC_COMMON_FLAGS "${QUIC_COMMON_FLAGS} -DQUIC_0RTT_UNSUPPORTED")
    endif()

    if(QUIC_SANITIZE_ADDRESS)
        message(STATUS "Configuring address sanitizer")
        set(QUIC_COMMON_FLAGS "${QUIC_COMMON_FLAGS} -fsanitize=address -fno-omit-frame-pointer -Wno-maybe-uninitialized -O0 -Og -g")
    endif()

    set(QUIC_C_FLAGS "${QUIC_COMMON_FLAGS}")
    set(QUIC_CXX_FLAGS "${QUIC_COMMON_FLAGS} --std=c++17 -g -Wno-reorder -Wno-sign-compare -Wno-format")
endif()

if(QUIC_TLS STREQUAL "openssl")
    # Configure and build OpenSSL.
    add_custom_command(
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/submodules/openssl
        OUTPUT ${CMAKE_SOURCE_DIR}/bld/openssl/include
        OUTPUT ${CMAKE_SOURCE_DIR}/bld/openssl/lib/libcrypto.so
        OUTPUT ${CMAKE_SOURCE_DIR}/bld/openssl/lib/libssl.so
        COMMAND ./Configure linux-x86_64 enable-tls1_3 --prefix=${CMAKE_SOURCE_DIR}/bld/openssl
        COMMAND make -j$(nproc)
        COMMAND make install_sw)
    add_custom_target(OpenSSL
        DEPENDS ${CMAKE_SOURCE_DIR}/bld/openssl/include
        DEPENDS ${CMAKE_SOURCE_DIR}/bld/openssl/lib/libcrypto.so
        DEPENDS ${CMAKE_SOURCE_DIR}/bld/openssl/lib/libssl.so)
endif()

include_directories(${CMAKE_SOURCE_DIR}/inc)

# Product code
add_subdirectory(core)
add_subdirectory(platform)
add_subdirectory(bin)

# Tool code
if(QUIC_BUILD_TOOLS)
    add_subdirectory(tools/attack)
    add_subdirectory(tools/interop)
    add_subdirectory(tools/ping)
    add_subdirectory(tools/sample)
    add_subdirectory(tools/spin)
    if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
        add_subdirectory(tools/etw)
    endif()
endif()

# Test code
if(QUIC_BUILD_TEST)
    # Build the googletest framework.
    set(BUILD_GMOCK OFF CACHE BOOL "Builds the googlemock subproject")
    set(INSTALL_GTEST OFF CACHE BOOL "Enable installation of googletest. (Projects embedding googletest may want to turn this OFF.)")
    if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
        option(gtest_force_shared_crt "Use shared (DLL) run-time lib even when Google Test is built as static lib." ON)
    endif()
    add_subdirectory(submodules/googletest)

    add_subdirectory(test/lib)
    add_subdirectory(test/bin)
endif()