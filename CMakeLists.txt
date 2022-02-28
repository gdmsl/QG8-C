#
# CMakeLists.txt
#
# Author       : Guido Masella (guido.masella@gmail.com)
# Date created : 2022/02/22
#

cmake_minimum_required(VERSION 3.12)

# Let's check if the library is being compiled as a master project or included
# in another project, for example via the add_subdirectory function
if (NOT DEFINED QG8_MASTER_PROJECT)
  set(QG8_MASTER_PROJECT OFF)
  if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
    set(QG8_MASTER_PROJECT ON)
    message(STATUS "Cmake version: ${CMAKE_VERSION}")
  endif()
endif()

# The default build type is set to Release if the library is compiled by
# itself, otherwise the settings from the master project or the ones
# given at generation time will be respected
if (QG8_MASTER_PROJECT AND NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif ()

# Joins arguments and places the results in ${result_var}.
function(join result_var)
  set(result "")
  foreach (arg ${ARGN})
    set(result "${result}${arg}")
  endforeach ()
  set(${result_var} "${result}" PARENT_SCOPE)
endfunction()

project(qg8 C)

include(GNUInstallDirs)

set(QG8_INC_DIR ${CMAKE_INSTALL_INCLUDEDIR})

option(QG8_PEDANTIC "Enable extra warnings and expensive tests." OFF)
option(QG8_WERROR "Halt the compilation with an error on compiler warnings." OFF)
# flags are not set now but later in target_compile_features only in
# case the related option is activated
# flags are not set now but later in target_compile_features only in
# case the related option is activated
if (CMAKE_CXX_COMPILER_ID MATCHES "GNU")
  set(WERROR_COMPILE_FLAGS -Werror)
  set(PEDANTIC_COMPILE_FLAGS
    -pedantic-errors -Wall -Wpedantic -Wextra -pedantic -Wundef)
endif ()
if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  set(WERROR_COMPILE_FLAGS -Werror)
  set(PEDANTIC_COMPILE_FLAGS
    -Wall -Wextra -pedantic -Wundef -Wdeprecated)
endif ()
if (MSVC)
  set(WERROR_COMPILE_FLAGS /WX)
  set(PEDANTIC_COMPILE_FLAGS /W3)
endif ()
  
# control whether extra targets must be generated
option(QG8_INSTALL "Generate the install target." ${QG8_MASTER_PROJECT})
option(QG8_TEST "Generate the test target." ${QG8_MASTER_PROJECT})

# Get version from qg8.h by regex matching.
# TODO use git tags and git commit hashes for the library version since
# QG8_VERSION indicates the version of the standard, not the library
file(READ include/qg8.h qg8_h)
if (NOT qg8_h MATCHES "QG8_VERSION\ +([0-9]+)")
  message(FATAL_ERROR "Cannot get QG8_VERSION from qg8.h.")
endif ()
# Use math to skip leading zeros if any.
math(EXPR CPACK_PACKAGE_VERSION_MAJOR ${CMAKE_MATCH_1})
join(QG8_VERSION ${CPACK_PACKAGE_VERSION_MAJOR})
message(STATUS "Version: ${QG8_VERSION}")

message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")

function(add_headers VAR)
  set(headers ${${VAR}})
  foreach (header ${ARGN})
    set(headers ${headers} include/${header})
  endforeach()
  set(${VAR} ${headers} PARENT_SCOPE)
endfunction()

# Define the library, its includes and the needed defines.
add_headers(QG8_HEADERS macros.h qg8.h)

set(QG8_SOURCES src/chunk.c src/file.c src/graph.c src/iter.c src/macros.c
  src/tensor.c)

add_library(qg8 ${QG8_SOURCES} ${QG8_HEADERS})
add_library(qg8::qg8 ALIAS qg8)

target_include_directories(qg8 PUBLIC
  $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:${QG8_INC_DIR}>)

# enable pedantic compile flags if enabled in options
if (QG8_PEDANTIC)
  target_compile_options(qg8 PRIVATE ${PEDANTIC_COMPILE_FLAGS})
endif ()

# enable Werror flag which will make the compilation fail if one or more
# warnings are generated
if (QG8_WERROR)
  target_compile_options(qg8 PRIVATE ${WERROR_COMPILE_FLAGS})
endif ()

# the library is written in C89 standard so we require that the compiler
# at least support all the feature of that standard
target_compile_features(qg8 INTERFACE c_std_90)

# set USE_GSL to enable build of extra functions that rely on GSL
# -- not currently  implemented --#
if (USE_GSL)
  find_package(GSL)
  target_link_libraries(qg8 GSL::gsl)
  target_compile_definitions(qg8 PRIVATE USE_GSL=1)
endif()

# set version properties of the target library and flag all the headers as publc
# headers (so they will be installed) or passed to targets linking against us
# SOVERSION is important mainly in the case where shared libraries are being
# generated
set_target_properties(qg8 PROPERTIES
  VERSION ${QG8_VERSION} SOVERSION ${CPACK_PACKAGE_VERSION_MAJOR}
  PUBLIC_HEADER "${QG8_HEADERS}")

# QG8_LIB_NAME for pkg-config qg8.pc
set(QG8_LIB_NAME qg8)

# TODO add here special settings for shared libraries
if (BUILD_SHARED_LIBS)
endif ()

# the install target can be deactivated by the user and it is automatically
# deactivated (see top) if the 
if (QG8_INSTALL)
  include(CMakePackageConfigHelpers)

  set(QG8_CMAKE_DIR ${CMAKE_INSTALL_LIBDIR}/cmake/qg8)

  set(version_config ${PROJECT_BINARY_DIR}/qg8-config-version.cmake)
  set(project_config ${PROJECT_BINARY_DIR}/qg8-config.cmake)
  set(pkgconfig ${PROJECT_BINARY_DIR}/qg8.pc)
  set(targets_export_name qg8-targets)

  set(QG8_LIB_DIR ${CMAKE_INSTALL_LIBDIR})

  set(QG8_PKGCONFIG_DIR ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

  # Generate the version, config and target files into the build directory
  write_basic_package_version_file(
    ${version_config}
    VERSION ${QG8_VERSION}
    COMPATIBILITY AnyNewerVersion)

  configure_file(
    "${PROJECT_SOURCE_DIR}/support/cmake/qg8.pc.in"
    "${pkgconfig}"
    @ONLY)
  configure_package_config_file(
    ${PROJECT_SOURCE_DIR}/support/cmake/qg8-config.cmake.in
    ${project_config}
    INSTALL_DESTINATION ${QG8_CMAKE_DIR})

  set(INSTALL_TARGETS qg8)

  install(TARGETS ${INSTALL_TARGETS} EXPORT ${targets_export_name}
    LIBRARY DESTINATION ${QG8_LIB_DIR}
    ARCHIVE DESTINATION ${QG8_LIB_DIR}
    PUBLIC_HEADER DESTINATION "${QG8_INC_DIR}/qg8"
    FRAMEWORK DESTINATION "."
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

  # Use a namespace because CMake provides better diagnostics for namespaced 
  # imported targets.
  export(TARGETS ${INSTALL_TARGETS} NAMESPACE qg8::
    FILE ${PROJECT_BINARY_DIR}/${targets_export_name}.cmake)

  # Install config and target files
  install(
    FILES ${project_config} ${version_config}
    DESTINATION ${QG8_CMAKE_DIR})
  install(EXPORT ${targets_export_name} DESTINATION ${QG8_CMAKE_DIR}
    NAMESPACE qg8::)
  install(FILES "${pkgconfig}" DESTINATION "${QG8_PKGCONFIG_DIR}")
endif ()

if (QG8_TEST)
  enable_testing()
  add_subdirectory(test)
endif ()
  
# vim: set ft=cmake ts=2 sts=2 et sw=2 tw=80 : #
