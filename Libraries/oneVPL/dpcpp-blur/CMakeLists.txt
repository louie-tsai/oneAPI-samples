# ##############################################################################
# Copyright (C) Intel Corporation
#
# SPDX-License-Identifier: MIT
# ##############################################################################
cmake_minimum_required(VERSION 3.10.2)

# This sample must be compiled and executed within the oneAPI product enviroment
# with the dpcpp compiler to use dpcpp features
set(DPCPP_COMPILER_NAME "dpcpp")
find_program(
  DPCPP_COMPILER
  NAMES ${DPCPP_COMPILER_NAME}
  HINTS /opt/intel/oneapi)

if(DPCPP_COMPILER)
  set(CMAKE_CXX_COMPILER ${DPCPP_COMPILER})
  message(STATUS "DPCPP compiler found, sample enabled")

  project(dpcpp-blur LANGUAGES CXX)

  # Default install places 64 bit runtimes in the environment, so we want to do
  # a 64 bit build by default.
  if(WIN32)
    if(NOT DEFINED CMAKE_GENERATOR_PLATFORM)
      set(CMAKE_GENERATOR_PLATFORM
          x64
          CACHE STRING "")
      message(STATUS "Generator Platform set to ${CMAKE_GENERATOR_PLATFORM}")
    endif()
  endif()

  set(CMAKE_CXX_STANDARD 17)

  set(TARGET dpcpp-blur)
  set(SOURCES src/dpcpp-blur.cpp)
  set(RUNARGS -i ${CMAKE_CURRENT_SOURCE_DIR}/content/cars_128x96.i420 -w 128 -h 96)

  # Set default build type to RelWithDebInfo if not specified
  if(NOT CMAKE_BUILD_TYPE)
    message(
      STATUS "Default CMAKE_BUILD_TYPE not set using Release with Debug Info")
    set(CMAKE_BUILD_TYPE
        "RelWithDebInfo"
        CACHE
          STRING
          "Choose build type from: None Debug Release RelWithDebInfo MinSizeRel"
          FORCE)
  endif()

  add_executable(${TARGET} ${SOURCES})

  if(MSVC)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
    if(NOT DEFINED ENV{VSCMD_VER})
      set(CMAKE_MSVCIDE_RUN_PATH $ENV{PATH})
    endif()
  endif()

  if(POLICY CMP0074)
    # ignore warning of VPL_ROOT in find_package search path
    cmake_policy(SET CMP0074 OLD)
  endif()

  find_package(VPL REQUIRED)
  target_link_libraries(${TARGET} VPL::dispatcher)

  if(UNIX)
    find_package(PkgConfig REQUIRED)
    # note: pkg-config version for libva is *API* version
    pkg_check_modules(PKG_LIBVA libva>=1.2 libva-drm>=1.2)
    if(PKG_LIBVA_FOUND)
      target_compile_definitions(${TARGET} PUBLIC -DLIBVA_SUPPORT)
      set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
      set(THREADS_PREFER_PTHREAD_FLAG TRUE)
      find_package(Threads REQUIRED)
      target_link_libraries(${TARGET} ${PKG_LIBVA_LIBRARIES}
                            ${PKG_THREAD_LIBRARIES} ze_loader)
      target_include_directories(${TARGET} PUBLIC ${PKG_LIBVA_INCLUDE_DIRS})
    else()
      message(WARNING "libva not found: building without libVA support")
    endif()
    target_compile_options(${TARGET} PRIVATE -Wall -Wextra -pedantic -Werror)
  endif()

  get_directory_property(has_parent PARENT_DIRECTORY)
  if(NOT has_parent)
    # only make run target available for stand-alone build
    add_custom_target(run ${TARGET} ${RUNARGS})
  endif()

else()
  message(STATUS "DPCPP compiler not found, dpcpp-blur build disabled")
endif()
