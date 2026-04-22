cmake_minimum_required(VERSION 3.18 FATAL_ERROR)

# Prefer module/system nvcc over Conda nvcc when available. This avoids
# Conda CUDA linker directory injection that conflicts with system readline/tinfo.
if (NOT DEFINED CMAKE_CUDA_COMPILER)
  if (DEFINED ENV{CRAY_CUDATOOLKIT_DIR} AND EXISTS "$ENV{CRAY_CUDATOOLKIT_DIR}/bin/nvcc")
    set(CMAKE_CUDA_COMPILER "$ENV{CRAY_CUDATOOLKIT_DIR}/bin/nvcc" CACHE FILEPATH "CUDA compiler" FORCE)
  elseif (DEFINED ENV{CUDATOOLKIT_HOME} AND EXISTS "$ENV{CUDATOOLKIT_HOME}/bin/nvcc")
    set(CMAKE_CUDA_COMPILER "$ENV{CUDATOOLKIT_HOME}/bin/nvcc" CACHE FILEPATH "CUDA compiler" FORCE)
  elseif (DEFINED ENV{CUDA_HOME} AND EXISTS "$ENV{CUDA_HOME}/bin/nvcc")
    set(CMAKE_CUDA_COMPILER "$ENV{CUDA_HOME}/bin/nvcc" CACHE FILEPATH "CUDA compiler" FORCE)
  endif()
endif()

project(SwiftTransformer LANGUAGES CXX CUDA)

# Conda often injects link flags that make ld prefer its libtinfo over system
# libreadline requirements. Strip those flags at the root of the build.
if (DEFINED ENV{CONDA_PREFIX})
  set(_CONDA_LIB_FLAG "-L$ENV{CONDA_PREFIX}/lib")
  set(_CONDA_STUBS_FLAG "-L$ENV{CONDA_PREFIX}/lib/stubs")

  function(st_strip_conda_flags _var)
    if (DEFINED ${_var})
      set(_value "${${_var}}")
      separate_arguments(_args UNIX_COMMAND "${_value}")
      set(_filtered "")
      foreach(_a IN LISTS _args)
        if (_a STREQUAL "${_CONDA_LIB_FLAG}" OR _a STREQUAL "${_CONDA_STUBS_FLAG}")
          continue()
        endif()
        if (_a MATCHES "^-Wl,-rpath,")
          string(REPLACE "-Wl,-rpath," "" _rpath_list "${_a}")
          string(REPLACE ":" ";" _rpath_elems "${_rpath_list}")
          set(_filtered_rpath "")
          foreach(_rp IN LISTS _rpath_elems)
            if (_rp MATCHES "^$ENV{CONDA_PREFIX}/lib")
              continue()
            endif()
            list(APPEND _filtered_rpath "${_rp}")
          endforeach()
          if (_filtered_rpath)
            string(REPLACE ";" ":" _filtered_rpath_join "${_filtered_rpath}")
            list(APPEND _filtered "-Wl,-rpath,${_filtered_rpath_join}")
          endif()
          continue()
        endif()
        list(APPEND _filtered "${_a}")
      endforeach()
      string(JOIN " " _joined ${_filtered})
      set(${_var} "${_joined}" PARENT_SCOPE)
    endif()
  endfunction()

  st_strip_conda_flags(CMAKE_EXE_LINKER_FLAGS)
  st_strip_conda_flags(CMAKE_SHARED_LINKER_FLAGS)
  st_strip_conda_flags(CMAKE_MODULE_LINKER_FLAGS)
  st_strip_conda_flags(CMAKE_CXX_FLAGS)
  st_strip_conda_flags(CMAKE_CUDA_FLAGS)
endif()

# Prefer module/system toolkit roots to keep runtime and linker resolution
# aligned with system libraries.
if (NOT DEFINED CUDAToolkit_ROOT)
  if (DEFINED ENV{CRAY_CUDATOOLKIT_DIR})
    set(CUDAToolkit_ROOT $ENV{CRAY_CUDATOOLKIT_DIR})
  elseif (DEFINED ENV{CUDATOOLKIT_HOME})
    set(CUDAToolkit_ROOT $ENV{CUDATOOLKIT_HOME})
  elseif (DEFINED ENV{CUDA_HOME})
    set(CUDAToolkit_ROOT $ENV{CUDA_HOME})
  endif()
