# Open Asset Import Library (assimp)
# ----------------------------------------------------------------------
# Copyright (c) 2006-2022, assimp team
#
# All rights reserved.
#
# Redistribution and use of this software in source and binary forms, with or
# without modification, are permitted provided that the following conditions are
# met:
#
# * Redistributions of source code must retain the above copyright notice, this
#   list of conditions and the following disclaimer.
#
# * Redistributions in binary form must reproduce the above copyright notice,
#   this list of conditions and the following disclaimer in the documentation
#   and/or other materials provided with the distribution.
#
# * Neither the name of the assimp team, nor the names of its contributors may
#   be used to endorse or promote products derived from this software without
#   specific prior written permission of the assimp team.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
# AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
# DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE LIABLE
# FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
# DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
# SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
# CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
# OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
# ----------------------------------------------------------------------
set(CMAKE_POLICY_DEFAULT_CMP0012 NEW)
set(CMAKE_POLICY_DEFAULT_CMP0074 NEW)
set(CMAKE_POLICY_DEFAULT_CMP0092 NEW)

cmake_minimum_required(VERSION 3.10)

# Disabled importers: m3d for 5.1
add_definitions(-DASSIMP_BUILD_NO_M3D_IMPORTER)
add_definitions(-DASSIMP_BUILD_NO_M3D_EXPORTER)
# Toggles the use of the hunter package manager
option(ASSIMP_HUNTER_ENABLED "Enable Hunter package manager support" OFF)

if(ASSIMP_HUNTER_ENABLED)
  include("cmake-modules/HunterGate.cmake")
  huntergate(URL "https://github.com/cpp-pm/hunter/archive/v0.24.0.tar.gz" SHA1
             "a3d7f4372b1dcd52faa6ff4a3bd5358e1d0e5efd")

  add_definitions(-DASSIMP_USE_HUNTER)
endif()

project(Assimp VERSION 5.2.0)

# All supported options ###############################################

option(BUILD_SHARED_LIBS "Build package with shared libraries." OFF)
option(ASSIMP_BUILD_FRAMEWORK "Build package as Mac OS X Framework bundle." OFF)
option(ASSIMP_DOUBLE_PRECISION
       "Set to ON to enable double precision processing" ON)
option(ASSIMP_OPT_BUILD_PACKAGES
       "Set to ON to generate CPack configuration files and packaging targets"
       OFF)
option(ASSIMP_ANDROID_JNIIOSYSTEM "Android JNI IOSystem support is active" OFF)
option(ASSIMP_NO_EXPORT "Disable Assimp's export functionality." OFF)
option(ASSIMP_BUILD_ZLIB "Build your own zlib" ON)
option(
  ASSIMP_BUILD_ASSIMP_TOOLS
  "If the supplementary tools for Assimp are built in addition to the library."
  OFF)
option(ASSIMP_BUILD_SAMPLES
       "If the official samples are built as well (needs Glut)." OFF)
option(ASSIMP_BUILD_TESTS
       "If the test suite for Assimp is built in addition to the library." OFF)
option(ASSIMP_COVERALLS "Enable this to measure test coverage." OFF)
option(ASSIMP_INSTALL "Disable this if you want to use assimp as a submodule."
       OFF)
option(ASSIMP_WARNINGS_AS_ERRORS "Treat all warnings as errors." ON)
option(ASSIMP_ASAN "Enable AddressSanitizer." OFF)
option(ASSIMP_UBSAN "Enable Undefined Behavior sanitizer." OFF)
option(ASSIMP_BUILD_DOCS "Build documentation using Doxygen." OFF)
option(ASSIMP_INJECT_DEBUG_POSTFIX
       "Inject debug postfix in .a/.so/.dll lib names" ON)

option(ASSIMP_IGNORE_GIT_HASH "Don't call git to get the hash." OFF)

if(WIN32)
  # Use subset of Windows.h
  add_definitions(-DWIN32_LEAN_AND_MEAN)

  if(MSVC)
    option(ASSIMP_INSTALL_PDB "Install MSVC debug files." ON)
    if(NOT (MSVC_VERSION LESS 1900))
      # Multibyte character set is deprecated since at least MSVC2015 (possibly
      # earlier)
      add_definitions(-DUNICODE -D_UNICODE)
    endif()

    # Link statically against c/c++ lib to avoid missing redistriburable such as
    # "VCRUNTIME140.dll not found. Try reinstalling the app.", but give users a
    # choice to opt for the shared runtime if they want.
    option(USE_STATIC_CRT "Link against the static runtime libraries." OFF)

    # The CMAKE_CXX_FLAGS vars can be overriden by some Visual Studio
    # generators, so we use an alternative global method here:
    if(${USE_STATIC_CRT})
      add_compile_options($<$<CONFIG:>:/MT> $<$<CONFIG:Debug>:/MTd>
                          $<$<CONFIG:Release>:/MT>)
    endif()
  endif()
endif()

if(IOS AND NOT ASSIMP_HUNTER_ENABLED)
  if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release")
  endif()
  add_definitions(-DENABLE_BITCODE)
endif()

if(ASSIMP_BUILD_FRAMEWORK)
  set(BUILD_SHARED_LIBS ON)
  message(STATUS "Framework bundle building enabled")
endif()

if(NOT BUILD_SHARED_LIBS)
  message(STATUS "Shared libraries disabled")
  set(LINK_SEARCH_START_STATIC TRUE)
  set(CMAKE_FIND_LIBRARY_SUFFIXES ${CMAKE_STATIC_LIBRARY_SUFFIX}
                                  ${CMAKE_FIND_LIBRARY_SUFFIXES})
else()
  message(STATUS "Shared libraries enabled")
endif()

