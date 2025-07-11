# Copyright (c) 2015-2023 The Khronos Group Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

cmake_minimum_required(VERSION 3.22.1)

project(spirv-tools)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O3 -pipe -integrated-as -mllvm -polly -mllvm -polly-vectorizer=stripmine -mllvm -polly-invariant-load-hoisting -mllvm -polly-run-inliner -mllvm -polly-run-dce -mllvm -polly-invariant-load-hoisting -mllvm -polly-run-inliner -mllvm -polly-run-dce -mllvm -polly-parallel -mllvm -polly-scheduling=dynamic -fdata-sections -ffunction-sections -fmerge-all-constants -mllvm -polly-detect-keep-going -mllvm -polly-ast-use-context -march=armv8-a+simd -fvisibility=hidden -Wall -Wextra -fPIC -DPIC -ferror-limit=0 -Wno-return-type -Wno-unused-parameter -Wno-c++98-compat -Wno-c++98-compat-pedantic -DGLM_FORCE_INTRINSICS")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 -pipe -integrated-as -mllvm -polly -mllvm -polly-vectorizer=stripmine -mllvm -polly-invariant-load-hoisting -mllvm -polly-run-inliner -mllvm -polly-run-dce -mllvm -polly-invariant-load-hoisting -mllvm -polly-run-inliner -mllvm -polly-run-dce -mllvm -polly-parallel -mllvm -polly-scheduling=dynamic -fdata-sections -ffunction-sections -fmerge-all-constants -mllvm -polly-detect-keep-going -mllvm -polly-ast-use-context -march=armv8-a+simd -fPIC -DPIC -Wall -Wextra -ferror-limit=0 -Wno-return-type -Wno-unused-parameter -Wno-c++98-compat -Wno-c++98-compat-pedantic -DGLM_FORCE_INTRINSICS -fno-rtti")

set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fuse-ld=lld -Wl,--strip-all -fvisibility=hidden -Wl,--gc-sections -Wl,--as-needed")

set(CMAKE_ANDROID_STL_TYPE c++_static)
set(CMAKE_ANDROID_LD "lld")

# Avoid a bug in CMake 3.22.1. By default it will set -std=c++11 for
# targets in test/*, when those tests need -std=c++17.
# https://github.com/KhronosGroup/SPIRV-Tools/issues/5340
# The bug is fixed in CMake 3.22.2
if (${CMAKE_VERSION} VERSION_GREATER_EQUAL "3.22.1")
  if (${CMAKE_VERSION} VERSION_LESS "3.22.2")
    cmake_policy(SET CMP0128 NEW)
  endif()
endif()

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

enable_testing()
set(SPIRV_TOOLS "SPIRV-Tools")

include(GNUInstallDirs)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# Require at least C++17
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
endif()
if(${CMAKE_CXX_STANDARD} LESS 17)
  message(FATAL_ERROR "SPIRV-Tools requires C++17 or later, but is configured for C++${CMAKE_CXX_STANDARD})")
endif()
set(CMAKE_CXX_EXTENSIONS OFF)


option(ENABLE_RTTI "Enables RTTI" OFF)
option(SPIRV_ALLOW_TIMERS "Allow timers via clock_gettime on supported platforms" ON)

if("${CMAKE_SYSTEM_NAME}" STREQUAL "Linux")
  set(SPIRV_TIMER_ENABLED ${SPIRV_ALLOW_TIMERS})
elseif("${CMAKE_SYSTEM_NAME}" MATCHES "Windows")
  add_definitions(-DSPIRV_WINDOWS)
elseif("${CMAKE_SYSTEM_NAME}" STREQUAL "CYGWIN")
  add_definitions(-DSPIRV_WINDOWS)
elseif("${CMAKE_SYSTEM_NAME}" STREQUAL "Android")
  set(SPIRV_TIMER_ENABLED ${SPIRV_ALLOW_TIMERS})
endif()

if (${SPIRV_TIMER_ENABLED})
  add_definitions(-DSPIRV_TIMER_ENABLED)
endif()

if ("${CMAKE_BUILD_TYPE}" STREQUAL "")
  message(STATUS "No build type selected, default to Debug")
  set(CMAKE_BUILD_TYPE "Debug")
endif()

option(SKIP_SPIRV_TOOLS_INSTALL "Skip installation" ${SKIP_SPIRV_TOOLS_INSTALL})
if(NOT ${SKIP_SPIRV_TOOLS_INSTALL})
  set(ENABLE_SPIRV_TOOLS_INSTALL ON)
