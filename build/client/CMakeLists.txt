# Copyright (c) 2019-2021, NVIDIA CORPORATION. All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
#  * Redistributions of source code must retain the above copyright
#    notice, this list of conditions and the following disclaimer.
#  * Redistributions in binary form must reproduce the above copyright
#    notice, this list of conditions and the following disclaimer in the
#    documentation and/or other materials provided with the distribution.
#  * Neither the name of NVIDIA CORPORATION nor the names of its
#    contributors may be used to endorse or promote products derived
#    from this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS ``AS IS'' AND ANY
# EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
# PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL THE COPYRIGHT OWNER OR
# CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
# EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
# PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
# PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY
# OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
# (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

cmake_minimum_required (VERSION 3.18)
project (client)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

if(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
  message("Using MSVC as compiler, default target on Windows 10. "
		  "If the target system is not Windows 10, please update _WIN32_WINNT "
		  "to corresponding value.")
  set(CMAKE_CXX_FLAGS "/W4 /D_WIN32_WINNT=0x0A00 /EHsc")
  set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
else()
  set(CMAKE_CXX_FLAGS "-Wall -Wextra -Wno-unused-parameter -Werror -Wno-deprecated-declarations")
  set(CMAKE_CXX_FLAGS_DEBUG "-g")
  set(CMAKE_CXX_FLAGS_RELEASE "-O3")
endif()
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

include_directories("${PROJECT_SOURCE_DIR}/../..")
include_directories("${PROJECT_BINARY_DIR}")

#
# Dependencies
#
# FetchContent's composibility isn't very good. We must include the
# transitive closure of all repos so that we can override the tag.
#
include(FetchContent)

FetchContent_Declare(
  repo-common
  GIT_REPOSITORY https://github.com/triton-inference-server/common.git
  GIT_TAG ${TRITON_COMMON_REPO_TAG}
  GIT_SHALLOW ON
)
FetchContent_Declare(
  repo-core
  GIT_REPOSITORY https://github.com/triton-inference-server/core.git
  GIT_TAG ${TRITON_CORE_REPO_TAG}
  GIT_SHALLOW ON
)
FetchContent_MakeAvailable(repo-common repo-core)


if(${TRITON_CLIENT_SKIP_EXAMPLES})
  add_definitions(-DTRITON_CLIENT_SKIP_EXAMPLES=1)
endif() # TRITON_CLIENT_SKIP_EXAMPLES

if(${TRITON_ENABLE_HTTP})
  add_definitions(-DTRITON_ENABLE_HTTP=1)
endif() # TRITON_ENABLE_HTTP

if(${TRITON_ENABLE_GRPC})
  add_definitions(-DTRITON_ENABLE_GRPC=1)
endif() # TRITON_ENABLE_GRPC

if(NOT WIN32)
  #
  # CUDA
  #
  if(${TRITON_ENABLE_GPU})
    find_package(CUDA REQUIRED)
    message(STATUS "Using CUDA ${CUDA_VERSION}")

    add_definitions(-DTRITON_ENABLE_GPU=1)
    set(CUDA_NVCC_FLAGS -std=c++11)
  endif() # TRITON_ENABLE_GPU
endif() # NOT WIN32

if(${TRITON_ENABLE_GRPC})
  #
  # Protobuf
  #
  set(protobuf_MODULE_COMPATIBLE TRUE CACHE BOOL "protobuf_MODULE_COMPATIBLE" FORCE)
  find_package(Protobuf CONFIG REQUIRED)
  message(STATUS "Using protobuf ${Protobuf_VERSION}")
  include_directories(${Protobuf_INCLUDE_DIRS})

  #
  # GRPC
  #
  find_package(gRPC CONFIG REQUIRED)
  message(STATUS "Using gRPC ${gRPC_VERSION}")
  include_directories($<TARGET_PROPERTY:gRPC::grpc,INTERFACE_INCLUDE_DIRECTORIES>)
endif() # TRITON_ENABLE_GRPC

add_subdirectory(../../src/core src/core)
add_subdirectory(../../src/clients/c++ src/clients/c++)
add_subdirectory(../../src/clients/python src/clients/python)