# Define here the needed parameters
set(ASSIMP_VERSION_MAJOR ${PROJECT_VERSION_MAJOR})
set(ASSIMP_VERSION_MINOR ${PROJECT_VERSION_MINOR})
set(ASSIMP_VERSION_PATCH ${PROJECT_VERSION_PATCH})
set(ASSIMP_VERSION
    ${ASSIMP_VERSION_MAJOR}.${ASSIMP_VERSION_MINOR}.${ASSIMP_VERSION_PATCH})
set(ASSIMP_SOVERSION 5)

set(ASSIMP_PACKAGE_VERSION
    "0"
    CACHE STRING "the package-specific version used for uploading the sources")
if(NOT ASSIMP_HUNTER_ENABLED)
  # Enable C++11 support globally
  set(CMAKE_CXX_STANDARD 11)
  set(CMAKE_CXX_STANDARD_REQUIRED ON)
  set(CMAKE_C_STANDARD 99)
endif()

if(NOT ASSIMP_IGNORE_GIT_HASH)
  # Get the current working branch
  execute_process(
    COMMAND git rev-parse --abbrev-ref HEAD
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_BRANCH
    OUTPUT_STRIP_TRAILING_WHITESPACE ERROR_QUIET)

  # Get the latest abbreviated commit hash of the working branch
  execute_process(
    COMMAND git rev-parse --short=8 HEAD
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_COMMIT_HASH
    OUTPUT_STRIP_TRAILING_WHITESPACE ERROR_QUIET)
endif()

if(NOT GIT_COMMIT_HASH)
  set(GIT_COMMIT_HASH 0)
endif()

if(ASSIMP_DOUBLE_PRECISION)
  add_definitions(-DASSIMP_DOUBLE_PRECISION)
endif()

include_directories(BEFORE ./ code/ include ${CMAKE_CURRENT_BINARY_DIR}
                    ${CMAKE_CURRENT_BINARY_DIR}/include)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake-modules")
set(LIBASSIMP_COMPONENT
    "libassimp${ASSIMP_VERSION_MAJOR}.${ASSIMP_VERSION_MINOR}.${ASSIMP_VERSION_PATCH}"
)
set(LIBASSIMP-DEV_COMPONENT
    "libassimp${ASSIMP_VERSION_MAJOR}.${ASSIMP_VERSION_MINOR}.${ASSIMP_VERSION_PATCH}-dev"
)
set(CPACK_COMPONENTS_ALL assimp-bin ${LIBASSIMP_COMPONENT}
                         ${LIBASSIMP-DEV_COMPONENT} assimp-dev)
set(ASSIMP_LIBRARY_SUFFIX
    ""
    CACHE STRING "Suffix to append to library names")

if(UNIX)
  # Use GNUInstallDirs for Unix predefined directories
  include(GNUInstallDirs)
  # Ensure that we do not run into issues like
  # http://www.tcm.phy.cam.ac.uk/sw/inodes64.html on 32 bit linux
  if(${OPERATING_SYSTEM} MATCHES "Android")

  else()
    if(CMAKE_SIZEOF_VOID_P EQUAL 4) # only necessary for 32-bit linux
      add_definitions(-D_FILE_OFFSET_BITS=64)
    endif()
  endif()
endif()

# Grouped compiler settings ########################################
if((CMAKE_C_COMPILER_ID MATCHES "GNU") AND NOT MINGW)
  if(NOT ASSIMP_HUNTER_ENABLED)
    set(CMAKE_CXX_STANDARD 11)
    set(CMAKE_POSITION_INDEPENDENT_CODE ON)
  endif()
  # hide all not-exported symbols
  if(CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "mips64")
    set(CMAKE_CXX_FLAGS
        "-mxgot -fvisibility=hidden -fno-strict-aliasing -Wall ${CMAKE_CXX_FLAGS}"
    )
    set(CMAKE_C_FLAGS "-fno-strict-aliasing ${CMAKE_C_FLAGS}")
    set(LIBSTDC++_LIBRARIES -lstdc++)
  else()
    set(CMAKE_CXX_FLAGS
        "-fvisibility=hidden -fno-strict-aliasing -Wall ${CMAKE_CXX_FLAGS}")
    set(CMAKE_C_FLAGS "-fno-strict-aliasing ${CMAKE_C_FLAGS}")
    set(LIBSTDC++_LIBRARIES -lstdc++)
  endif()
elseif(MSVC)
  # enable multi-core compilation with MSVC
  if(CMAKE_CXX_COMPILER_ID MATCHES "Clang") # clang-cl
    add_compile_options(/bigobj /W4 /WX)
  else() # msvc
    add_compile_options(/MP /bigobj /W4 /WX)
  endif()
  # disable "elements of array '' will be default initialized" warning on
  # MSVC2013
  if(MSVC12)
    add_compile_options(/wd4351)
  endif()
  add_compile_options(/wd4244) # supress warning for double to float conversion
                               # if Double precission is activated
  set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /D_DEBUG /Zi /Od")
  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /Zi")
  set(CMAKE_SHARED_LINKER_FLAGS_RELEASE
      "${CMAKE_SHARED_LINKER_FLAGS_RELEASE} /DEBUG:FULL /PDBALTPATH:%_PDB% /OPT:REF /OPT:ICF"
  )
elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  if(NOT ASSIMP_HUNTER_ENABLED)
    set(CMAKE_CXX_STANDARD 11)
    set(CMAKE_POSITION_INDEPENDENT_CODE ON)
  endif()
  set(CMAKE_CXX_FLAGS
      "-fvisibility=hidden -fno-strict-aliasing -Wall -Wno-long-long ${CMAKE_CXX_FLAGS}"
  )
  set(CMAKE_C_FLAGS "-fno-strict-aliasing ${CMAKE_C_FLAGS}")