endif()

option(SPIRV_BUILD_COMPRESSION "Build SPIR-V compressing codec" OFF)
if(SPIRV_BUILD_COMPRESSION)
  message(FATAL_ERROR "SPIR-V compression codec has been removed from SPIR-V tools. "
          "Please remove SPIRV_BUILD_COMPRESSION from your build options.")
endif(SPIRV_BUILD_COMPRESSION)

option(SPIRV_BUILD_FUZZER "Build spirv-fuzz" OFF)

set(SPIRV_LIB_FUZZING_ENGINE_LINK_OPTIONS "" CACHE STRING "Used by OSS-Fuzz to control, via link options, which fuzzing engine should be used")

option(SPIRV_BUILD_LIBFUZZER_TARGETS "Build libFuzzer targets" OFF)

option(SPIRV_WERROR "Enable error on warning" ON)
if(("${CMAKE_CXX_COMPILER_ID}" MATCHES "GNU") OR (("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang") AND (NOT CMAKE_CXX_SIMULATE_ID STREQUAL "MSVC")))
  set(COMPILER_IS_LIKE_GNU TRUE)
endif()
if(${COMPILER_IS_LIKE_GNU})
  set(SPIRV_WARNINGS -Wall -Wextra -Wnon-virtual-dtor -Wno-missing-field-initializers)

  if("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
    set(SPIRV_WARNINGS ${SPIRV_WARNINGS} -Wno-self-assign)
  endif()

  option(SPIRV_WARN_EVERYTHING "Enable -Weverything" ${SPIRV_WARN_EVERYTHING})
  if(${SPIRV_WARN_EVERYTHING})
    if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
      set(SPIRV_WARNINGS ${SPIRV_WARNINGS}
        -Weverything -Wno-c++98-compat -Wno-c++98-compat-pedantic -Wno-padded)
    elseif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
      set(SPIRV_WARNINGS ${SPIRV_WARNINGS} -Wpedantic -pedantic-errors)
    else()
      message(STATUS "Unknown compiler ${CMAKE_CXX_COMPILER_ID}, "
                     "so SPIRV_WARN_EVERYTHING has no effect")
    endif()
  endif()

  if(${SPIRV_WERROR})
    set(SPIRV_WARNINGS ${SPIRV_WARNINGS} -Werror)
  endif()
elseif(MSVC)
  set(SPIRV_WARNINGS -D_CRT_SECURE_NO_WARNINGS -D_SCL_SECURE_NO_WARNINGS /wd4800 /wd4819)

  if(${SPIRV_WERROR})
    set(SPIRV_WARNINGS ${SPIRV_WARNINGS} /WX)
  endif()
endif()

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/)

option(SPIRV_COLOR_TERMINAL "Enable color terminal output" ON)
if(${SPIRV_COLOR_TERMINAL})
  add_definitions(-DSPIRV_COLOR_TERMINAL)
endif()

option(SPIRV_LOG_DEBUG "Enable excessive debug output" OFF)
if(${SPIRV_LOG_DEBUG})
  add_definitions(-DSPIRV_LOG_DEBUG)
endif()

if (DEFINED SPIRV_TOOLS_EXTRA_DEFINITIONS)
  add_definitions(${SPIRV_TOOLS_EXTRA_DEFINITIONS})
endif()

# Library build setting definitions:
#
# * SPIRV_TOOLS_BUILD_STATIC - ON or OFF - Defaults to ON.
#   If enabled the following targets will be created:
#     ${SPIRV_TOOLS}-static - STATIC library.
#                             Has full public symbol visibility.
#     ${SPIRV_TOOLS}-shared - SHARED library.
#                             Has default-hidden symbol visibility.
#     ${SPIRV_TOOLS}        - will alias to one of above, based on BUILD_SHARED_LIBS.
#   If disabled the following targets will be created:
#     ${SPIRV_TOOLS}        - either STATIC or SHARED based on SPIRV_TOOLS_LIBRARY_TYPE.
#                             Has full public symbol visibility.
#     ${SPIRV_TOOLS}-shared - SHARED library.
#                             Has default-hidden symbol visibility.
#
# * SPIRV_TOOLS_LIBRARY_TYPE - SHARED or STATIC.
#   Specifies the library type used for building SPIRV-Tools libraries.
#   Defaults to SHARED when BUILD_SHARED_LIBS=1, otherwise STATIC.
#
# * SPIRV_TOOLS_FULL_VISIBILITY - "${SPIRV_TOOLS}-static" or "${SPIRV_TOOLS}"
#   Evaluates to the SPIRV_TOOLS target library name that has no hidden symbols.
#   This is used by internal targets for accessing symbols that are non-public.
#   Note this target provides no API stability guarantees.
#
# Ideally, all of these will go away - see https://github.com/KhronosGroup/SPIRV-Tools/issues/3909.
option(ENABLE_EXCEPTIONS_ON_MSVC "Build SPIRV-TOOLS with c++ exceptions enabled in MSVC" ON)
option(SPIRV_TOOLS_BUILD_STATIC "Build ${SPIRV_TOOLS}-static target. ${SPIRV_TOOLS} will alias to ${SPIRV_TOOLS}-static or ${SPIRV_TOOLS}-shared based on BUILD_SHARED_LIBS" ON)
if(SPIRV_TOOLS_BUILD_STATIC)
  set(SPIRV_TOOLS_FULL_VISIBILITY ${SPIRV_TOOLS}-static)
  set(SPIRV_TOOLS_LIBRARY_TYPE "STATIC")
