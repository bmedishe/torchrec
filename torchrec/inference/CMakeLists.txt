# Copyright (c) Meta Platforms, Inc. and affiliates.
# All rights reserved.
#
# This source code is licensed under the BSD-style license found in the
# LICENSE file in the root directory of this source tree.

cmake_minimum_required(VERSION 3.13 FATAL_ERROR)
project(inference)

# This step is crucial to ensure that the
# _REFLECTION, _GRPC_GRPCPP and _PROTOBUF_LIBPROTOBUF variables are set.
# e.g. ~/gprc/examples/cpp/cmake/common.cmake
include(${GRPC_COMMON_CMAKE_PATH}/common.cmake)


# abi and other flags

if(DEFINED GLIBCXX_USE_CXX11_ABI)
  if(${GLIBCXX_USE_CXX11_ABI} EQUAL 1)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_GLIBCXX_USE_CXX11_ABI=1")
    set(TORCH_CXX_FLAGS "-D_GLIBCXX_USE_CXX11_ABI=1")
  endif()
endif()

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC")

# dependencies

find_package(Torch REQUIRED)
find_package(folly REQUIRED)
find_package(gflags REQUIRED)
set(TORCHREC ${CMAKE_CURRENT_SOURCE_DIR}/../../)
set(THIRDPARTY ${TORCHREC}/third_party)

list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake" "${THIRDPARTY}/hipify_torch/cmake")
include(Hipify)
list (APPEND CMAKE_PREFIX_PATH /opt/rocm/hip /opt/rocm)
set(CMAKE_MODULE_PATH ${HIP_PATH}/cmake ${CMAKE_MODULE_PATH})

find_package(rocBLAS REQUIRED)
find_package(hipFFT REQUIRED)
find_package(hipRAND REQUIRED)
find_package(rocRAND REQUIRED)
find_package(hipSPARSE REQUIRED)
find_package(OpenMP REQUIRED)
find_package(rocPRIM REQUIRED)

include_directories(${Torch_INCLUDE_DIRS})
include_directories(${folly_INCLUDE_DIRS})
include_directories(${PYTORCH_FMT_INCLUDE_PATH})

set(CMAKE_CXX_STANDARD 17)

# torch deploy library
message("deploy interpreter", ${DEPLOY_INTERPRETER_PATH})
add_library(torch_deploy_internal STATIC
    ${DEPLOY_INTERPRETER_PATH}
    ${DEPLOY_SRC_PATH}/deploy.cpp
    ${DEPLOY_SRC_PATH}/loader.cpp
    ${DEPLOY_SRC_PATH}/path_environment.cpp
    ${DEPLOY_SRC_PATH}/elf_file.cpp)

# For python builtins. caffe2_interface_library properly
# makes use of the --whole-archive option.
target_link_libraries(torch_deploy_internal PRIVATE
  crypt pthread dl util m z ffi lzma readline nsl ncursesw panelw
)
target_link_libraries(torch_deploy_internal
  PUBLIC shm torch ${PYTORCH_LIB_FMT}
)
caffe2_interface_library(torch_deploy_internal torch_deploy)
# hipify code 
set(header_include_dir
${CMAKE_CURRENT_SOURCE_DIR}/include
${CMAKE_CURRENT_SOURCE_DIR}/src
${CMAKE_CURRENT_SOURCE_DIR}
)
hipify(CUDA_SOURCE_DIR ${PROJECT_SOURCE_DIR} HEADER_INCLUDE_DIR ${header_include_dir})
# inference library

# for our own header files
include_directories(include/)
include_directories(gen/)

# define our library target
add_library(inference SHARED
  src/Batching_hip.cpp
  src/BatchingQueue_hip.cpp
  src/GPUExecutor_hip.cpp
  src/ResultSplit_hip.cpp
  src/Exception_hip.cpp
)

# -rdynamic is needed to link against the static library
target_link_libraries(inference "-Wl,--no-as-needed -rdynamic"
  dl torch_deploy "${TORCH_LIBRARIES}" ${FBGEMM_LIB} ${FOLLY_LIBRARIES}
)

# for generated protobuf

# grpc headers. e.g. ~/.local/include
include_directories(${GRPC_HEADER_INCLUDE_PATH})

set(pred_grpc_srcs "gen/torchrec/inference/predictor.grpc.pb.cc")
set(pred_grpc_hdrs "gen/torchrec/inference/predictor.grpc.pb.h")
set(pred_proto_srcs "gen/torchrec/inference/predictor.pb.cc")
set(pred_proto_hdrs "gen/torchrec/inference/predictor.pb.h")

add_library(pred_grpc_proto SHARED
  ${pred_grpc_srcs}
  ${pred_grpc_hdrs}
  ${pred_proto_srcs}
  ${pred_proto_hdrs})

target_link_libraries(pred_grpc_proto
  ${_REFLECTION}
  ${_GRPC_GRPCPP}
  ${_PROTOBUF_LIBPROTOBUF})

# server

add_executable(server server_hip.cpp)
target_link_libraries(server
  inference
  pred_grpc_proto
  "${TORCH_LIBRARIES}"
  ${FOLLY_LIBRARIES}
  ${_REFLECTION}
  ${_GRPC_GRPCPP}
  ${_PROTOBUF_LIBPROTOBUF})