elseif(MINGW)
  if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS 7.0)
    message(
      FATAL_ERROR
        "MinGW is too old to be supported. Please update MinGW and try again.")
  elseif(CMAKE_CXX_COMPILER_VERSION VERSION_LESS 7.3)
    message(WARNING "MinGW is old, if you experience errors, update MinGW.")
  endif()
  if(NOT ASSIMP_HUNTER_ENABLED)
    set(CMAKE_CXX_FLAGS "-std=gnu++11 ${CMAKE_CXX_FLAGS}")
    set(CMAKE_C_FLAGS "-fPIC ${CMAKE_C_FLAGS}")
  endif()
  set(CMAKE_CXX_FLAGS
      "-fvisibility=hidden -fno-strict-aliasing -Wall -Wno-long-long -Wa,-mbig-obj -O3 ${CMAKE_CXX_FLAGS}"
  )
  set(CMAKE_C_FLAGS "-fno-strict-aliasing ${CMAKE_C_FLAGS}")
endif()

if(IOS AND NOT ASSIMP_HUNTER_ENABLED)
  if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fembed-bitcode -Og")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fembed-bitcode -Og")
  else()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fembed-bitcode -O3")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fembed-bitcode -O3")
    # Experimental for pdb generation
  endif()
endif()

if(ASSIMP_COVERALLS)
  message(STATUS "Coveralls enabled")
  include(Coveralls)
  set(CMAKE_CXX_FLAGS
      "${CMAKE_CXX_FLAGS} -g -O0 -fprofile-arcs -ftest-coverage")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -O0 -fprofile-arcs -ftest-coverage")
endif()

if(ASSIMP_ASAN)
  message(STATUS "AddressSanitizer enabled")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=address")
endif()

if(ASSIMP_UBSAN)
  message(STATUS "Undefined Behavior sanitizer enabled")
  set(CMAKE_CXX_FLAGS
      "${CMAKE_CXX_FLAGS} -fsanitize=undefined,shift,shift-exponent,integer-divide-by-zero,unreachable,vla-bound,null,return,signed-integer-overflow,bounds,float-divide-by-zero,float-cast-overflow,nonnull-attribute,returns-nonnull-attribute,bool,enum,vptr,pointer-overflow,builtin -fno-sanitize-recover=all"
  )
  set(CMAKE_C_FLAGS
      "${CMAKE_C_FLAGS} -fsanitize=undefined,shift,shift-exponent,integer-divide-by-zero,unreachable,vla-bound,null,return,signed-integer-overflow,bounds,float-divide-by-zero,float-cast-overflow,nonnull-attribute,returns-nonnull-attribute,bool,enum,vptr,pointer-overflow,builtin -fno-sanitize-recover=all"
  )
endif()

include(FindPkgMacros)
include(PrecompiledHeader)

# Set Assimp project output directory variables. Will respect top-level
# CMAKE_*_OUTPUT_DIRECTORY variables if any are set.
if(NOT DEFINED CMAKE_RUNTIME_OUTPUT_DIRECTORY)
  set(ASSIMP_RUNTIME_OUTPUT_DIRECTORY
      "${CMAKE_CURRENT_BINARY_DIR}/bin"
      CACHE STRING "Path for runtime output files")
else()
  set(ASSIMP_RUNTIME_OUTPUT_DIRECTORY
      ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
      CACHE STRING "Path for runtime output files")
endif()

if(NOT DEFINED CMAKE_LIBRARY_OUTPUT_DIRECTORY)
  set(ASSIMP_LIBRARY_OUTPUT_DIRECTORY
      "${CMAKE_CURRENT_BINARY_DIR}/bin"
      CACHE STRING "Path for library output files")
else()
  set(ASSIMP_LIBRARY_OUTPUT_DIRECTORY
      ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}
      CACHE STRING "Path for runtime output files")
endif()

if(NOT DEFINED CMAKE_ARCHIVE_OUTPUT_DIRECTORY)
  set(ASSIMP_ARCHIVE_OUTPUT_DIRECTORY
      "${CMAKE_CURRENT_BINARY_DIR}/lib"
      CACHE STRING "Path for library output files")
else()
  set(ASSIMP_ARCHIVE_OUTPUT_DIRECTORY
      ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY}
      CACHE STRING "Path for runtime output files")
endif()

# Macro used to set the output directories of a target to the respective Assimp
# output directories.
macro(TARGET_USE_COMMON_OUTPUT_DIRECTORY target)
  set_target_properties(
    ${target}
    PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${ASSIMP_RUNTIME_OUTPUT_DIRECTORY}
               LIBRARY_OUTPUT_DIRECTORY ${ASSIMP_LIBRARY_OUTPUT_DIRECTORY}
               ARCHIVE_OUTPUT_DIRECTORY ${ASSIMP_ARCHIVE_OUTPUT_DIRECTORY})
endmacro()

get_cmake_property(is_multi_config GENERATOR_IS_MULTI_CONFIG)

if(ASSIMP_INJECT_DEBUG_POSTFIX AND (is_multi_config OR CMAKE_BUILD_TYPE STREQUAL
                                                       "Debug"))
  set(CMAKE_DEBUG_POSTFIX
      "d"
      CACHE STRING "Debug Postfix for lib, samples and tools")
else()
  set(CMAKE_DEBUG_POSTFIX
      ""
      CACHE STRING "Debug Postfix for lib, samples and tools")
endif()

# Only generate this target if no higher-level project already has
if(NOT TARGET uninstall AND ASSIMP_INSTALL)
  # add make uninstall capability
  configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake-modules/cmake_uninstall.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake" IMMEDIATE @ONLY)
  add_custom_target(
    uninstall "${CMAKE_COMMAND}" -P
              "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")
endif()

# cmake configuration files
if(${BUILD_SHARED_LIBS})
  set(BUILD_LIB_TYPE SHARED)
else()
  set(BUILD_LIB_TYPE STATIC)
endif()

