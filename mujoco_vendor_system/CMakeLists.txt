cmake_minimum_required(VERSION 3.16)
project(mujoco_vendor)

# Drop-in replacement for pal-robotics/mujoco_vendor. Instead of downloading a MuJoCo
# release it wraps an existing binary SDK so that the C++ simulation matches the MuJoCo
# version used by the Python tooling (e.g. robosuite). Installs into the same
# opt/mujoco_vendor layout and exports the same targets/variables as upstream, so the
# mujoco_ros2_control packages build unchanged.

find_package(ament_cmake REQUIRED)
include(FetchContent)

if(DEFINED MUJOCO_DIR)
  set(MUJOCO_ROOT_DIR "${MUJOCO_DIR}")
elseif(DEFINED ENV{MUJOCO_DIR})
  set(MUJOCO_ROOT_DIR "$ENV{MUJOCO_DIR}")
elseif(DEFINED ENV{MUJOCO_PY_MUJOCO_PATH})
  set(MUJOCO_ROOT_DIR "$ENV{MUJOCO_PY_MUJOCO_PATH}")
else()
  set(MUJOCO_ROOT_DIR "/opt/.mujoco/mujoco-3.5.0")
endif()

if(NOT EXISTS "${MUJOCO_ROOT_DIR}/include/mujoco/mujoco.h" OR NOT EXISTS "${MUJOCO_ROOT_DIR}/lib/libmujoco.so")
  message(FATAL_ERROR "mujoco_vendor (system): no MuJoCo SDK found at '${MUJOCO_ROOT_DIR}'. "
    "Set -DMUJOCO_DIR=... or the MUJOCO_DIR environment variable.")
endif()
message(STATUS "mujoco_vendor (system): using MuJoCo SDK at ${MUJOCO_ROOT_DIR}")

set(_SIM_DIR "${MUJOCO_ROOT_DIR}/simulate")
if(NOT EXISTS "${_SIM_DIR}/simulate.h")
  set(_SIM_DIR "${MUJOCO_ROOT_DIR}/include/simulate")
endif()
if(NOT EXISTS "${_SIM_DIR}/simulate.h")
  message(FATAL_ERROR "mujoco_vendor (system): simulate sources not found in '${MUJOCO_ROOT_DIR}'.")
endif()

install(DIRECTORY ${MUJOCO_ROOT_DIR}/include/ DESTINATION opt/${PROJECT_NAME}/include)
install(DIRECTORY ${MUJOCO_ROOT_DIR}/lib/ DESTINATION opt/${PROJECT_NAME}/lib USE_SOURCE_PERMISSIONS)
if(EXISTS "${MUJOCO_ROOT_DIR}/bin")
  install(DIRECTORY ${MUJOCO_ROOT_DIR}/bin/ DESTINATION opt/${PROJECT_NAME}/bin USE_SOURCE_PERMISSIONS)
endif()

add_library(mujoco INTERFACE)
add_library(mujoco::mujoco ALIAS mujoco)
target_include_directories(mujoco INTERFACE
  $<BUILD_INTERFACE:${MUJOCO_ROOT_DIR}/include>
  $<INSTALL_INTERFACE:opt/${PROJECT_NAME}/include>
)
target_link_libraries(mujoco INTERFACE
  $<BUILD_INTERFACE:${MUJOCO_ROOT_DIR}/lib/libmujoco.so>
  $<INSTALL_INTERFACE:opt/${PROJECT_NAME}/lib/libmujoco.so>
)
install(TARGETS mujoco EXPORT export_${PROJECT_NAME})

# Same pinned lodepng as upstream MuJoCo's simulate.
set(MUJOCO_DEP_VERSION_lodepng "b4ed2cd7ecf61d29076169b49199371456d4f90b")
FetchContent_Declare(lodepng
  GIT_REPOSITORY https://github.com/lvandeve/lodepng.git
  GIT_TAG ${MUJOCO_DEP_VERSION_lodepng}
)
FetchContent_GetProperties(lodepng)
if(NOT lodepng_POPULATED)
  FetchContent_Populate(lodepng)
endif()
add_library(lodepng STATIC ${lodepng_SOURCE_DIR}/lodepng.cpp)
set_target_properties(lodepng PROPERTIES POSITION_INDEPENDENT_CODE ON)
target_include_directories(lodepng PUBLIC $<BUILD_INTERFACE:${lodepng_SOURCE_DIR}>)
install(TARGETS lodepng ARCHIVE DESTINATION opt/${PROJECT_NAME}/lib)
install(FILES ${lodepng_SOURCE_DIR}/lodepng.h DESTINATION opt/${PROJECT_NAME}/include)

find_package(glfw3 REQUIRED)
add_library(mujoco_simulate STATIC
  ${_SIM_DIR}/platform_ui_adapter.cc
  ${_SIM_DIR}/glfw_adapter.cc
  ${_SIM_DIR}/glfw_dispatch.cc
  ${_SIM_DIR}/simulate.cc
)
set_target_properties(mujoco_simulate PROPERTIES POSITION_INDEPENDENT_CODE ON)
target_compile_features(mujoco_simulate PUBLIC cxx_std_17)
target_include_directories(mujoco_simulate PUBLIC
  $<BUILD_INTERFACE:${_SIM_DIR}>
  $<BUILD_INTERFACE:${MUJOCO_ROOT_DIR}/include>
)
target_link_libraries(mujoco_simulate PUBLIC mujoco glfw PRIVATE lodepng)
target_compile_options(mujoco_simulate PRIVATE
  -Wno-missing-field-initializers -Wno-unused-parameter -Wno-sign-compare -Wno-psabi
  -Wno-shadow -Wno-float-equal
)
install(TARGETS mujoco_simulate ARCHIVE DESTINATION opt/${PROJECT_NAME}/lib)
install(DIRECTORY ${_SIM_DIR}/ DESTINATION opt/${PROJECT_NAME}/include/simulate
  FILES_MATCHING PATTERN "*.h"
)

ament_export_targets(export_${PROJECT_NAME})
ament_export_include_directories(opt/${PROJECT_NAME}/include)
ament_environment_hooks("${CMAKE_CURRENT_SOURCE_DIR}/env-hooks/${PROJECT_NAME}.dsv")

ament_package(
  CONFIG_EXTRAS_POST "mujoco_vendor-extras.cmake"
)
