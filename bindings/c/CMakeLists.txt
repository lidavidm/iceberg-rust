# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.

cmake_minimum_required(VERSION 3.24)
cmake_policy(SET CMP0135 NEW)

# We do not enable C/CXX languages - Cargo doesn't need them(?)
project(iceberg-c VERSION "0.1.0")

include(CMakePackageConfigHelpers)
include(GNUInstallDirs)

# ------------------------------------------------------------
# Version definitions

# If we want to use -SNAPSHOT etc, strip that
string(REGEX MATCH "^[0-9]+\\.[0-9]+\\.[0-9]+" ICEBERG_C_BASE_VERSION "${PROJECT_VERSION}")
string(REPLACE "." ";" _iceberg_c_version_list "${ICEBERG_C_BASE_VERSION}")
list(GET _iceberg_c_version_list 0 ICEBERG_C_VERSION_MAJOR)
list(GET _iceberg_c_version_list 1 ICEBERG_C_VERSION_MINOR)
list(GET _iceberg_c_version_list 2 ICEBERG_C_VERSION_PATCH)

math(EXPR ICEBERG_C_SO_VERSION "${ICEBERG_C_VERSION_MAJOR} * 100 + ${ICEBERG_C_VERSION_MINOR}")
set(ICEBERG_C_FULL_SO_VERSION "${ICEBERG_C_SO_VERSION}.${ICEBERG_C_VERSION_PATCH}.0")

# ------------------------------------------------------------
# Set up build variables

# TODO: make Cargo build into `build/` to be self-contained
set(CARGO_DIST_DIR "${PROJECT_SOURCE_DIR}/target/debug")
if (NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CARGO_BUILD_TYPE "--release")
    set(CARGO_DIST_DIR "${PROJECT_SOURCE_DIR}/target/release")
endif()

# We want to install libiceberg.so.1.0.0 (the actual binary), libiceberg.so.1
# (a symlink), and libiceberg.so (another symlink).  But cargo will build the
# actual binary to libiceberg.so.  So we'll invoke cargo build, then copy the
# binary to the "correct" name, then finally generate the symlinks.  We'll
# install from the copy, not the "original" files.

set(ICEBERG_C_STATIC_LIB "${CMAKE_STATIC_LIBRARY_PREFIX}iceberg_c${CMAKE_STATIC_LIBRARY_SUFFIX}")
set(ICEBERG_C_SHARED_LIB "${CMAKE_SHARED_LIBRARY_PREFIX}iceberg_c${CMAKE_SHARED_LIBRARY_SUFFIX}")
set(ICEBERG_C_HEADER "${PROJECT_SOURCE_DIR}/include/iceberg.h")
set(ICEBERG_C_INCLUDEDIR "${CMAKE_INSTALL_INCLUDEDIR}/iceberg-c")
set(ICEBERG_C_LIBDIR "${CMAKE_INSTALL_LIBDIR}")

# ------------------------------------------------------------
# Set up build command

add_custom_command(
  OUTPUT
  "${CARGO_DIST_DIR}/${ICEBERG_C_STATIC_LIB}"
  "${CARGO_DIST_DIR}/${ICEBERG_C_SHARED_LIB}"
  "${ICEBERG_C_HEADER}"
  COMMAND cargo build ${CARGO_BUILD_TYPE}
  WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
  COMMENT "Building Iceberg C bindings"
)
add_custom_command(
  OUTPUT
  "${CMAKE_CURRENT_BINARY_DIR}/${ICEBERG_C_SHARED_LIB}"
  "${CMAKE_CURRENT_BINARY_DIR}/${ICEBERG_C_SHARED_LIB}.${ICEBERG_C_SO_VERSION}"
  "${CMAKE_CURRENT_BINARY_DIR}/${ICEBERG_C_SHARED_LIB}.${ICEBERG_C_FULL_SO_VERSION}"
  DEPENDS
  "${CARGO_DIST_DIR}/${ICEBERG_C_SHARED_LIB}"
  COMMAND ${CMAKE_COMMAND} -E copy "${CARGO_DIST_DIR}/${ICEBERG_C_SHARED_LIB}" "${ICEBERG_C_SHARED_LIB}.${ICEBERG_C_FULL_SO_VERSION}"
  COMMAND ${CMAKE_COMMAND} -E create_symlink "${ICEBERG_C_SHARED_LIB}.${ICEBERG_C_FULL_SO_VERSION}" "${ICEBERG_C_SHARED_LIB}.${ICEBERG_C_SO_VERSION}"
  COMMAND ${CMAKE_COMMAND} -E create_symlink "${ICEBERG_C_SHARED_LIB}.${ICEBERG_C_SO_VERSION}" "${ICEBERG_C_SHARED_LIB}"
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
)

