cmake_minimum_required(VERSION 3.0)

project(UDT4 VERSION 4.11.0)

# Some boilerplate to setup nice output directories
set(CMAKE_INSTALL_BINDIR bin CACHE STRING "Installation runtime subdirectory")
set(CMAKE_INSTALL_LIBDIR lib CACHE STRING "Installation library subdirectory")
set(CMAKE_INSTALL_INCLUDEDIR include
  CACHE STRING "Installation include subdirectory")
if(WIN32 AND NOT CYGWIN)
  set(CMAKE_INSTALL_CMAKEDIR CMake
    CACHE STRING "Installation CMake subdirectory")
else()
  set(CMAKE_INSTALL_CMAKEDIR ${CMAKE_INSTALL_LIBDIR}/cmake/udt4
    CACHE STRING "Installation CMake subdirectory")
endif()
mark_as_advanced(CMAKE_INSTALL_BINDIR)
mark_as_advanced(CMAKE_INSTALL_LIBDIR)
mark_as_advanced(CMAKE_INSTALL_INCLUDEDIR)
mark_as_advanced(CMAKE_INSTALL_CMAKEDIR)

list(INSERT CMAKE_MODULE_PATH 0 ${PROJECT_SOURCE_DIR}/cmake)
if(NOT CMAKE_ARCHIVE_OUTPUT_DIRECTORY)
  set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY
  ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
endif()
if(NOT CMAKE_LIBRARY_OUTPUT_DIRECTORY)
  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY
  ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
endif()
if(NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY)
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY
  ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_BINDIR})
endif()

list(INSERT CMAKE_PREFIX_PATH 0 ${CMAKE_INSTALL_PREFIX})

# Default to a RelWithDebInfo build if not specified
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY VALUE RelWithDebugInfo)
endif()

include(CMakeDependentOption)
include(CheckEpoll)

# Setup shared library defaults.  If explicitly specified somehow, then default
# to that.  Otherwise base the default on whether or not shared libs are even
# supported.
get_property(SHARED_LIBS_SUPPORTED GLOBAL PROPERTY TARGET_SUPPORTS_SHARED_LIBS)
cmake_dependent_option(BUILD_SHARED_LIBS
  "Build shared libraries (so/dylib/dll)." ${SHARED_LIBS_SUPPORTED}
  "SHARED_LIBS_SUPPORTED" OFF)
mark_as_advanced(BUILD_SHARED_LIBS)

# Setup PIC defaults.  If explicitly specified somehow, then default
# to that.  Otherwise base the default on whether or not shared libs are even
# supported.
if(DEFINED UDT4_ENABLE_PIC)
  set(UDT4_ENABLE_PIC_DEFAULT ${UDT4_ENABLE_PIC})
elseif(DEFINED CMAKE_POSITION_INDEPENDENT_CODE)
  set(UDT4_ENABLE_PIC_DEFAULT ${CMAKE_POSITION_INDEPENDENT_CODE})
else()
  set(UDT4_ENABLE_PIC_DEFAULT ${SHARED_LIBS_SUPPORTED})
endif()
cmake_dependent_option(UDT4_ENABLE_PIC
  "Build with Position Independent Code" ${UDT4_ENABLE_PIC_DEFAULT}
  "SHARED_LIBS_SUPPORTED" OFF)
mark_as_advanced(UDT4_ENABLE_PIC)
set(CMAKE_POSITION_INDEPENDENT_CODE ${UDT4_ENABLE_PIC})
mark_as_advanced(CMAKE_POSITION_INDEPENDENT_CODE)

list(APPEND UDT4_MASTER_SRC_LIST udt4/src/api.cpp udt4/src/buffer.cpp udt4/src/cache.cpp udt4/src/ccc.cpp udt4/src/channel.cpp udt4/src/common.cpp udt4/src/core.cpp udt4/src/epoll.cpp udt4/src/list.cpp udt4/src/md5.cpp udt4/src/packet.cpp udt4/src/queue.cpp udt4/src/window.cpp)

CHECK_EPOLL(HAS_EPOLL)
set(UDT4_LIBRARY_PREFIX "" CACHE STRING
  "Prefix to prepend to the output library name")
mark_as_advanced(UDT4_LIBRARY_PREFIX)

add_library(udt4 ${UDT4_MASTER_SRC_LIST})
set_target_properties(udt4 PROPERTIES
  OUTPUT_NAME ${UDT4_LIBRARY_PREFIX}udt4
  VERSION ${UDT4_VERSION}
  SOVERSION ${UDT4_VERSION_MAJOR})