endif()

if (DEFINED ENV{CONDA_PREFIX} AND DEFINED CUDAToolkit_ROOT)
  if (CUDAToolkit_ROOT MATCHES "^$ENV{CONDA_PREFIX}(/|$)")
    message(FATAL_ERROR
      "CUDAToolkit_ROOT points to Conda (${CUDAToolkit_ROOT}). "
      "Load a system/module CUDA toolkit (for example CRAY_CUDATOOLKIT_DIR or CUDA_HOME) "
      "to avoid Conda library mixing during link.")
  endif()
endif()

find_package(CUDAToolkit 11.4 REQUIRED)

# gcc >= 8 is required, we do not support other compilers
if ((NOT CMAKE_CXX_COMPILER_ID STREQUAL "GNU") OR (CMAKE_CXX_COMPILER_VERSION VERSION_LESS 8.0))
    message(FATAL_ERROR "GCC 8.0 or higher is required")
endif()
# Add filesystem library for gcc < 9
link_libraries( "$<$<AND:$<CXX_COMPILER_ID:GNU>,$<VERSION_LESS:$<CXX_COMPILER_VERSION>,9.0>>:-lstdc++fs>" )

# Set up C++ standard
set(CXX_STD "17" CACHE STRING "C++ standard")
set(CMAKE_CXX_STANDARD ${CXX_STD})
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Switch between release mode and debug mode
# The user can use `-DBUILD_MODE=DEBUG` or `-DBUILD_MODE=RELEASE` to
# choose the build mode.
# If no option is provided, default to debug mode
if (BUILD_MODE)
  string(TOUPPER ${BUILD_MODE} BUILD_MODE)
  if (BUILD_MODE STREQUAL "DEBUG")
    set(DEBUG ON)
  elseif (BUILD_MODE STREQUAL "RELEASE")
    set(RELEASE ON)
  else()
    message(FATAL_ERROR "Unknown build mode: ${BUILD_MODE}")
  endif()
else()
  message("No build type selected, defaulting to RELEASE mode")
  message("Use -DBUILD_MODE=DEBUG or -DBUILD_MODE=RELEASE to specify build type")
  set(RELEASE ON)
endif()

# Set up C++ flag and CUDA flag
if (DEBUG)
  message("Building in debug mode")
  set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -g -G -DDEBUG")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -Wall -Wextra -Wno-unused-parameter -Wno-unused-function -DDEBUG")
else()
  message("Building in release mode")
  set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -O3 -DRELEASE -lineinfo --prec-div=false")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Ofast -Wall -Wextra -Wno-unused-parameter -Wno-unused-function -DRELEASE")
endif()

# Set up COMMON_HEADER_DIRS and COMMON_LIB_DIRS
set(COMMON_HEADER_DIRS
  ${PROJECT_SOURCE_DIR}
  ${PROJECT_SOURCE_DIR}/src/csrc
)

# Set up MPI and NCCL for multi-GPU communication
message("Building with MPI and NCCL")
set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/Modules)
set(MKL_MPI "openmpi")
find_package(NCCL REQUIRED)
find_package(MPI REQUIRED)
set(CMAKE_MODULE_PATH "") # prevent the bugs for pytorch building

# Add MPI and NCCL into COMMON_HEADER_DIRS
list(APPEND COMMON_HEADER_DIRS ${MPI_INCLUDE_PATH} ${NCCL_INCLUDE_DIR})

set(COMMON_LIBS CUDA::cudart)

# Add Python into COMMON_HEADER_DIRS & COMMON_LIB_DIRS
set(PYTHON_PATH "python" CACHE STRING "Python path")
execute_process(COMMAND ${PYTHON_PATH} "-c" "import sysconfig;
print(sysconfig.get_paths()['include']);"
                  RESULT_VARIABLE _PYTHON_SUCCESS
                  OUTPUT_VARIABLE PY_INCLUDE_DIR)
if (NOT _PYTHON_SUCCESS MATCHES 0)
	message(FATAL_ERROR "Python config Error.")
endif()
list(APPEND COMMON_HEADER_DIRS ${PY_INCLUDE_DIR})