add_custom_target(iceberg_c_target ALL
  DEPENDS
  "${CARGO_DIST_DIR}/${ICEBERG_C_STATIC_LIB}"
  "${CMAKE_CURRENT_BINARY_DIR}/${ICEBERG_C_SHARED_LIB}"
  "${CMAKE_CURRENT_BINARY_DIR}/${ICEBERG_C_SHARED_LIB}.${ICEBERG_C_SO_VERSION}"
  "${CMAKE_CURRENT_BINARY_DIR}/${ICEBERG_C_SHARED_LIB}.${ICEBERG_C_FULL_SO_VERSION}"
  "${ICEBERG_C_HEADER}"
)

# ------------------------------------------------------------
# Set up CMake targets

add_library(iceberg_c_shared SHARED IMPORTED GLOBAL)
set_target_properties(iceberg_c_shared PROPERTIES
  IMPORTED_LOCATION "${CMAKE_CURRENT_BINARY_DIR}/${ICEBERG_C_SHARED_LIB}.${ICEBERG_C_FULL_SO_VERSION}"
  IMPORTED_SONAME "${ICEBERG_C_SHARED_LIB}"
)
add_dependencies(iceberg_c_shared iceberg_c_target)

add_library(iceberg_c_static STATIC IMPORTED GLOBAL)
set_target_properties(iceberg_c_static PROPERTIES
  IMPORTED_LOCATION "${CARGO_DIST_DIR}/${ICEBERG_C_STATIC_LIB}"
)
add_dependencies(iceberg_c_static iceberg_c_target)

# These IcebergC:: prefixed targets are defined in our
# IcebergCConfig.cmake. When using FetchContent, however, that file is not
# evaluated, so for consistency define the the prefixed versions here too,
# even though we don't use them here.

add_library(IcebergC::iceberg_c_shared ALIAS iceberg_c_shared)
add_library(IcebergC::iceberg_c_static ALIAS iceberg_c_static)

# ------------------------------------------------------------
# Install artifacts

install(IMPORTED_RUNTIME_ARTIFACTS
  iceberg_c_shared
  ${INSTALL_IS_OPTIONAL}
  RUNTIME
  DESTINATION
  ${RUNTIME_INSTALL_DIR}
  LIBRARY
  DESTINATION
  ${CMAKE_INSTALL_LIBDIR})

install(FILES "${CARGO_DIST_DIR}/${ICEBERG_C_STATIC_LIB}" TYPE LIB)
install(FILES "${ICEBERG_C_HEADER}" DESTINATION "${ICEBERG_C_INCLUDEDIR}")

if(WIN32)
  install(FILES "${ICEBERG_C_SHARED_LIB}.${ICEBERG_C_SO_VERSION}" TYPE BIN)
else()
  install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/${ICEBERG_C_SHARED_LIB}"
    "${CMAKE_CURRENT_BINARY_DIR}/${ICEBERG_C_SHARED_LIB}.${ICEBERG_C_SO_VERSION}"
    TYPE LIB)
endif()

# ------------------------------------------------------------
# Install pkg-config helpers

configure_file(iceberg-c.pc.in "${CMAKE_CURRENT_BINARY_DIR}/iceberg-c.pc" @ONLY)
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/iceberg-c.pc"
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig/")

# ------------------------------------------------------------
# Install CMake find_package helpers
# This is so you can call find_package(IcebergC)

set(PACKAGE_NAME "IcebergC")
set(CONFIG_CMAKE "${PACKAGE_NAME}Config.cmake")
set(BUILT_CONFIG_CMAKE "${CMAKE_CURRENT_BINARY_DIR}/${CONFIG_CMAKE}")
configure_package_config_file("${CONFIG_CMAKE}.in" "${BUILT_CONFIG_CMAKE}"
  INSTALL_DESTINATION "Iceberg/${PACKAGE_NAME}")
set(CONFIG_VERSION_CMAKE "${PACKAGE_NAME}ConfigVersion.cmake")
set(BUILT_CONFIG_VERSION_CMAKE "${CMAKE_CURRENT_BINARY_DIR}/${CONFIG_VERSION_CMAKE}")
write_basic_package_version_file("${BUILT_CONFIG_VERSION_CMAKE}"
  COMPATIBILITY SameMajorVersion)
set(INSTALL_CMAKEDIR "${CMAKE_INSTALL_LIBDIR}/cmake/${PACKAGE_NAME}")
install(FILES "${BUILT_CONFIG_CMAKE}" "${BUILT_CONFIG_VERSION_CMAKE}"
  DESTINATION "${INSTALL_CMAKEDIR}")
if(EXPORT_NAME)
  set(TARGETS_CMAKE "${PACKAGE_NAME}Targets.cmake")
  install(EXPORT ${EXPORT_NAME}
    DESTINATION "${INSTALL_CMAKEDIR}"
    NAMESPACE "${PACKAGE_NAME}::"
    FILE "${TARGETS_CMAKE}")
endif()