add_library(udt4::udt4 ALIAS udt4)

if(UNIX)
  target_link_libraries(udt4 PRIVATE m)
  list(APPEND _pkg_config_private_libs m)
endif()
if(SHARED_LIBS_SUPPORTED)
  unset(NO_DYNAMIC_LINKING)
  target_link_libraries(udt4 PRIVATE ${CMAKE_DL_LIBS})
  list(APPEND _pkg_config_private_libs ${CMAKE_DL_LIBS})
else()
  set(NO_DYNAMIC_LINKING TRUE)
endif()

target_include_directories(udt4 PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/udt4>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/udt4>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)

# Setup pkgconfig
set(__pkg_config_private_libs ${_pkg_config_private_libs})
set(_pkg_config_private_libs)
foreach(L ${__pkg_config_private_libs})
  if(L MATCHES "(.*)/?lib(.*)\\.")
    if(CMAKE_MATCH_1)
      list(APPEND _pkg_config_private_libs "-L${CMAKE_MATCH_1}")
    endif()
    list(APPEND _pkg_config_private_libs "-l${CMAKE_MATCH_2}")
  elseif(L MATCHES "^-")
    list(APPEND _pkg_config_private_libs "${L}")
  else()
    list(APPEND _pkg_config_private_libs "-l${L}")
  endif()
endforeach()
string(REPLACE ";" " " _pkg_config_private_libs "${_pkg_config_private_libs}")
string(REPLACE ";" " " _pkg_config_private_reqs "${_pkg_config_private_reqs}")
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/config.h.cmake
  ${CMAKE_CURRENT_BINARY_DIR}/config.h)
configure_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/udt4.pc.in
  ${CMAKE_CURRENT_BINARY_DIR}/udt4.pc
  @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/udt4.pc
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")
configure_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/udt4-config.in
  ${CMAKE_CURRENT_BINARY_DIR}/udt4-config
  @ONLY)
install(PROGRAMS ${CMAKE_CURRENT_BINARY_DIR}/udt4-config
  DESTINATION "${CMAKE_INSTALL_BINDIR}")

install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/udt4/src/udt.h
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

install(TARGETS udt4 EXPORT udt4-targets
  RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}" COMPONENT bin
  LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}" COMPONENT shlib
  ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}" COMPONENT lib
  PUBLIC_HEADER DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/udt4" COMPONENT dev)

if(CMAKE_C_COMPILER_ID MATCHES Intel)
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -shared-intel")
endif()

# Add all targets to the build-tree export set
export(TARGETS udt4 NAMESPACE udt4::
  FILE "${PROJECT_BINARY_DIR}/udt4-targets.cmake")

# Create the udt4-config.cmake and udt4-config-version files
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  "${PROJECT_BINARY_DIR}/udt4-config-version.cmake"
  COMPATIBILITY SameMajorVersion)
configure_file(udt4-config.cmake.in
  "${PROJECT_BINARY_DIR}/udt4-config.cmake" @ONLY)

# Install the udt4-config.cmake and udt4-config-version.cmake
install(
  FILES
    "${PROJECT_BINARY_DIR}/udt4-config.cmake"
    "${PROJECT_BINARY_DIR}/udt4-config-version.cmake"
  DESTINATION "${CMAKE_INSTALL_CMAKEDIR}" COMPONENT dev)

# Install the export set for use with the install-tree
install(EXPORT udt4-targets NAMESPACE udt4::
  DESTINATION "${CMAKE_INSTALL_CMAKEDIR}" COMPONENT dev)

# display status message for important variables
option(UDT4_LIBRARIES_ONLY "Whether or not to build the associated executables"
  OFF)
option(UDT4_QUIET "Suppress summary output" OFF)
if(NOT UDT4_QUIET)
  message(STATUS)
  message(STATUS "-----------------------------------------------------------------------------")
  message(STATUS "CMAKE_INSTALL_PREFIX = ${CMAKE_INSTALL_PREFIX}")
  message(STATUS "CMAKE_BUILD_TYPE = ${CMAKE_BUILD_TYPE}")
  message(STATUS "UDT4_USE_DILL = ${UDT4_USE_DILL}")
  message(STATUS "UDT4_USE_ATL = ${UDT4_USE_ATL}")
  message(STATUS "UDT4_LIBRARIES_ONLY = ${UDT4_LIBRARIES_ONLY}")
  message(STATUS "Change a value with: cmake -D<Variable>=<Value>")
  message(STATUS "-----------------------------------------------------------------------------")
endif()
