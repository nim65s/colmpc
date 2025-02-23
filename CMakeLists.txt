cmake_minimum_required(VERSION 3.10)

# Project properties
set(PROJECT_NAME colmpc)
set(PROJECT_DESCRIPTION "Collision avoidance for MPC")
set(PROJECT_URL https://github.com/AarthuH91/${PROJECT_NAME})

# Project options
option(BUILD_PYTHON_INTERFACE "Build the python binding" ON)
option(SUFFIX_SO_VERSION "Suffix library name with its version" ON)
option(GENERATE_PYTHON_STUBS
       "Generate the Python stubs associated to the Python library" OFF)

# Project configuration
set(PROJECT_USE_CMAKE_EXPORT TRUE)

# Check if the submodule cmake have been initialized
set(JRL_CMAKE_MODULES "${CMAKE_CURRENT_LIST_DIR}/cmake")
if(NOT EXISTS "${CMAKE_SOURCE_DIR}/cmake/base.cmake")
  if(${CMAKE_VERSION} VERSION_LESS "3.14.0")
    message(
      FATAL_ERROR
        "\nPlease run the following command first:\ngit submodule update --init\n"
    )
  else()
    include(FetchContent)
    FetchContent_Declare(
      "jrl-cmakemodules"
      GIT_REPOSITORY "https://github.com/jrl-umi3218/jrl-cmakemodules.git")
    FetchContent_MakeAvailable("jrl-cmakemodules")
    FetchContent_GetProperties("jrl-cmakemodules" SOURCE_DIR JRL_CMAKE_MODULES)
  endif()
endif()

# JRL-cmakemodule setup
include("${JRL_CMAKE_MODULES}/base.cmake")
include("${JRL_CMAKE_MODULES}/boost.cmake")
include("${JRL_CMAKE_MODULES}/stubs.cmake")
include("${JRL_CMAKE_MODULES}/apple.cmake")

# Project definition
compute_project_args(PROJECT_ARGS LANGUAGES CXX)
project(${PROJECT_NAME} ${PROJECT_ARGS})

add_compile_options(-Wno-error)

apply_default_apple_configuration()

# Project dependencies
if(BUILD_PYTHON_INTERFACE)
  add_project_dependency(eigenpy 3.0.0 REQUIRED)
  string(REGEX REPLACE "-" "_" PY_NAME ${PROJECT_NAME})
  set(${PY_NAME}_INSTALL_DIR ${PYTHON_SITELIB})
endif()

add_project_dependency(crocoddyl REQUIRED)

# Main Library
set(${PROJECT_NAME}_cpp_HEADERS
    include/${PROJECT_NAME}/fwd.hpp
    include/${PROJECT_NAME}/residual-distance-collision.hpp
    include/${PROJECT_NAME}/residual-distance-collision.hxx)

add_library(${PROJECT_NAME}_cpp SHARED ${${PROJECT_NAME}_cpp_HEADERS})
set_target_properties(${PROJECT_NAME}_cpp PROPERTIES LINKER_LANGUAGE CXX)
target_include_directories(${PROJECT_NAME}_cpp
                           PUBLIC $<INSTALL_INTERFACE:include>)
target_link_libraries(${PROJECT_NAME}_cpp PUBLIC crocoddyl::crocoddyl)
if(UNIX)
  get_relative_rpath(${CMAKE_INSTALL_LIBDIR} ${PROJECT_NAME}_cpp_INSTALL_RPATH)
  set_target_properties(
    ${PROJECT_NAME}_cpp PROPERTIES INSTALL_RPATH
                                   "${${PROJECT_NAME}_cpp_INSTALL_RPATH}")
endif()

if(SUFFIX_SO_VERSION)
  set_target_properties(${PROJECT_NAME}_cpp PROPERTIES SOVERSION
                                                       ${PROJECT_VERSION})
endif()

# Python Bindings
if(BUILD_PYTHON_INTERFACE)
  add_subdirectory(python)
endif()

# # Unit tests if(BUILD_TESTING) find_package(Boost REQUIRED COMPONENTS
# unit_test_framework) add_subdirectory(tests) endif()

# Installation
install(
  TARGETS ${PROJECT_NAME}_cpp
  EXPORT ${TARGETS_EXPORT_NAME}
  DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(FILES package.xml DESTINATION share/${PROJECT_NAME})