# Add LibTorch into COMMON_HEADER_DIRS & COMMON_LIB_DIRS
execute_process(COMMAND ${PYTHON_PATH} "-c" "import os; import torch;
print(os.path.dirname(torch.__file__), end='');"
                  RESULT_VARIABLE _PYTHON_SUCCESS
                  OUTPUT_VARIABLE TORCH_DIR)
if (NOT _PYTHON_SUCCESS MATCHES 0)
	message(FATAL_ERROR "Torch config Error.")
endif()
list(APPEND CMAKE_PREFIX_PATH ${TORCH_DIR})
set(CAFFE2_USE_CUDNN 1)

# Optional compatibility bridge for Torch/Caffe2 builds that still require
# legacy nvToolsExt even when the selected CUDA toolkit does not ship it.
set(ST_NVTOOLSEXT_PATH "" CACHE FILEPATH "Absolute path to libnvToolsExt.so compatibility library")

# Torch's Caffe2 config still uses legacy FindCUDA variables.
if (NOT DEFINED CUDA_TOOLKIT_ROOT_DIR AND DEFINED CUDAToolkit_ROOT)
  set(CUDA_TOOLKIT_ROOT_DIR ${CUDAToolkit_ROOT})
endif()
if (NOT DEFINED CUDA_CUDART_LIBRARY AND DEFINED CUDAToolkit_ROOT)
  if (EXISTS "${CUDAToolkit_ROOT}/lib64/libcudart.so")
    set(CUDA_CUDART_LIBRARY "${CUDAToolkit_ROOT}/lib64/libcudart.so")
  elseif (EXISTS "${CUDAToolkit_ROOT}/lib/libcudart.so")
    set(CUDA_CUDART_LIBRARY "${CUDAToolkit_ROOT}/lib/libcudart.so")
  endif()
endif()
if (NOT DEFINED LIBNVTOOLSEXT)
  if (DEFINED CUDAToolkit_ROOT AND EXISTS "${CUDAToolkit_ROOT}/lib64/libnvToolsExt.so")
    set(LIBNVTOOLSEXT "${CUDAToolkit_ROOT}/lib64/libnvToolsExt.so")
  elseif (DEFINED CUDAToolkit_ROOT AND EXISTS "${CUDAToolkit_ROOT}/lib/libnvToolsExt.so")
    set(LIBNVTOOLSEXT "${CUDAToolkit_ROOT}/lib/libnvToolsExt.so")
  elseif (ST_NVTOOLSEXT_PATH AND EXISTS "${ST_NVTOOLSEXT_PATH}")
    set(LIBNVTOOLSEXT "${ST_NVTOOLSEXT_PATH}")
  elseif (DEFINED ENV{ST_NVTOOLSEXT_PATH} AND EXISTS "$ENV{ST_NVTOOLSEXT_PATH}")
    set(LIBNVTOOLSEXT "$ENV{ST_NVTOOLSEXT_PATH}")
  elseif (DEFINED ENV{CONDA_PREFIX} AND EXISTS "$ENV{CONDA_PREFIX}/lib/libnvToolsExt.so")
    # Single-file compatibility bridge: avoid adding -L$CONDA_PREFIX/lib.
    set(LIBNVTOOLSEXT "$ENV{CONDA_PREFIX}/lib/libnvToolsExt.so")
  endif()
endif()
if (DEFINED LIBNVTOOLSEXT)
  message(STATUS "Using nvToolsExt compatibility library: ${LIBNVTOOLSEXT}")
  set(CUDA_nvToolsExt_LIBRARY ${LIBNVTOOLSEXT})
  set(CUDA_NVTOOLSEXT_LIBRARY ${LIBNVTOOLSEXT})
  if (NOT TARGET CUDA::nvToolsExt)
    add_library(CUDA::nvToolsExt SHARED IMPORTED GLOBAL)
    set_target_properties(CUDA::nvToolsExt PROPERTIES IMPORTED_LOCATION "${LIBNVTOOLSEXT}")
  endif()
endif()

find_package(Torch REQUIRED)
list(APPEND COMMON_HEADER_DIRS "${TORCH_INCLUDE_DIRS}")

# Torch shared libraries need to stay together and must not be dropped by as-needed.
set(TORCH_LINK_LIBRARIES -Wl,--no-as-needed ${TORCH_LIBRARIES} -Wl,--as-needed)