if(UNIX)
  # Use GNUInstallDirs for Unix predefined directories
  include(GNUInstallDirs)

  set(ASSIMP_LIB_INSTALL_DIR ${CMAKE_INSTALL_LIBDIR})
  set(ASSIMP_INCLUDE_INSTALL_DIR ${CMAKE_INSTALL_INCLUDEDIR})
  set(ASSIMP_BIN_INSTALL_DIR ${CMAKE_INSTALL_BINDIR})
else()
  # Cache these to allow the user to override them on non-Unix platforms
  set(ASSIMP_LIB_INSTALL_DIR
      "lib"
      CACHE STRING "Path the built library files are installed to.")
  set(ASSIMP_INCLUDE_INSTALL_DIR
      "include"
      CACHE STRING "Path the header files are installed to.")
  set(ASSIMP_BIN_INSTALL_DIR
      "bin"
      CACHE STRING "Path the tool executables are installed to.")

  set(CMAKE_INSTALL_FULL_INCLUDEDIR
      ${CMAKE_INSTALL_PREFIX}/${ASSIMP_INCLUDE_INSTALL_DIR})
  set(CMAKE_INSTALL_FULL_LIBDIR
      ${CMAKE_INSTALL_PREFIX}/${ASSIMP_LIB_INSTALL_DIR})
  set(CMAKE_INSTALL_FULL_BINDIR
      ${CMAKE_INSTALL_PREFIX}/${ASSIMP_BIN_INSTALL_DIR})
endif()

set(GENERATED_DIR "${CMAKE_CURRENT_BINARY_DIR}/generated")

if(ASSIMP_HUNTER_ENABLED)
  set(CONFIG_INSTALL_DIR "lib/cmake/${PROJECT_NAME}")
  set(CMAKE_CONFIG_TEMPLATE_FILE "cmake-modules/assimp-hunter-config.cmake.in")
  set(NAMESPACE "${PROJECT_NAME}::")
  set(TARGETS_EXPORT_NAME "${PROJECT_NAME}Targets")
  set(VERSION_CONFIG "${GENERATED_DIR}/${PROJECT_NAME}ConfigVersion.cmake")
  set(PROJECT_CONFIG "${GENERATED_DIR}/${PROJECT_NAME}Config.cmake")
else()
  set(CONFIG_INSTALL_DIR
      "${ASSIMP_LIB_INSTALL_DIR}/cmake/assimp-${ASSIMP_VERSION_MAJOR}.${ASSIMP_VERSION_MINOR}"
  )
  set(CMAKE_CONFIG_TEMPLATE_FILE "cmake-modules/assimp-plain-config.cmake.in")
  string(TOLOWER ${PROJECT_NAME} PROJECT_NAME_LOWERCASE)
  set(NAMESPACE "${PROJECT_NAME_LOWERCASE}::")
  set(TARGETS_EXPORT_NAME "${PROJECT_NAME_LOWERCASE}Targets")
  set(VERSION_CONFIG
      "${GENERATED_DIR}/${PROJECT_NAME_LOWERCASE}ConfigVersion.cmake")
  set(PROJECT_CONFIG "${GENERATED_DIR}/${PROJECT_NAME_LOWERCASE}Config.cmake")
endif()

set(INCLUDE_INSTALL_DIR "include")

# Include module with fuction 'write_basic_package_version_file'
include(CMakePackageConfigHelpers)

# Note: PROJECT_VERSION is used as a VERSION
write_basic_package_version_file("${VERSION_CONFIG}"
                                 COMPATIBILITY SameMajorVersion)

configure_package_config_file(${CMAKE_CONFIG_TEMPLATE_FILE} "${PROJECT_CONFIG}"
                              INSTALL_DESTINATION "${CONFIG_INSTALL_DIR}")

install(
  FILES "${PROJECT_CONFIG}" "${VERSION_CONFIG}"
  DESTINATION "${CONFIG_INSTALL_DIR}"
  COMPONENT ${LIBASSIMP-DEV_COMPONENT})

install(
  EXPORT "${TARGETS_EXPORT_NAME}"
  NAMESPACE "${NAMESPACE}"
  DESTINATION "${CONFIG_INSTALL_DIR}"
  COMPONENT ${LIBASSIMP-DEV_COMPONENT})

if(ASSIMP_BUILD_DOCS)
  add_subdirectory(doc)
endif()

# Search for external dependencies, and build them from source if not found
# Search for zlib
if(ASSIMP_HUNTER_ENABLED)
  hunter_add_package(ZLIB)
  find_package(ZLIB CONFIG REQUIRED)

  add_definitions(-DASSIMP_BUILD_NO_OWN_ZLIB)
  set(ZLIB_FOUND TRUE)
  set(ZLIB_LIBRARIES ZLIB::zlib)
  set(ASSIMP_BUILD_MINIZIP TRUE)
else()
  # If the zlib is already found outside, add an export in case assimpTargets
  # can't find it.
  if(ZLIB_FOUND)
    install(TARGETS zlib zlibstatic EXPORT "${TARGETS_EXPORT_NAME}")
  endif()

  if(NOT ASSIMP_BUILD_ZLIB)
    find_package(ZLIB)
  endif()

  if(NOT ZLIB_FOUND)
    message(STATUS "compiling zlib from sources")
    include(CheckIncludeFile)
    include(CheckTypeSize)
    include(CheckFunctionExists)

    # Explicitly turn off ASM686 and AMD64 cmake options. The AMD64 option
    # causes a build failure on MSVC and the ASM builds seem to have problems:
    # https://github.com/madler/zlib/issues/41#issuecomment-125848075 Also
    # prevents these options from "polluting" the cmake options if assimp is
    # being included as a submodule.
    set(ASM686
        FALSE
        CACHE INTERNAL "Override ZLIB flag to turn off assembly" FORCE)
    set(AMD64
        FALSE
        CACHE INTERNAL "Override ZLIB flag to turn off assembly" FORCE)

    # compile from sources
    add_subdirectory(contrib/zlib)
    set(ZLIB_FOUND 1)
    set(ZLIB_LIBRARIES zlibstatic)
    set(ZLIB_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/contrib/zlib
                         ${CMAKE_CURRENT_BINARY_DIR}/contrib/zlib)
    # need to ensure we don't link with system zlib or minizip as well.
    set(ASSIMP_BUILD_MINIZIP 1)
  else()
    add_definitions(-DASSIMP_BUILD_NO_OWN_ZLIB)
    set(ZLIB_LIBRARIES_LINKED -lz)
  endif()
  include_directories(${ZLIB_INCLUDE_DIR})
