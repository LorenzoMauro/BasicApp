cmake_minimum_required(VERSION 3.28)
set(CMAKE_C_COMPILER_WORKS 1)
set(CMAKE_CXX_COMPILER_WORKS 1)
set(CMAKE_CUDA_COMPILER_WORKS 1)
set(PROJECT_NAME "StarterApp")

set(CMAKE_EXPORT_COMPILE_COMMANDS ON) # Generate compile_commands.json

# UTILS
function(assign_source_group)
  foreach(_source IN ITEMS ${ARGN})
    if(IS_ABSOLUTE "${_source}")
      file(RELATIVE_PATH _source_rel "${CMAKE_CURRENT_SOURCE_DIR}" "${_source}")
    else()
      set(_source_rel "${_source}")
    endif()
    get_filename_component(_source_path "${_source_rel}" PATH)
    string(REPLACE "/" "\\" _source_path_msvc "${_source_path}")
    source_group("${_source_path_msvc}" FILES "${_source}")
  endforeach()
endfunction(assign_source_group)
# Use vcpkg toolchain set(CMAKE_TOOLCHAIN_FILE
# "${CMAKE_SOURCE_DIR}/ext/vcpkg/scripts/buildsystems/vcpkg.cmake" CACHE STRING
# "Vcpkg toolchain file")

# Enable CMake's vcpkg integration include(CMakeDependentOption)
# include(FetchContent)

# Ensure the manifest mode is picked up set(VCPKG_FEATURE_FLAGS "manifests")
set(VCPKG_TARGET_TRIPLET x64-windows)

set(EXTERNAL ${CMAKE_SOURCE_DIR}/ext)

set(CMAKE_MODULE_PATH "${EXTERNAL}/CMake")

project(
  ${PROJECT_NAME}
  VERSION 1.0
  LANGUAGES CUDA CXX)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED True)
set(CMAKE_CUDA_STANDARD 20)
set(CMAKE_CUDA_STANDARD_REQUIRED True)

find_package(CUDAToolkit)
find_package(OptiX REQUIRED)
find_package(imgui CONFIG REQUIRED)
find_package(spdlog CONFIG REQUIRED)
find_package(Stb REQUIRED)
find_package(glad CONFIG REQUIRED)
find_package(glfw3 CONFIG REQUIRED)
add_subdirectory(src)

set_property(GLOBAL PROPERTY USE_FOLDERS ON)
