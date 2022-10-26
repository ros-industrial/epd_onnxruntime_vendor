cmake_minimum_required(VERSION 3.13)

project(epd_onnxruntime_vendor)

find_package(ament_cmake REQUIRED)

macro(build_onnxruntime)
  set(extra_cmake_args)

  if(DEFINED CMAKE_BUILD_TYPE)
    list(APPEND extra_cmake_args -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE})
  endif()

  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -w")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14 -w")

  list(APPEND extra_cmake_args "-Donnxruntime_BUILD_UNIT_TESTS=OFF")
  list(APPEND extra_cmake_args "-Donnxruntime_USE_OPENMP=ON")
  list(APPEND extra_cmake_args "-Donnxruntime_BUILD_SHARED_LIB=ON")
  list(APPEND extra_cmake_args "-DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}")
  list(APPEND extra_cmake_args "-DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}")

  # TODO(anyone): Verify Windows build
  if(WIN32 AND NOT ${CMAKE_VERBOSE_MAKEFILE})
    set(should_log ON)  # prevent warnings in Windows CI
  else()
    set(should_log OFF)
  endif()

  if(DEFINED CMAKE_TOOLCHAIN_FILE)
    list(APPEND extra_cmake_args "-DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}")
  endif()

  # Cuda flag
  if(DEFINED USE_CUDA)
    # Add additional cuda related cmake variables
    list(APPEND extra_cmake_args "-Donnxruntime_USE_CUDA=ON")
    list(APPEND extra_cmake_args "-Donnxruntime_CUDA_HOME=/usr/local/cuda/")
    list(APPEND extra_cmake_args "-DCUDA_CUDA_LIBRARY=/usr/local/cuda/lib64/stubs/")
    list(APPEND extra_cmake_args "-Donnxruntime_CUDNN_HOME=/usr")
    list(APPEND extra_cmake_args "-Donnxruntime_USE_FULL_PROTOBUF=ON")
  endif()

  include(ExternalProject)
  ExternalProject_Add(onnxruntime
    GIT_REPOSITORY https://github.com/microsoft/onnxruntime
    GIT_TAG 36dc057913f968566eaa1646cb5db41d8c5e7654
    # GIT_SUBMODULES_RECURSE 1
    LOG_CONFIGURE ${should_log}
    LOG_BUILD ${should_log}
    SOURCE_SUBDIR cmake
    CMAKE_ARGS
      -DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/onnxruntime_install
      ${extra_cmake_args}
      -Wno-dev
  )

  # The external project will install to the build folder, but we'll install that on make install.
  install(
    DIRECTORY
      ${CMAKE_CURRENT_BINARY_DIR}/onnxruntime_install/
    DESTINATION
      ${CMAKE_INSTALL_PREFIX}/opt/epd_onnxruntime_vendor
    USE_SOURCE_PERMISSIONS
  )
endmacro()

build_onnxruntime()

# TODO(Briancbn): environment variables in Windows
if(WIN32)
  ament_environment_hooks(env_hook/epd_onnxruntime_vendor_library_path.bat)
  set(ENV_VAR_NAME "PATH")
  set(ENV_VAR_VALUE "opt\\epd_onnxruntime_vendor\\bin")
else()
  ament_environment_hooks(env_hook/epd_onnxruntime_vendor_library_path.sh)
  if(APPLE)
    set(ENV_VAR_NAME "DYLD_LIBRARY_PATH")
  else()
    set(ENV_VAR_NAME "LD_LIBRARY_PATH")
  endif()
  set(ENV_VAR_VALUE "opt/epd_onnxruntime_vendor/lib")
endif()
ament_environment_hooks(env_hook/epd_onnxruntime_vendor_library_path.dsv.in)

ament_package(
  CONFIG_EXTRAS "epd_onnxruntime_vendor-extras.cmake.in"
)