endif()

if(NOT IOS)
  if(NOT ASSIMP_BUILD_MINIZIP)
    use_pkgconfig(UNZIP minizip)
  endif()
else()
  if(NOT BUILD_SHARED_LIBS)
    if(NOT ASSIMP_BUILD_MINIZIP)
      use_pkgconfig(UNZIP minizip)
    endif()
  endif()
endif()

if(ASSIMP_NO_EXPORT)
  add_definitions(-DASSIMP_BUILD_NO_EXPORT)
  message(STATUS "Build an import-only version of Assimp.")
endif()

set(ASSIMP_BUILD_ARCHITECTURE
    ""
    CACHE STRING "describe the current architecture.")
if(ASSIMP_BUILD_ARCHITECTURE STREQUAL "")

else()
  add_definitions(-D'ASSIMP_BUILD_ARCHITECTURE="${ASSIMP_BUILD_ARCHITECTURE}"')
endif()

# ${CMAKE_GENERATOR}
set(ASSIMP_BUILD_COMPILER
    ""
    CACHE STRING "describe the current compiler.")
if(ASSIMP_BUILD_COMPILER STREQUAL "")

else()
  add_definitions(-D'ASSIMP_BUILD_COMPILER="${ASSIMP_BUILD_COMPILER}"')
endif()

mark_as_advanced(ASSIMP_BUILD_ARCHITECTURE ASSIMP_BUILD_COMPILER)

set(ASSIMP_BUILD_NONFREE_C4D_IMPORTER
    OFF
    CACHE BOOL
          "Build the C4D importer, which relies on the non-free Cineware SDK.")

if(ASSIMP_BUILD_NONFREE_C4D_IMPORTER)
  if(MSVC)
    set(C4D_INCLUDES "${CMAKE_CURRENT_SOURCE_DIR}/contrib/Cineware/includes")

    # pick the correct prebuilt library
    if(MSVC15)
      set(C4D_LIB_POSTFIX "_2017")
    elseif(MSVC14)
      set(C4D_LIB_POSTFIX "_2015")
    elseif(MSVC12)
      set(C4D_LIB_POSTFIX "_2013")
    elseif(MSVC11)
      set(C4D_LIB_POSTFIX "_2012")
    elseif(MSVC10)
      set(C4D_LIB_POSTFIX "_2010")
    else()
      message(
        FATAL_ERROR "C4D is currently only supported with MSVC 10, 11, 12, 14")
    endif()

    set(C4D_LIB_BASE_PATH
        "${CMAKE_CURRENT_SOURCE_DIR}/contrib/Cineware/libraries/win")

    set(C4D_DEBUG_LIBRARIES
        "${C4D_LIB_BASE_PATH}/cinewarelib${C4D_LIB_POSTFIX}/cinewarelib_debug.lib"
        "${C4D_LIB_BASE_PATH}/jpeglib${C4D_LIB_POSTFIX}/jpeglib_debug.lib")
    set(C4D_RELEASE_LIBRARIES
        "${C4D_LIB_BASE_PATH}/cinewarelib${C4D_LIB_POSTFIX}/cinewarelib_release.lib"
        "${C4D_LIB_BASE_PATH}/jpeglib${C4D_LIB_POSTFIX}/jpeglib_release.lib")

    # winsock and winmm are necessary (and undocumented) dependencies of
    # Cineware SDK because it can be used to communicate with a running Cinema
    # 4D instance
    set(C4D_EXTRA_LIBRARIES WSock32.lib Winmm.lib)
  else()
    message(
      FATAL_ERROR
        "C4D is currently only available on Windows with Cineware SDK installed in contrib/Cineware"
    )
  endif()
else()
  add_definitions(-DASSIMP_BUILD_NO_C4D_IMPORTER)
endif()

# Draco requires cmake 3.12
if(DEFINED CMAKE_VERSION AND "${CMAKE_VERSION}" VERSION_LESS "3.12")
  message(
    NOTICE
    "draco requires cmake 3.12 or newer, cmake is ${CMAKE_VERSION} . Draco is disabled"
  )
  set(ASSIMP_BUILD_DRACO
      OFF
      CACHE BOOL "Disabled: Draco requires newer cmake" FORCE)