else(SPIRV_TOOLS_BUILD_STATIC)
  set(SPIRV_TOOLS_FULL_VISIBILITY ${SPIRV_TOOLS})
  if (NOT DEFINED SPIRV_TOOLS_LIBRARY_TYPE)
      if(BUILD_SHARED_LIBS)
        set(SPIRV_TOOLS_LIBRARY_TYPE "SHARED")
      else()
        set(SPIRV_TOOLS_LIBRARY_TYPE "STATIC")
      endif()
  endif()
endif(SPIRV_TOOLS_BUILD_STATIC)

function(spvtools_default_compile_options TARGET)
  target_compile_options(${TARGET} PRIVATE ${SPIRV_WARNINGS})

  if (${COMPILER_IS_LIKE_GNU})
    target_compile_options(${TARGET} PRIVATE
      -Wall -Wextra -Wno-long-long -Wshadow -Wundef -Wconversion
      -Wno-sign-conversion -fno-exceptions)

    if(NOT ENABLE_RTTI)
        add_compile_options(-fno-rtti)
    endif()
    # For good call stacks in profiles, keep the frame pointers.
    if(NOT "${SPIRV_PERF}" STREQUAL "")
      target_compile_options(${TARGET} PRIVATE -fno-omit-frame-pointer)
    endif()
    if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang" OR "${CMAKE_CXX_COMPILER_ID}" STREQUAL "AppleClang")
      set(SPIRV_USE_SANITIZER "" CACHE STRING
        "Use the clang sanitizer [address|memory|thread|...]")
      if(NOT "${SPIRV_USE_SANITIZER}" STREQUAL "")
        target_compile_options(${TARGET} PRIVATE
          -fsanitize=${SPIRV_USE_SANITIZER})
        set_target_properties(${TARGET} PROPERTIES
          LINK_FLAGS -fsanitize=${SPIRV_USE_SANITIZER})
      endif()
      target_compile_options(${TARGET} PRIVATE
         -ftemplate-depth=1024)
    else()
      target_compile_options(${TARGET} PRIVATE
         -Wno-missing-field-initializers)
    endif()
  endif()

  if (MSVC)
    # Specify /EHs for exception handling. This makes using SPIRV-Tools as
    # dependencies in other projects easier.
    if(ENABLE_EXCEPTIONS_ON_MSVC)
      target_compile_options(${TARGET} PRIVATE /EHs)
    endif()
  endif()

  # For MinGW cross compile, statically link to the C++ runtime.
  # But it still depends on MSVCRT.dll.
  if (${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    if (NOT MSVC)
      set_target_properties(${TARGET} PROPERTIES
        LINK_FLAGS -static -static-libgcc -static-libstdc++)
    endif()
  endif()
endfunction()

if(NOT COMMAND find_host_package)
  macro(find_host_package)
    find_package(${ARGN})
  endmacro()
endif()
if(NOT COMMAND find_host_program)
  macro(find_host_program)
    find_program(${ARGN})
  endmacro()
endif()

# Tests require Python3
find_host_package(Python3 REQUIRED)

# Check type annotations in Perl code. Assumes mypy.
add_custom_target(spirv-tools-check-python-types
	COMMAND mypy Table
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/utils
)
set_target_properties(spirv-tools-check-python-types
  PROPERTIES EXCLUDE_FROM_ALL ON)

# Run Python unit tests
add_custom_target(spirv-tools-check-python-tests
	COMMAND Python3::Interpreter -m unittest discover -v -p "*test.py"
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/utils
)

# Check for symbol exports on Linux.
# At the moment, this check will fail on the OSX build machines for the Android NDK.
# It appears they don't have objdump.
if("${CMAKE_SYSTEM_NAME}" STREQUAL "Linux")
  macro(spvtools_check_symbol_exports TARGET)
    if (NOT "${SPIRV_SKIP_TESTS}")
      add_test(NAME spirv-tools-symbol-exports-${TARGET}
               COMMAND Python3::Interpreter
               ${spirv-tools_SOURCE_DIR}/utils/check_symbol_exports.py "$<TARGET_FILE:${TARGET}>")
    endif()
  endmacro()
else()
  macro(spvtools_check_symbol_exports TARGET)
    if (NOT "${SPIRV_SKIP_TESTS}")
      message("Skipping symbol exports test for ${TARGET}")
    endif()
  endmacro()
endif()

if(ENABLE_SPIRV_TOOLS_INSTALL)
  if(WIN32 AND NOT MINGW)
    macro(spvtools_config_package_dir TARGET PATH)
      set(${PATH} ${TARGET}/cmake)
    endmacro()
  else()
    macro(spvtools_config_package_dir TARGET PATH)
      set(${PATH} ${CMAKE_INSTALL_LIBDIR}/cmake/${TARGET})
    endmacro()
  endif()

  macro(spvtools_generate_config_file TARGET)
    file(WRITE ${CMAKE_BINARY_DIR}/${TARGET}Config.cmake
      "include(CMakeFindDependencyMacro)\n"
      "find_dependency(${SPIRV_TOOLS})\n"
      "include(\${CMAKE_CURRENT_LIST_DIR}/${TARGET}Targets.cmake)\n"
      "set(${TARGET}_LIBRARIES ${TARGET})\n"
      "get_target_property(${TARGET}_INCLUDE_DIRS ${TARGET} INTERFACE_INCLUDE_DIRECTORIES)\n")
  endmacro()
endif()

# Currently iOS and Android are very similar.
# They both have their own packaging (APP/APK).
# Which makes regular executables/testing problematic.
#
# Currently the only deliverables for these platforms are
# libraries (either STATIC or SHARED).
#
# Furthermore testing is equally problematic.
if (IOS OR ANDROID)
  set(SPIRV_SKIP_EXECUTABLES ON)
endif()

option(SPIRV_SKIP_EXECUTABLES "Skip building the executable and tests along with the library")
if (SPIRV_SKIP_EXECUTABLES)
  set(SPIRV_SKIP_TESTS ON)
endif()
option(SPIRV_SKIP_TESTS "Skip building tests along with the library")

# Defaults to ON.  The checks can be time consuming.
# Turn off if they take too long.
option(SPIRV_CHECK_CONTEXT "In a debug build, check if the IR context is in a valid state." ON)
if (${SPIRV_CHECK_CONTEXT})
  add_compile_options($<$<CONFIG:Debug>:-DSPIRV_CHECK_CONTEXT>)
endif()

# Precompiled header macro. Parameters are source file list and filename for pch cpp file.
macro(spvtools_pch SRCS PCHPREFIX)
  if(MSVC AND CMAKE_GENERATOR MATCHES "^Visual Studio" AND NOT "${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
    set(PCH_NAME "$(IntDir)\\${PCHPREFIX}.pch")
    # make source files use/depend on PCH_NAME
    set_source_files_properties(${${SRCS}} PROPERTIES COMPILE_FLAGS "/Yu${PCHPREFIX}.h /FI${PCHPREFIX}.h /Fp${PCH_NAME} /Zm300" OBJECT_DEPENDS "${PCH_NAME}")
    # make PCHPREFIX.cpp file compile and generate PCH_NAME
    set_source_files_properties("${PCHPREFIX}.cpp" PROPERTIES COMPILE_FLAGS "/Yc${PCHPREFIX}.h /Fp${PCH_NAME} /Zm300" OBJECT_OUTPUTS "${PCH_NAME}")
    list(APPEND ${SRCS} "${PCHPREFIX}.cpp")
  endif()
endmacro(spvtools_pch)

add_subdirectory(external)

# Warning about extra semi-colons.
#
# This is not supported on all compilers/versions. so enabling only
# for clang, since that works for all versions that our bots run.
#
# This is intentionally done after adding the external subdirectory,
# so we don't enforce this flag on our dependencies, some of which do
# not pass it.
#
# If the minimum version of CMake supported is updated to 3.0 or
# later, then check_cxx_compiler_flag could be used instead.
if("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
    add_compile_options("-Wextra-semi")
endif()

add_subdirectory(source)
add_subdirectory(tools)

add_subdirectory(test)
add_subdirectory(examples)

if(ENABLE_SPIRV_TOOLS_INSTALL)
  install(
    FILES
      ${CMAKE_CURRENT_SOURCE_DIR}/include/spirv-tools/libspirv.h
      ${CMAKE_CURRENT_SOURCE_DIR}/include/spirv-tools/libspirv.hpp
      ${CMAKE_CURRENT_SOURCE_DIR}/include/spirv-tools/optimizer.hpp
      ${CMAKE_CURRENT_SOURCE_DIR}/include/spirv-tools/linker.hpp
    DESTINATION
      ${CMAKE_INSTALL_INCLUDEDIR}/spirv-tools/)
endif(ENABLE_SPIRV_TOOLS_INSTALL)

if (NOT "${SPIRV_SKIP_TESTS}")
  add_test(NAME spirv-tools-copyrights
           COMMAND Python3::Interpreter utils/check_copyright.py
           WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
endif()

set(SPIRV_LIBRARIES "-lSPIRV-Tools-opt -lSPIRV-Tools -lSPIRV-Tools-link")
set(SPIRV_SHARED_LIBRARIES "-lSPIRV-Tools-shared")

# Build pkg-config file
# Use a first-class target so it's regenerated when relevant files are updated.
add_custom_command(
        OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/SPIRV-Tools.pc
        COMMAND ${CMAKE_COMMAND}
                      -DCHANGES_FILE=${CMAKE_CURRENT_SOURCE_DIR}/CHANGES
                      -DTEMPLATE_FILE=${CMAKE_CURRENT_SOURCE_DIR}/cmake/SPIRV-Tools.pc.in
                      -DOUT_FILE=${CMAKE_CURRENT_BINARY_DIR}/SPIRV-Tools.pc
                      -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
                      -DCMAKE_INSTALL_LIBDIR=${CMAKE_INSTALL_LIBDIR}
                      -DCMAKE_INSTALL_INCLUDEDIR=${CMAKE_INSTALL_INCLUDEDIR}
                      -DSPIRV_LIBRARIES=${SPIRV_LIBRARIES}
                      -P ${CMAKE_CURRENT_SOURCE_DIR}/cmake/write_pkg_config.cmake
        DEPENDS "CHANGES" "${CMAKE_CURRENT_SOURCE_DIR}/cmake/SPIRV-Tools.pc.in" "${CMAKE_CURRENT_SOURCE_DIR}/cmake/write_pkg_config.cmake")
add_custom_command(
        OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/SPIRV-Tools-shared.pc
        COMMAND ${CMAKE_COMMAND}
                      -DCHANGES_FILE=${CMAKE_CURRENT_SOURCE_DIR}/CHANGES
                      -DTEMPLATE_FILE=${CMAKE_CURRENT_SOURCE_DIR}/cmake/SPIRV-Tools-shared.pc.in
                      -DOUT_FILE=${CMAKE_CURRENT_BINARY_DIR}/SPIRV-Tools-shared.pc
                      -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
                      -DCMAKE_INSTALL_LIBDIR=${CMAKE_INSTALL_LIBDIR}
                      -DCMAKE_INSTALL_INCLUDEDIR=${CMAKE_INSTALL_INCLUDEDIR}
                      -DSPIRV_SHARED_LIBRARIES=${SPIRV_SHARED_LIBRARIES}
                      -P ${CMAKE_CURRENT_SOURCE_DIR}/cmake/write_pkg_config.cmake
        DEPENDS "CHANGES" "${CMAKE_CURRENT_SOURCE_DIR}/cmake/SPIRV-Tools-shared.pc.in" "${CMAKE_CURRENT_SOURCE_DIR}/cmake/write_pkg_config.cmake")
add_custom_target(spirv-tools-pkg-config 
        ALL 
        DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/SPIRV-Tools-shared.pc ${CMAKE_CURRENT_BINARY_DIR}/SPIRV-Tools.pc)

# Install pkg-config file
if (ENABLE_SPIRV_TOOLS_INSTALL)
  install(
    FILES
      ${CMAKE_CURRENT_BINARY_DIR}/SPIRV-Tools.pc
      ${CMAKE_CURRENT_BINARY_DIR}/SPIRV-Tools-shared.pc
    DESTINATION
      ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
endif()
