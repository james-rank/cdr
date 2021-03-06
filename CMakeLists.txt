#
# Copyright 2014-2018 Neueda Ltd.
#
cmake_minimum_required(VERSION 2.8.0)
project(cdr CXX)

if(PYTHON_LIBRARY AND PYTHON_INCLUDE_DIR AND PYTHON_EXECUTABLE)
    set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DPYTHON_LIBRARY=${PYTHON_LIBRARY})
    set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DPYTHON_INCLUDE_DIR=${PYTHON_INCLUDE_DIR})
    set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DPYTHON_EXECUTABLE=${PYTHON_EXECUTABLE})
endif()

if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set (CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}/install" CACHE PATH "default install path" FORCE )
endif()

if(WIN32)
  set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
  set(BUILD_SHARED_LIBS TRUE)
  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MT")
  set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd")
elseif(APPLE)
  set(CMAKE_XCODE_ATTRIBUTE_CLANG_CXX_LIBRARY "libc++")
  set(CMAKE_XCODE_ATTRIBUTE_GCC_VERSION "com.apple.compilers.llvm.clang.1_0")
  set(CMAKE_SHARED_LINKER_FLAGS "-Wl,-undefined,dynamic_lookup")
  set(CMAKE_MACOSX_RPATH 1)
endif()

# options
option(DEBUG "Enable debug build" OFF)
option(JAVA "Enable Java bindings" OFF)
option(PYTHON "Enable Python bindings" OFF)
option(CSHARP "Enable C# bindings" OFF)
option(COVERAGE "Enable gcov coverage" OFF)
option(JSON "Enable json encode/decode support" OFF)
option(EXAMPLES "Enable examples" ON)
set(PYTHON_CONFIG "python-config" CACHE STRING "python-config for build env config")

if(COVERAGE)
    set(COVERAGE_COMPILE_FLAGS "-g -O0 -coverage -fprofile-arcs -ftest-coverage")
    set(COVERAGE_LINK_FLAGS    "-coverage -lgcov")
    set(CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} ${COVERAGE_COMPILE_FLAGS}" )
    set(CMAKE_EXE_LINKER_FLAGS  "${CMAKE_EXE_LINKER_FLAGS} ${COVERAGE_LINK_FLAGS}" ) 
endif(COVERAGE)

if(JSON)
    add_definitions(-DJSON)

    find_package(PkgConfig REQUIRED)
    pkg_check_modules(JSONCPP jsoncpp)

    if (JSONCPP_LIBRARIES AND JSONCPP_INCLUDE_DIRS)
        message(STATUS "jsoncpp-found: " ${JSONCPP_LIBRARIES})
        message(STATUS "jsoncpp-include-dirs: " ${JSONCPP_INCLUDE_DIRS})
    else()
        message(FATAL_ERROR "jsoncpp-not-found")
    endif()
endif(JSON)

# set version info
set(CDR_MAJOR_VERSION 0)
set(CDR_MINOR_VERSION 1)
set(CDR_PATCH_VERSION 0)
set(CDR_VERSION ${CDR_MAJOR_VERSION}.${CDR_MINOR_VERSION}.${CDR_PATCH_VERSION})

if (UNIX)
  macro (add_library _name)
    _add_library(${ARGV})
    if (NOT APPLE AND TARGET ${_name})
      SET_TARGET_PROPERTIES(${_name}
        PROPERTIES
            VERSION ${CDR_VERSION}
            # SOVERSION 0.0.0
        )
    endif()
  endmacro()
endif (UNIX)

# compile options
add_compile_options(-Wall)
if (DEBUG)
  if(UNIX)
	  add_compile_options(-g3)
  endif()
  message(STATUS "cdr DEBUG: ON")
  set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DDEBUG=on)
else ()
  message(STATUS "cdr DEBUG: OFF")
  if(UNIX)
	  add_compile_options(-g -O2)
  endif()
endif (DEBUG)

# allow external projects
include(${CMAKE_ROOT}/Modules/ExternalProject.cmake)

# add source
add_subdirectory(src)

# unit-tests
option(TESTS "Enable unit-tests" OFF)
if(TESTS)
  if(WIN32)
    ExternalProject_Add(
      googletest
      SOURCE_DIR ${PROJECT_SOURCE_DIR}/ext/googletest
      CMAKE_ARGS -DCMAKE_CXX_FLAGS="/w" -DBUILD_GMOCK=ON -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}/googletest -DCMAKE_INSTALL_LIBDIR=${CMAKE_BINARY_DIR}/googletest/lib
      )
  else()
    ExternalProject_Add(
      googletest
      SOURCE_DIR ${PROJECT_SOURCE_DIR}/ext/googletest
      CMAKE_ARGS -DBUILD_GMOCK=ON -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}/googletest -DCMAKE_INSTALL_LIBDIR=${CMAKE_BINARY_DIR}/googletest/lib
      )
  endif()

  include(CTest)
  enable_testing()
  
  # add test dir
  add_subdirectory(test)
endif()

if(EXAMPLES)
    add_subdirectory(examples)
endif()

SET(CPACK_GENERATOR "TGZ")
set(CPACK_PACKAGE_VERSION_MAJOR "${CDR_MAJOR_VERSION}")
set(CPACK_PACKAGE_VERSION_MINOR "${CDR_MINOR_VERSION}")
set(CPACK_PACKAGE_VERSION_PATCH "${CDR_PATCH_VERSION}")
set(CPACK_PACKAGE_NAME "cdr")
set(CPACK_PACKAGE_VERSION "${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}")
if(DEBUG)
    set(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME}_dbg-${CPACK_PACKAGE_VERSION}")
else()
    set(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}")
endif()

INCLUDE(CPack)