else()
  option(ASSIMP_BUILD_DRACO
         "If the Draco libraries are to be built. Primarily for glTF" OFF)
  if(ASSIMP_BUILD_DRACO)
    # Primarily for glTF v2 Enable Draco glTF feature set
    set(DRACO_GLTF
        ON
        CACHE BOOL "" FORCE)
    # Disable unnecessary or omitted components
    set(DRACO_JS_GLUE
        OFF
        CACHE BOOL "" FORCE)
    set(DRACO_WASM
        OFF
        CACHE BOOL "" FORCE)
    set(DRACO_MAYA_PLUGIN
        OFF
        CACHE BOOL "" FORCE)
    set(DRACO_UNITY_PLUGIN
        OFF
        CACHE BOOL "" FORCE)
    set(DRACO_TESTS
        OFF
        CACHE BOOL "" FORCE)

    if(ASSIMP_HUNTER_ENABLED)
      hunter_add_package(draco)
      find_package(draco CONFIG REQUIRED)
      set(draco_LIBRARIES draco::draco)
    else()
      # Draco 1.4.1 has many warnings and will not build with /WX or -Werror See
      # https://github.com/google/draco/issues/672 and
      # https://github.com/google/draco/issues/673
      if(MSVC)
        set(DRACO_CXX_FLAGS "/W0")
      else()
        list(
          APPEND
          DRACO_CXX_FLAGS
          "-Wno-bool-compare"
          "-Wno-comment"
          "-Wno-maybe-uninitialized"
          "-Wno-sign-compare"
          "-Wno-unused-local-typedefs")
        # Draco 1.4.1 does not explicitly export any symbols under GCC/clang
        list(APPEND DRACO_CXX_FLAGS "-fvisibility=default")
      endif()

      # Don't build or install all of Draco by default
      add_subdirectory("contrib/draco" EXCLUDE_FROM_ALL)

      if(MSVC OR WIN32)
        set(draco_LIBRARIES "draco")
      else()
        if(BUILD_SHARED_LIBS)
          set(draco_LIBRARIES "draco_shared")
        else()
          set(draco_LIBRARIES "draco_static")
        endif()
      endif()

      # Don't build the draco command-line tools by default
      set_target_properties(
        draco_encoder draco_decoder PROPERTIES EXCLUDE_FROM_ALL TRUE
                                               EXCLUDE_FROM_DEFAULT_BUILD TRUE)

      # Do build the draco shared library
      set_target_properties(
        ${draco_LIBRARIES} PROPERTIES EXCLUDE_FROM_ALL FALSE
                                      EXCLUDE_FROM_DEFAULT_BUILD FALSE)

      target_use_common_output_directory(${draco_LIBRARIES})
      target_use_common_output_directory(draco_encoder)
      target_use_common_output_directory(draco_decoder)

      set(draco_INCLUDE_DIRS "${CMAKE_CURRENT_SOURCE_DIR}/contrib/draco/src")

      # This is probably wrong
      install(
        TARGETS ${draco_LIBRARIES}
        EXPORT "${TARGETS_EXPORT_NAME}"
        LIBRARY DESTINATION ${ASSIMP_LIB_INSTALL_DIR}
        ARCHIVE DESTINATION ${ASSIMP_LIB_INSTALL_DIR}
        RUNTIME DESTINATION ${ASSIMP_BIN_INSTALL_DIR}
        FRAMEWORK DESTINATION ${ASSIMP_LIB_INSTALL_DIR}
                  COMPONENT ${LIBASSIMP_COMPONENT}
        INCLUDES
        DESTINATION include)

    endif()
  endif()
endif()

# Main assimp code
add_subdirectory(code/)

if(ASSIMP_BUILD_ASSIMP_TOOLS)
  # The viewer for windows only
  if(WIN32)
    find_package(DirectX)
    option(ASSIMP_BUILD_ASSIMP_VIEW
           "If the Assimp view tool is built. (requires DirectX)"
           ${DirectX_FOUND})
    if(ASSIMP_BUILD_ASSIMP_VIEW)
      add_subdirectory(tools/assimp_view/)
    endif()
  else()
    message("Building Assimp Viewer only supported on Windows.")
  endif()
  # The command line tool
  add_subdirectory(tools/assimp_cmd/)
endif()

if(ASSIMP_BUILD_SAMPLES)
  set(SAMPLES_DIR ${CMAKE_CURRENT_SOURCE_DIR}/samples)
  set(SAMPLES_SHARED_CODE_DIR ${SAMPLES_DIR}/SharedCode)
  if(WIN32)
    add_subdirectory(samples/SimpleTexturedOpenGL/)
    add_subdirectory(samples/SimpleTexturedDirectx11)
  endif()
  add_subdirectory(samples/SimpleOpenGL/)
endif()

if(ASSIMP_BUILD_TESTS)
  add_subdirectory(test/)
endif()

# Generate a pkg-config .pc, revision.h, and config.h for the Assimp library.
configure_file("${PROJECT_SOURCE_DIR}/assimp.pc.in"
               "${PROJECT_BINARY_DIR}/assimp.pc" @ONLY)
if(ASSIMP_INSTALL)
  install(
    FILES "${PROJECT_BINARY_DIR}/assimp.pc"
    DESTINATION ${ASSIMP_LIB_INSTALL_DIR}/pkgconfig/
    COMPONENT ${LIBASSIMP-DEV_COMPONENT})
endif()

configure_file(${CMAKE_CURRENT_LIST_DIR}/revision.h.in
               ${CMAKE_CURRENT_BINARY_DIR}/revision.h)

configure_file(${CMAKE_CURRENT_LIST_DIR}/include/assimp/config.h.in
               ${CMAKE_CURRENT_BINARY_DIR}/include/assimp/config.h)