# Conda's Torch often injects -L$CONDA_PREFIX/lib and -L$CONDA_PREFIX/lib/stubs,
# which makes ld prefer Conda's libtinfo over system libreadline requirements.
# Strip those search flags and any rpath entries that point at the conda lib dir.
if (DEFINED ENV{CONDA_PREFIX})
  set(_CONDA_LIB_FLAG "-L$ENV{CONDA_PREFIX}/lib")
  set(_CONDA_STUBS_FLAG "-L$ENV{CONDA_PREFIX}/lib/stubs")
  set(_TORCH_LINK_LIBRARIES_SANITIZED "")
  foreach(_item IN LISTS TORCH_LINK_LIBRARIES)
    if (_item STREQUAL "${_CONDA_LIB_FLAG}" OR _item STREQUAL "${_CONDA_STUBS_FLAG}")
      continue()
    endif()
    if (_item MATCHES "^-Wl,-rpath,")
      string(REPLACE "-Wl,-rpath," "" _rpath_list "${_item}")
      string(REPLACE ":" ";" _rpath_elems "${_rpath_list}")
      set(_filtered_rpath "")
      foreach(_rp IN LISTS _rpath_elems)
        if (_rp MATCHES "^$ENV{CONDA_PREFIX}/lib")
          continue()
        endif()
        list(APPEND _filtered_rpath "${_rp}")
      endforeach()
      if (_filtered_rpath)
        string(REPLACE ";" ":" _filtered_rpath_join "${_filtered_rpath}")
        list(APPEND _TORCH_LINK_LIBRARIES_SANITIZED "-Wl,-rpath,${_filtered_rpath_join}")
      endif()
      continue()
    endif()
    list(APPEND _TORCH_LINK_LIBRARIES_SANITIZED "${_item}")
  endforeach()
  set(TORCH_LINK_LIBRARIES ${_TORCH_LINK_LIBRARIES_SANITIZED})

  # Torch can inject linker flags globally while loading Caffe2/TorchConfig.
  # Strip these after find_package(Torch) to keep system readline/tinfo resolution intact.
  st_strip_conda_flags(CMAKE_EXE_LINKER_FLAGS)
  st_strip_conda_flags(CMAKE_SHARED_LINKER_FLAGS)
  st_strip_conda_flags(CMAKE_MODULE_LINKER_FLAGS)
  st_strip_conda_flags(CMAKE_CXX_FLAGS)
  st_strip_conda_flags(CMAKE_CUDA_FLAGS)
endif()


# Let COMMON_HEADER_DIRS take effect and make CUDA visible globally.
include_directories(${COMMON_HEADER_DIRS})
link_libraries(${COMMON_LIBS})


# Should turn off CXX11 ABI if pytorch is built with CXX11 ABI off
execute_process(COMMAND ${PYTHON_PATH} "-c" "import torch;
print(torch._C._GLIBCXX_USE_CXX11_ABI,end='');"
                  RESULT_VARIABLE _PYTHON_SUCCESS
                  OUTPUT_VARIABLE USE_CXX11_ABI)
message("-- USE_CXX11_ABI=${USE_CXX11_ABI}")
if (USE_CXX11_ABI)
  set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -D_GLIBCXX_USE_CXX11_ABI=1")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_GLIBCXX_USE_CXX11_ABI=1")
else()
  set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -D_GLIBCXX_USE_CXX11_ABI=0")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_GLIBCXX_USE_CXX11_ABI=0")
endif()


# GoogleTest Preparation - Code block copied from
#   https://google.github.io/googletest/quickstart-cmake.html
include(FetchContent)
FetchContent_Declare(
  googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG release-1.12.1
)
FetchContent_MakeAvailable(googletest)


# nlohmann_json Preparation - Code block copied from
#   https://github.com/nlohmann/json#cmake
FetchContent_Declare(
  json
  URL https://github.com/nlohmann/json/releases/download/v3.11.2/json.tar.xz
)
FetchContent_MakeAvailable(json)


# fetch latest argparse
FetchContent_Declare(
    argparse
    GIT_REPOSITORY https://github.com/p-ranav/argparse.git
)
FetchContent_MakeAvailable(argparse)

# Let all executable targets go to bin
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

# Add subdirectories
add_subdirectory(src)