if(ASSIMP_INSTALL)
  if(CMAKE_CPACK_COMMAND
     AND UNIX
     AND ASSIMP_OPT_BUILD_PACKAGES)
    # Packing information
    set(CPACK_PACKAGE_NAME
        "assimp{ASSIMP_VERSION_MAJOR}.{ASSIMP_VERSION_MINOR}")
    set(CPACK_PACKAGE_CONTACT
        ""
        CACHE STRING "Package maintainer and PGP signer.")
    set(CPACK_PACKAGE_VENDOR "https://github.com/assimp")
    set(CPACK_PACKAGE_DISPLAY_NAME "Assimp ${ASSIMP_VERSION}")
    set(CPACK_PACKAGE_DESCRIPTION_SUMMARY
        " - Open Asset Import Library ${ASSIMP_VERSION}")
    set(CPACK_PACKAGE_VERSION "${ASSIMP_VERSION}.${ASSIMP_PACKAGE_VERSION}")
    set(CPACK_PACKAGE_VERSION_MAJOR "${ASSIMP_VERSION_MAJOR}")
    set(CPACK_PACKAGE_VERSION_MINOR "${ASSIMP_VERSION_MINOR}")
    set(CPACK_PACKAGE_VERSION_PATCH "${ASSIMP_VERSION_PATCH}")
    set(CPACK_PACKAGE_INSTALL_DIRECTORY
        "assimp${ASSIMP_VERSION_MAJOR}.${ASSIMP_VERSION_MINOR}")
    set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE")

    string(TOUPPER ${LIBASSIMP_COMPONENT} "LIBASSIMP_COMPONENT_UPPER")
    string(TOUPPER ${LIBASSIMP-DEV_COMPONENT} "LIBASSIMP-DEV_COMPONENT_UPPER")

    set(CPACK_COMPONENT_ASSIMP-BIN_DISPLAY_NAME "tools")
    set(CPACK_COMPONENT_ASSIMP-BIN_DEPENDS "${LIBASSIMP_COMPONENT}")
    set(CPACK_COMPONENT_${LIBASSIMP_COMPONENT_UPPER}_DISPLAY_NAME "libraries")
    set(CPACK_COMPONENT_${LIBASSIMP-DEV_COMPONENT_UPPER}_DISPLAY_NAME
        "common headers and installs")
    set(CPACK_COMPONENT_${LIBASSIMP-DEV_COMPONENT_UPPER}_DEPENDS
        $ "{LIBASSIMP_COMPONENT}")
    set(CPACK_COMPONENT_ASSIMP-DEV_DISPLAY_NAME
        "${CPACK_COMPONENT_${LIBASSIMP-DEV_COMPONENT}_DISPLAY_NAME}")
    set(CPACK_COMPONENT_ASSIMP-DEV_DEPENDS "${LIBASSIMP-DEV_COMPONENT}")
    set(CPACK_DEBIAN_BUILD_DEPENDS debhelper cmake zlib1g-dev pkg-config)

    # debian
    set(CPACK_DEBIAN_PACKAGE_PRIORITY "optional")
    set(CPACK_DEBIAN_CMAKE_OPTIONS
        "-DBUILD_ASSIMP_SAMPLES:BOOL=${ASSIMP_BUILD_SAMPLES}")
    set(CPACK_DEBIAN_PACKAGE_SECTION "libs")
    set(CPACK_DEBIAN_PACKAGE_DEPENDS "${CPACK_COMPONENTS_ALL}")
    set(CPACK_DEBIAN_PACKAGE_SUGGESTS)
    set(cPACK_DEBIAN_PACKAGE_NAME "assimp")
    set(CPACK_DEBIAN_PACKAGE_REMOVE_SOURCE_FILES
        contrib/gtest
        contrib/zlib
        workspaces
        test
        doc
        obj
        samples
        packaging)
    set(CPACK_DEBIAN_PACKAGE_SOURCE_COPY svn export --force)
    set(CPACK_DEBIAN_CHANGELOG)
    execute_process(
      COMMAND lsb_release -is
      OUTPUT_VARIABLE _lsb_distribution
      OUTPUT_STRIP_TRAILING_WHITESPACE
      RESULT_VARIABLE _lsb_release_failed)
    set(CPACK_DEBIAN_DISTRIBUTION_NAME
        ${_lsb_distribution}
        CACHE STRING "Name of the distrubiton")
    string(TOLOWER ${CPACK_DEBIAN_DISTRIBUTION_NAME}
                   CPACK_DEBIAN_DISTRIBUTION_NAME)
    if(${CPACK_DEBIAN_DISTRIBUTION_NAME} STREQUAL "ubuntu")
      set(CPACK_DEBIAN_DISTRIBUTION_RELEASES
          lucid maverick natty oneiric precise
          CACHE STRING "Release code-names of the distrubiton release")
    endif()
    set(DPUT_HOST
        ""
        CACHE STRING "PPA repository to upload the debian sources")
    include(CPack)
    include(DebSourcePPA)
  endif()
endif()

if(WIN32)
  if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(BIN_DIR "${PROJECT_SOURCE_DIR}/bin64/")
    set(LIB_DIR "${PROJECT_SOURCE_DIR}/lib64/")
  else()
    set(BIN_DIR "${PROJECT_SOURCE_DIR}/bin32/")
    set(LIB_DIR "${PROJECT_SOURCE_DIR}/lib32/")
  endif()

  if(MSVC_TOOLSET_VERSION)
    set(MSVC_PREFIX "vc${MSVC_TOOLSET_VERSION}")
    set(ASSIMP_MSVC_VERSION ${MSVC_PREFIX})
  else()
    if(MSVC12)
      set(ASSIMP_MSVC_VERSION "vc120")
    elseif(MSVC14)
      set(ASSIMP_MSVC_VERSION "vc140")
    elseif(MSVC15)
      set(ASSIMP_MSVC_VERSION "vc141")
    endif()
  endif()

  if(MSVC12
     OR MSVC14
     OR MSVC15)
    add_custom_target(
      UpdateAssimpLibsDebugSymbolsAndDLLs
      COMMENT "Copying Assimp Libraries ..."
      VERBATIM)
    if(CMAKE_GENERATOR MATCHES "^Visual Studio")
      add_custom_command(
        TARGET UpdateAssimpLibsDebugSymbolsAndDLLs
        COMMAND
          ${CMAKE_COMMAND} -E copy
          ${CMAKE_BINARY_DIR}/code/Release/assimp-${ASSIMP_MSVC_VERSION}-mt.dll
          ${BIN_DIR}assimp-${ASSIMP_MSVC_VERSION}-mt.dll
        VERBATIM)
      add_custom_command(
        TARGET UpdateAssimpLibsDebugSymbolsAndDLLs
        COMMAND
          ${CMAKE_COMMAND} -E copy
          ${CMAKE_BINARY_DIR}/code/Release/assimp-${ASSIMP_MSVC_VERSION}-mt.exp
          ${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mt.exp
        VERBATIM)
      add_custom_command(
        TARGET UpdateAssimpLibsDebugSymbolsAndDLLs
        COMMAND
          ${CMAKE_COMMAND} -E copy
          ${CMAKE_BINARY_DIR}/code/Release/assimp-${ASSIMP_MSVC_VERSION}-mt.lib
          ${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mt.lib
        VERBATIM)
      add_custom_command(
        TARGET UpdateAssimpLibsDebugSymbolsAndDLLs
        COMMAND
          ${CMAKE_COMMAND} -E copy
          ${CMAKE_BINARY_DIR}/code/Debug/assimp-${ASSIMP_MSVC_VERSION}-mtd.dll
          ${BIN_DIR}assimp-${ASSIMP_MSVC_VERSION}-mtd.dll
        VERBATIM)
      add_custom_command(
        TARGET UpdateAssimpLibsDebugSymbolsAndDLLs
        COMMAND
          ${CMAKE_COMMAND} -E copy
          ${CMAKE_BINARY_DIR}/code/Debug/assimp-${ASSIMP_MSVC_VERSION}-mtd.exp
          ${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mtd.exp
        VERBATIM)
      add_custom_command(
        TARGET UpdateAssimpLibsDebugSymbolsAndDLLs
        COMMAND
          ${CMAKE_COMMAND} -E copy
          ${CMAKE_BINARY_DIR}/code/Debug/assimp-${ASSIMP_MSVC_VERSION}-mtd.ilk
          ${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mtd.ilk
        VERBATIM)
      add_custom_command(
        TARGET UpdateAssimpLibsDebugSymbolsAndDLLs
        COMMAND
          ${CMAKE_COMMAND} -E copy
          ${CMAKE_BINARY_DIR}/code/Debug/assimp-${ASSIMP_MSVC_VERSION}-mtd.lib
          ${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mtd.lib
        VERBATIM)
      add_custom_command(
        TARGET UpdateAssimpLibsDebugSymbolsAndDLLs
        COMMAND
          ${CMAKE_COMMAND} -E copy
          ${CMAKE_BINARY_DIR}/code/Debug/assimp-${ASSIMP_MSVC_VERSION}-mtd.pdb
          ${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mtd.pdb
        VERBATIM)
    else()
      add_custom_command(
        TARGET UpdateAssimpLibsDebugSymbolsAndDLLs
        COMMAND
          ${CMAKE_COMMAND} -E copy
          ${CMAKE_BINARY_DIR}/code/assimp-${ASSIMP_MSVC_VERSION}-mt.dll
          ${BIN_DIR}assimp-${ASSIMP_MSVC_VERSION}-mt.dll
        VERBATIM)
      add_custom_command(
        TARGET UpdateAssimpLibsDebugSymbolsAndDLLs
        COMMAND
          ${CMAKE_COMMAND} -E copy
          ${CMAKE_BINARY_DIR}/code/assimp-${ASSIMP_MSVC_VERSION}-mt.exp
          ${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mt.exp
        VERBATIM)
      add_custom_command(
        TARGET UpdateAssimpLibsDebugSymbolsAndDLLs
        COMMAND
          ${CMAKE_COMMAND} -E copy
          ${CMAKE_BINARY_DIR}/code/assimp-${ASSIMP_MSVC_VERSION}-mt.lib
          ${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mt.lib
        VERBATIM)
      add_custom_command(
        TARGET UpdateAssimpLibsDebugSymbolsAndDLLs
        COMMAND
          ${CMAKE_COMMAND} -E copy
          ${CMAKE_BINARY_DIR}/code/assimp-${ASSIMP_MSVC_VERSION}-mtd.dll
          ${BIN_DIR}assimp-${ASSIMP_MSVC_VERSION}-mtd.dll
        VERBATIM)
      add_custom_command(
        TARGET UpdateAssimpLibsDebugSymbolsAndDLLs
        COMMAND
          ${CMAKE_COMMAND} -E copy
          ${CMAKE_BINARY_DIR}/code/assimp-${ASSIMP_MSVC_VERSION}-mtd.exp
          ${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mtd.exp
        VERBATIM)
      add_custom_command(
        TARGET UpdateAssimpLibsDebugSymbolsAndDLLs
        COMMAND
          ${CMAKE_COMMAND} -E copy
          ${CMAKE_BINARY_DIR}/code/assimp-${ASSIMP_MSVC_VERSION}-mtd.ilk
          ${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mtd.ilk
        VERBATIM)
      add_custom_command(
        TARGET UpdateAssimpLibsDebugSymbolsAndDLLs
        COMMAND
          ${CMAKE_COMMAND} -E copy
          ${CMAKE_BINARY_DIR}/code/assimp-${ASSIMP_MSVC_VERSION}-mtd.lib
          ${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mtd.lib
        VERBATIM)
      add_custom_command(
        TARGET UpdateAssimpLibsDebugSymbolsAndDLLs
        COMMAND
          ${CMAKE_COMMAND} -E copy
          ${CMAKE_BINARY_DIR}/code/assimp-${ASSIMP_MSVC_VERSION}-mtd.pdb
          ${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mtd.pdb
        VERBATIM)
      add_custom_command(
        TARGET UpdateAssimpLibsDebugSymbolsAndDLLs
        COMMAND
          ${CMAKE_COMMAND} -E copy
          ${CMAKE_BINARY_DIR}/code/assimp-${ASSIMP_MSVC_VERSION}-mtd.pdb
          ${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mtd.pdb
        VERBATIM)
    endif()
  endif()
endif()
