# See docs/CMake.html for instructions about how to build LLVM with CMake.

cmake_minimum_required(VERSION 3.4.3)

if(POLICY CMP0068)
  cmake_policy(SET CMP0068 NEW)
  set(CMAKE_BUILD_WITH_INSTALL_NAME_DIR ON)
endif()

if(POLICY CMP0075)
  cmake_policy(SET CMP0075 NEW)
endif()

set(LLVM_VERSION_MAJOR 8)
set(LLVM_VERSION_MINOR 0)
set(LLVM_VERSION_PATCH 1)
set(LLVM_VERSION_SUFFIX "dismal")

# WORKAROUND: for lld/CMakeLists.txt
if (NOT PACKAGE_VERSION)
  set(PACKAGE_VERSION
    "${LLVM_VERSION_MAJOR}.${LLVM_VERSION_MINOR}.${LLVM_VERSION_PATCH}${LLVM_VERSION_SUFFIX}")
endif()

project(LLVM
  VERSION ${LLVM_VERSION_MAJOR}.${LLVM_VERSION_MINOR}.${LLVM_VERSION_PATCH}
  LANGUAGES C CXX ASM)

if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "No build type selected, default to Debug")
  set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "Build type (default Debug)" FORCE)
endif()

# Side-by-side subprojects layout: automatically set the
# LLVM_EXTERNAL_${project}_SOURCE_DIR using LLVM_ALL_PROJECTS
# This allows an easy way of setting up a build directory for llvm and another
# one for llvm+clang+... using the same sources.
set(LLVM_ALL_PROJECTS "clang;compiler-rt;lld")
set(LLVM_ENABLE_PROJECTS "" CACHE STRING
	"Semicolon-separated list of projects to build (${LLVM_ALL_PROJECTS}), or \"all\".")
if( LLVM_ENABLE_PROJECTS STREQUAL "all" )
  set( LLVM_ENABLE_PROJECTS ${LLVM_ALL_PROJECTS})
endif()
foreach(proj ${LLVM_ENABLE_PROJECTS})
  set(PROJ_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../${proj}")
  if(NOT EXISTS "${PROJ_DIR}" OR NOT IS_DIRECTORY "${PROJ_DIR}")
    message(FATAL_ERROR "LLVM_ENABLE_PROJECTS requests ${proj} but directory not found: ${PROJ_DIR}")
  endif()
  string(TOUPPER "${proj}" upper_proj)
  STRING(REGEX REPLACE "-" "_" upper_proj ${upper_proj})
  set(LLVM_EXTERNAL_${upper_proj}_SOURCE_DIR   "${CMAKE_CURRENT_SOURCE_DIR}/../${proj}")
endforeach()

# Add path for custom modules
set(CMAKE_MODULE_PATH
  ${CMAKE_MODULE_PATH}
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake"
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules"
  )

# Generate a CompilationDatabase (compile_commands.json file) for our build,
# for use by clang_complete, YouCompleteMe, etc.
set(CMAKE_EXPORT_COMPILE_COMMANDS 1)

option(LLVM_INSTALL_BINUTILS_SYMLINKS
  "Install symlinks from the binutils tool names to the corresponding LLVM tools." OFF)

option(LLVM_INSTALL_UTILS "Include utility binaries in the 'install' target." OFF)

option(LLVM_INSTALL_TOOLCHAIN_ONLY "Only include toolchain files in the 'install' target." OFF)

# Unfortunatly Clang is too eager to search directories for module maps, which can cause the
# installed version of the maps to be found when building LLVM from source. Therefore we turn off
# the installation by default. See llvm.org/PR31905.
option(LLVM_INSTALL_MODULEMAPS "Install the modulemap files in the 'install' target." OFF)

# WORKAROUND: for llvm/lib/Support/CommandLine.cpp
set(PACKAGE_NAME LLVM)

# Sanity check our source directory to make sure that we are not trying to
# generate an in-source build (unless on MSVC_IDE, where it is ok), and to make
# sure that we don't have any stray generated files lying around in the tree
# (which would end up getting picked up by header search, instead of the correct
# versions).
if( CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR AND NOT MSVC_IDE )
  message(FATAL_ERROR "In-source builds are not allowed.
Please create a directory and run cmake from there, passing the path
to this source directory as the last argument.
This process created the file `CMakeCache.txt' and the directory `CMakeFiles'.
Please delete them.")
endif()

string(TOUPPER "${CMAKE_BUILD_TYPE}" uppercase_CMAKE_BUILD_TYPE)

if (CMAKE_BUILD_TYPE AND
    NOT uppercase_CMAKE_BUILD_TYPE MATCHES "^(DEBUG|RELEASE|RELWITHDEBINFO|MINSIZEREL)$")
  message(FATAL_ERROR "Invalid value for CMAKE_BUILD_TYPE: ${CMAKE_BUILD_TYPE}")
endif()

set(LLVM_LIBDIR_SUFFIX "" CACHE STRING "Define suffix of library directory name (32/64)" )

set(LLVM_TOOLS_INSTALL_DIR "bin" CACHE STRING "Path for binary subdirectory (defaults to 'bin')")
mark_as_advanced(LLVM_TOOLS_INSTALL_DIR)

set(LLVM_UTILS_INSTALL_DIR "${LLVM_TOOLS_INSTALL_DIR}" CACHE STRING
    "Path to install LLVM utilities (enabled by LLVM_INSTALL_UTILS=ON) (defaults to LLVM_TOOLS_INSTALL_DIR)")
mark_as_advanced(LLVM_UTILS_INSTALL_DIR)

# They are used as destination of target generators.
set(LLVM_RUNTIME_OUTPUT_INTDIR ${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_CFG_INTDIR}/bin)
set(LLVM_LIBRARY_OUTPUT_INTDIR ${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_CFG_INTDIR}/lib${LLVM_LIBDIR_SUFFIX})
if(WIN32 OR CYGWIN)
  # DLL platform -- put DLLs into bin.
  set(LLVM_SHLIB_OUTPUT_INTDIR ${LLVM_RUNTIME_OUTPUT_INTDIR})
else()
  set(LLVM_SHLIB_OUTPUT_INTDIR ${LLVM_LIBRARY_OUTPUT_INTDIR})
endif()

# Each of them corresponds to llvm-config's.
set(LLVM_TOOLS_BINARY_DIR ${LLVM_RUNTIME_OUTPUT_INTDIR}) # --bindir
set(LLVM_LIBRARY_DIR      ${LLVM_LIBRARY_OUTPUT_INTDIR}) # --libdir
set(LLVM_MAIN_SRC_DIR     ${CMAKE_CURRENT_SOURCE_DIR}  ) # --src-root
set(LLVM_MAIN_INCLUDE_DIR ${LLVM_MAIN_SRC_DIR}/include ) # --includedir
set(LLVM_BINARY_DIR       ${CMAKE_CURRENT_BINARY_DIR}  ) # --prefix

# Note: LLVM_CMAKE_PATH does not include generated files
set(LLVM_CMAKE_PATH ${LLVM_MAIN_SRC_DIR}/cmake/modules)
set(LLVM_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR}/include)

# List of all targets to be built by default:
set(LLVM_ALL_TARGETS
  AArch64
  AMDGPU
  ARM
  BPF
  Hexagon
  Lanai
  Mips
  MSP430
  NVPTX
  PowerPC
  Sparc
  SystemZ
  WebAssembly
  X86
  XCore
  )

# List of targets with JIT support:
set(LLVM_TARGETS_WITH_JIT X86 PowerPC AArch64 ARM Mips SystemZ)

set(LLVM_TARGETS_TO_BUILD "all"
    CACHE STRING "Semicolon-separated list of targets to build, or \"all\".")

set(LLVM_EXPERIMENTAL_TARGETS_TO_BUILD ""
  CACHE STRING "Semicolon-separated list of experimental targets to build.")

option(BUILD_SHARED_LIBS
  "Build all libraries as shared libraries instead of static" OFF)

option(LLVM_ENABLE_BACKTRACES "Enable embedding backtraces on crash." ON)
if(LLVM_ENABLE_BACKTRACES)
  set(ENABLE_BACKTRACES 1)
endif()

option(LLVM_ENABLE_CRASH_OVERRIDES "Enable crash overrides." ON)
if(LLVM_ENABLE_CRASH_OVERRIDES)
  set(ENABLE_CRASH_OVERRIDES 1)
endif()

option(LLVM_ENABLE_FFI "Use libffi to call external functions from the interpreter" OFF)
set(FFI_LIBRARY_DIR "" CACHE PATH "Additional directory, where CMake should search for libffi.so")
set(FFI_INCLUDE_DIR "" CACHE PATH "Additional directory, where CMake should search for ffi.h or ffi/ffi.h")

set(LLVM_TARGET_ARCH "host"
  CACHE STRING "Set target to use for LLVM JIT or use \"host\" for automatic detection.")

option(LLVM_ENABLE_TERMINFO "Use terminfo database if available." ON)

set(LLVM_ENABLE_LIBXML2 "ON" CACHE STRING "Use libxml2 if available. Can be ON, OFF, or FORCE_ON")

option(LLVM_ENABLE_LIBEDIT "Use libedit if available." ON)

option(LLVM_ENABLE_LIBPFM "Use libpfm for performance counters if available." ON)

option(LLVM_ENABLE_THREADS "Use threads if available." ON)

option(LLVM_ENABLE_ZLIB "Use zlib for compression/decompression if available." ON)

if( LLVM_TARGETS_TO_BUILD STREQUAL "all" )
  set( LLVM_TARGETS_TO_BUILD ${LLVM_ALL_TARGETS} )
endif()

set(LLVM_TARGETS_TO_BUILD
   ${LLVM_TARGETS_TO_BUILD}
   ${LLVM_EXPERIMENTAL_TARGETS_TO_BUILD})
list(REMOVE_DUPLICATES LLVM_TARGETS_TO_BUILD)

option(LLVM_ENABLE_PIC "Build Position-Independent Code" ON)
option(LLVM_ENABLE_WARNINGS "Enable compiler warnings." ON)
option(LLVM_ENABLE_MODULES "Compile with C++ modules enabled." OFF)
if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
  option(LLVM_ENABLE_MODULE_DEBUGGING "Compile with -gmodules." ON)
  option(LLVM_ENABLE_LOCAL_SUBMODULE_VISIBILITY "Compile with -fmodules-local-submodule-visibility." OFF)
else()
  option(LLVM_ENABLE_MODULE_DEBUGGING "Compile with -gmodules." OFF)
  option(LLVM_ENABLE_LOCAL_SUBMODULE_VISIBILITY "Compile with -fmodules-local-submodule-visibility." ON)
endif()
option(LLVM_ENABLE_CXX1Y "Compile with C++1y enabled." OFF)
option(LLVM_ENABLE_CXX1Z "Compile with C++1z enabled." OFF)
option(LLVM_ENABLE_LIBCXX "Use libc++ if available." OFF)
option(LLVM_ENABLE_LLD "Use lld as C and C++ linker." OFF)
option(LLVM_ENABLE_PEDANTIC "Compile with pedantic enabled." ON)
option(LLVM_ENABLE_WERROR "Fail and stop if a warning is triggered." OFF)

option(LLVM_ENABLE_DUMP "Enable dump functions even when assertions are disabled" OFF)

option(LLVM_ENABLE_ASSERTIONS "Enable assertions" ON)

option(LLVM_ENABLE_EXPENSIVE_CHECKS "Enable expensive checks" OFF)

set(LLVM_ABI_BREAKING_CHECKS "WITH_ASSERTS" CACHE STRING
  "Enable abi-breaking checks.  Can be WITH_ASSERTS, FORCE_ON or FORCE_OFF.")

option(LLVM_EXTERNALIZE_DEBUGINFO
  "Generate dSYM files and strip executables and libraries (Darwin Only)" OFF)

set(LLVM_USE_SANITIZER "" CACHE STRING
  "Define the sanitizer used to build binaries and tests.")
option(LLVM_OPTIMIZE_SANITIZED_BUILDS "Pass -O1 on debug sanitizer builds" ON)
set(LLVM_LIB_FUZZING_ENGINE "" CACHE PATH
  "Path to fuzzing library for linking with fuzz targets")

option(LLVM_USE_SPLIT_DWARF
  "Use -gsplit-dwarf when compiling llvm." OFF)

# Define an option controlling whether we should build for 32-bit on 64-bit
# platforms, where supported.
if( CMAKE_SIZEOF_VOID_P EQUAL 8 AND NOT WIN32 )
  # TODO: support other platforms and toolchains.
  option(LLVM_BUILD_32_BITS "Build 32 bits executables and libraries." OFF)
endif()

# Define the default arguments to use with 'lit', and an option for the user to
# override.
set(LIT_ARGS_DEFAULT "-sv")
if (MSVC_IDE OR XCODE)
  set(LIT_ARGS_DEFAULT "${LIT_ARGS_DEFAULT} --no-progress-bar")
endif()
set(LLVM_LIT_ARGS "${LIT_ARGS_DEFAULT}" CACHE STRING "Default options for lit")

# Define options to control the inclusion and default build behavior for
# components which may not strictly be necessary (tools and tests).
#
# This is primarily to support building smaller or faster project files.
option(LLVM_INCLUDE_TOOLS "Generate build targets for the LLVM tools." ON)
option(LLVM_BUILD_TOOLS
  "Build the LLVM tools. If OFF, just generate build targets." ON)

option(LLVM_INCLUDE_UTILS "Generate build targets for the LLVM utils." ON)
option(LLVM_BUILD_UTILS
  "Build LLVM utility binaries. If OFF, just generate build targets." ON)

option(LLVM_INCLUDE_RUNTIMES "Generate build targets for the LLVM runtimes." ON)
option(LLVM_BUILD_RUNTIMES
  "Build the LLVM runtimes. If OFF, just generate build targets." ON)

option(LLVM_BUILD_RUNTIME
  "Build the LLVM runtime libraries." ON)

option(LLVM_BUILD_TESTS
  "Build LLVM unit tests. If OFF, just generate build targets." OFF)
option(LLVM_INCLUDE_TESTS "Generate build targets for the LLVM unit tests." ON)

option (LLVM_BUILD_DOCS "Build the llvm documentation." OFF)
option (LLVM_INCLUDE_DOCS "Generate build targets for llvm documentation." ON)
option (LLVM_ENABLE_DOXYGEN "Use doxygen to generate llvm API documentation." OFF)
option (LLVM_ENABLE_SPHINX "Use Sphinx to generate llvm documentation." OFF)

set(LLVM_INSTALL_DOXYGEN_HTML_DIR "share/doc/llvm/doxygen-html"
    CACHE STRING "Doxygen-generated HTML documentation install directory")
set(LLVM_INSTALL_OCAMLDOC_HTML_DIR "share/doc/llvm/ocaml-html"
    CACHE STRING "OCamldoc-generated HTML documentation install directory")

option (LLVM_BUILD_EXTERNAL_COMPILER_RT
  "Build compiler-rt as an external project." OFF)

option (LLVM_VERSION_PRINTER_SHOW_HOST_TARGET_INFO
  "Show target and host info when tools are invoked with --version." ON)

# You can configure which libraries from LLVM you want to include in the
# shared library by setting LLVM_DYLIB_COMPONENTS to a semi-colon delimited
# list of LLVM components. All component names handled by llvm-config are valid.
if(NOT DEFINED LLVM_DYLIB_COMPONENTS)
  set(LLVM_DYLIB_COMPONENTS "all" CACHE STRING
    "Semicolon-separated list of components to include in libLLVM, or \"all\".")
endif()
option(LLVM_LINK_LLVM_DYLIB "Link tools against the libllvm dynamic library" OFF)
set(LLVM_BUILD_LLVM_DYLIB_default OFF)
if(LLVM_LINK_LLVM_DYLIB OR (LLVM_BUILD_LLVM_C_DYLIB AND NOT MSVC))
  set(LLVM_BUILD_LLVM_DYLIB_default ON)
endif()
option(LLVM_BUILD_LLVM_DYLIB "Build libllvm dynamic library" ${LLVM_BUILD_LLVM_DYLIB_default})

option(LLVM_OPTIMIZED_TABLEGEN "Force TableGen to be built with optimization" OFF)
if(CMAKE_CROSSCOMPILING OR (LLVM_OPTIMIZED_TABLEGEN AND (LLVM_ENABLE_ASSERTIONS OR CMAKE_CONFIGURATION_TYPES)))
  set(LLVM_USE_HOST_TOOLS ON)
endif()

if (LLVM_BUILD_INSTRUMENTED OR LLVM_BUILD_INSTRUMENTED_COVERAGE OR
    LLVM_ENABLE_IR_PGO)
  if(NOT LLVM_PROFILE_MERGE_POOL_SIZE)
    # A pool size of 1-2 is probably sufficient on a SSD. 3-4 should be fine
    # for spining disks. Anything higher may only help on slower mediums.
    set(LLVM_PROFILE_MERGE_POOL_SIZE "4")
  endif()
  if(NOT LLVM_PROFILE_FILE_PATTERN)
    if(NOT LLVM_PROFILE_DATA_DIR)
      file(TO_NATIVE_PATH "${LLVM_BINARY_DIR}/profiles" LLVM_PROFILE_DATA_DIR)
    endif()
		file(TO_NATIVE_PATH "${LLVM_PROFILE_DATA_DIR}/%${LLVM_PROFILE_MERGE_POOL_SIZE}m.profraw" LLVM_PROFILE_FILE_PATTERN)
  endif()
endif()

if (LLVM_BUILD_STATIC)
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -static")
endif()

# Override the default target with an environment variable named by LLVM_TARGET_TRIPLE_ENV.
set(LLVM_TARGET_TRIPLE_ENV CACHE STRING "The name of environment variable to override default target. Disabled by blank.")
mark_as_advanced(LLVM_TARGET_TRIPLE_ENV)

set(LLVM_ENABLE_PER_TARGET_RUNTIME_DIR OFF CACHE BOOL
  "Enable per-target runtimes directory")

# All options referred to from HandleLLVMOptions have to be specified
# BEFORE this include, otherwise options will not be correctly set on
# first cmake run
include(config-ix)

string(REPLACE "Native" ${LLVM_NATIVE_ARCH}
  LLVM_TARGETS_TO_BUILD "${LLVM_TARGETS_TO_BUILD}")
list(REMOVE_DUPLICATES LLVM_TARGETS_TO_BUILD)

# By default, we target the host, but this can be overridden at CMake
# invocation time.
set(LLVM_DEFAULT_TARGET_TRIPLE "${LLVM_HOST_TRIPLE}" CACHE STRING
  "Default target for which LLVM will generate code." )
set(TARGET_TRIPLE "${LLVM_DEFAULT_TARGET_TRIPLE}")
message(STATUS "LLVM host triple: ${LLVM_HOST_TRIPLE}")
message(STATUS "LLVM default target triple: ${LLVM_DEFAULT_TARGET_TRIPLE}")

include(HandleLLVMOptions)

# Verify that we can find a Python 2 interpreter.  Python 3 is unsupported.
# FIXME: We should support systems with only Python 3, but that requires work
# on LLDB.
set(Python_ADDITIONAL_VERSIONS 2.7)
include(FindPythonInterp)
if( NOT PYTHONINTERP_FOUND )
  message(FATAL_ERROR
"Unable to find Python interpreter, required for builds and testing.

Please install Python or specify the PYTHON_EXECUTABLE CMake variable.")
endif()

if( ${PYTHON_VERSION_STRING} VERSION_LESS 2.7 )
  message(FATAL_ERROR "Python 2.7 or newer is required")
endif()

######
# LLVMBuild Integration
#
# We use llvm-build to generate all the data required by the CMake based
# build system in one swoop:
#
#  - We generate a file (a CMake fragment) in the object root which contains
#    all the definitions that are required by CMake.
#
#  - We generate the library table used by llvm-config.
#
#  - We generate the dependencies for the CMake fragment, so that we will
#    automatically reconfigure ourselves.

set(LLVMBUILDTOOL "${LLVM_MAIN_SRC_DIR}/utils/llvm-build/llvm-build")
set(LLVMCONFIGLIBRARYDEPENDENCIESINC
  "${LLVM_BINARY_DIR}/tools/llvm-config/LibraryDependencies.inc")
set(LLVMBUILDCMAKEFRAG
  "${LLVM_BINARY_DIR}/LLVMBuild.cmake")

message(STATUS "Constructing LLVMBuild project information")
execute_process(
  COMMAND ${PYTHON_EXECUTABLE} -B ${LLVMBUILDTOOL}
            --native-target "${LLVM_NATIVE_ARCH}"
            --enable-targets "${LLVM_TARGETS_TO_BUILD}"
            --enable-optional-components "${LLVMOPTIONALCOMPONENTS}"
            --write-library-table ${LLVMCONFIGLIBRARYDEPENDENCIESINC}
            --write-cmake-fragment ${LLVMBUILDCMAKEFRAG}
            OUTPUT_VARIABLE LLVMBUILDOUTPUT
            ERROR_VARIABLE LLVMBUILDERRORS
            OUTPUT_STRIP_TRAILING_WHITESPACE
            ERROR_STRIP_TRAILING_WHITESPACE
  RESULT_VARIABLE LLVMBUILDRESULT)

# On Win32, CMake doesn't properly handle piping the default output/error
# streams into the GUI console. So, we explicitly catch and report them.
if( NOT "${LLVMBUILDOUTPUT}" STREQUAL "")
  message(STATUS "llvm-build output: ${LLVMBUILDOUTPUT}")
endif()
if( NOT "${LLVMBUILDRESULT}" STREQUAL "0" )
  message(FATAL_ERROR
    "Unexpected failure executing llvm-build: ${LLVMBUILDERRORS}")
endif()

# Include the generated CMake fragment. This will define properties from the
# LLVMBuild files in a format which is easy to consume from CMake, and will add
# the dependencies so that CMake will reconfigure properly when the LLVMBuild
# files change.
include(${LLVMBUILDCMAKEFRAG})

######

# Configure all of the various header file fragments LLVM uses which depend on
# configuration variables.
set(LLVM_ENUM_TARGETS "")
set(LLVM_ENUM_ASM_PRINTERS "")
set(LLVM_ENUM_ASM_PARSERS "")
set(LLVM_ENUM_DISASSEMBLERS "")
foreach(t ${LLVM_TARGETS_TO_BUILD})
  set( td ${LLVM_MAIN_SRC_DIR}/lib/Target/${t} )

  list(FIND LLVM_ALL_TARGETS ${t} idx)
  list(FIND LLVM_EXPERIMENTAL_TARGETS_TO_BUILD ${t} idy)
  # At this point, LLVMBUILDTOOL already checked all the targets passed in
  # LLVM_TARGETS_TO_BUILD and LLVM_EXPERIMENTAL_TARGETS_TO_BUILD, so
  # this test just makes sure that any experimental targets were passed via
  # LLVM_EXPERIMENTAL_TARGETS_TO_BUILD, not LLVM_TARGETS_TO_BUILD.
  if( idx LESS 0 AND idy LESS 0 )
    message(FATAL_ERROR "The target `${t}' is experimental and must be passed "
      "via LLVM_EXPERIMENTAL_TARGETS_TO_BUILD.")
  else()
    set(LLVM_ENUM_TARGETS "${LLVM_ENUM_TARGETS}LLVM_TARGET(${t})\n")
  endif()

  file(GLOB asmp_file "${td}/*AsmPrinter.cpp")
  if( asmp_file )
    set(LLVM_ENUM_ASM_PRINTERS
      "${LLVM_ENUM_ASM_PRINTERS}LLVM_ASM_PRINTER(${t})\n")
  endif()
  if( EXISTS ${td}/AsmParser/CMakeLists.txt )
    set(LLVM_ENUM_ASM_PARSERS
      "${LLVM_ENUM_ASM_PARSERS}LLVM_ASM_PARSER(${t})\n")
  endif()
  if( EXISTS ${td}/Disassembler/CMakeLists.txt )
    set(LLVM_ENUM_DISASSEMBLERS
      "${LLVM_ENUM_DISASSEMBLERS}LLVM_DISASSEMBLER(${t})\n")
  endif()
endforeach(t)

# Produce the target definition files, which provide a way for clients to easily
# include various classes of targets.
configure_file(
  ${LLVM_MAIN_INCLUDE_DIR}/llvm/Config/AsmPrinters.def.in
  ${LLVM_INCLUDE_DIR}/llvm/Config/AsmPrinters.def
  )
configure_file(
  ${LLVM_MAIN_INCLUDE_DIR}/llvm/Config/AsmParsers.def.in
  ${LLVM_INCLUDE_DIR}/llvm/Config/AsmParsers.def
  )
configure_file(
  ${LLVM_MAIN_INCLUDE_DIR}/llvm/Config/Disassemblers.def.in
  ${LLVM_INCLUDE_DIR}/llvm/Config/Disassemblers.def
  )
configure_file(
  ${LLVM_MAIN_INCLUDE_DIR}/llvm/Config/Targets.def.in
  ${LLVM_INCLUDE_DIR}/llvm/Config/Targets.def
  )

# Configure the three LLVM configuration header files.
configure_file(
  ${LLVM_MAIN_INCLUDE_DIR}/llvm/Config/config.h.cmake
  ${LLVM_INCLUDE_DIR}/llvm/Config/config.h)
configure_file(
  ${LLVM_MAIN_INCLUDE_DIR}/llvm/Config/llvm-config.h.cmake
  ${LLVM_INCLUDE_DIR}/llvm/Config/llvm-config.h)
configure_file(
  ${LLVM_MAIN_INCLUDE_DIR}/llvm/Config/abi-breaking.h.cmake
  ${LLVM_INCLUDE_DIR}/llvm/Config/abi-breaking.h)


# They are not referenced. See set_output_directory().
set( CMAKE_RUNTIME_OUTPUT_DIRECTORY ${LLVM_BINARY_DIR}/bin )
set( CMAKE_LIBRARY_OUTPUT_DIRECTORY ${LLVM_BINARY_DIR}/lib${LLVM_LIBDIR_SUFFIX} )
set( CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${LLVM_BINARY_DIR}/lib${LLVM_LIBDIR_SUFFIX} )

# Work around a broken bfd ld behavior. When linking a binary with a
# foo.so library, it will try to find any library that foo.so uses and
# check its symbols. This is wasteful (the check was done when foo.so
# was created) and can fail since it is not the dynamic linker and
# doesn't know how to handle search paths correctly.
if (UNIX AND NOT APPLE AND NOT ${CMAKE_SYSTEM_NAME} MATCHES "SunOS|AIX")
  set(CMAKE_EXE_LINKER_FLAGS
      "${CMAKE_EXE_LINKER_FLAGS} -Wl,-allow-shlib-undefined")
endif()

set(CMAKE_INCLUDE_CURRENT_DIR ON)

include_directories( ${LLVM_INCLUDE_DIR} ${LLVM_MAIN_INCLUDE_DIR})

# when crosscompiling import the executable targets from a file
if(LLVM_USE_HOST_TOOLS)
  include(CrossCompile)
endif(LLVM_USE_HOST_TOOLS)
if(LLVM_TARGET_IS_CROSSCOMPILE_HOST)
# Dummy use to avoid CMake Warning: Manually-specified variables were not used
# (this is a variable that CrossCompile sets on recursive invocations)
endif()

# Make sure we don't get -rdynamic in every binary. For those that need it,
# use export_executable_symbols(target).
set(CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS "")

set(LLVM_PROFDATA_FILE "" CACHE FILEPATH
  "Profiling data file to use when compiling in order to improve runtime performance.")

include(AddLLVM)
include(TableGen)

if( MINGW AND NOT "${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang" )
  # People report that -O3 is unreliable on MinGW. The traditional
  # build also uses -O2 for that reason:
  llvm_replace_compiler_option(CMAKE_CXX_FLAGS_RELEASE "-O3" "-O2")
endif()

# Put this before tblgen. Else we have a circular dependence.
add_subdirectory(lib/Demangle)
add_subdirectory(lib/Support)
add_subdirectory(lib/TableGen)

add_subdirectory(utils/TableGen)

add_subdirectory(include/llvm)

add_subdirectory(lib)

if( LLVM_INCLUDE_UTILS )
  add_subdirectory(utils/FileCheck)
  add_subdirectory(utils/PerfectShuffle)
  add_subdirectory(utils/count)
  add_subdirectory(utils/not)
  add_subdirectory(utils/yaml-bench)
else()
  if ( LLVM_INCLUDE_TESTS )
    message(FATAL_ERROR "Including tests when not building utils will not work.
    Either set LLVM_INCLUDE_UTILS to On, or set LLVM_INCLUDE_TESTS to Off.")
  endif()
endif()

# Use LLVM_ADD_NATIVE_VISUALIZERS_TO_SOLUTION instead of LLVM_INCLUDE_UTILS because it is not really a util
if (LLVM_ADD_NATIVE_VISUALIZERS_TO_SOLUTION)
  add_subdirectory(utils/LLVMVisualizers)
endif()

add_subdirectory(projects)

if( LLVM_INCLUDE_TOOLS )
  add_subdirectory(tools)
endif()

if( LLVM_INCLUDE_RUNTIMES )
  add_subdirectory(runtimes)
endif()

if( LLVM_INCLUDE_TESTS )
  if(EXISTS ${LLVM_MAIN_SRC_DIR}/projects/test-suite AND TARGET clang)
    include(LLVMExternalProjectUtils)
    llvm_ExternalProject_Add(test-suite ${LLVM_MAIN_SRC_DIR}/projects/test-suite
      USE_TOOLCHAIN
      EXCLUDE_FROM_ALL
      NO_INSTALL
      ALWAYS_CLEAN)
  endif()
  add_subdirectory(utils/lit)
  add_subdirectory(test)
  add_subdirectory(unittests)
  if( LLVM_INCLUDE_UTILS )
    add_subdirectory(utils/unittest)
  endif()

  # Add a global check rule now that all subdirectories have been traversed
  # and we know the total set of lit testsuites.
  get_property(LLVM_LIT_TESTSUITES GLOBAL PROPERTY LLVM_LIT_TESTSUITES)
  get_property(LLVM_LIT_PARAMS GLOBAL PROPERTY LLVM_LIT_PARAMS)
  get_property(LLVM_LIT_DEPENDS GLOBAL PROPERTY LLVM_LIT_DEPENDS)
  get_property(LLVM_LIT_EXTRA_ARGS GLOBAL PROPERTY LLVM_LIT_EXTRA_ARGS)
  get_property(LLVM_ADDITIONAL_TEST_TARGETS
               GLOBAL PROPERTY LLVM_ADDITIONAL_TEST_TARGETS)
  get_property(LLVM_ADDITIONAL_TEST_DEPENDS
               GLOBAL PROPERTY LLVM_ADDITIONAL_TEST_DEPENDS)
  add_lit_target(check-all
    "Running all regression tests"
    ${LLVM_LIT_TESTSUITES}
    PARAMS ${LLVM_LIT_PARAMS}
    DEPENDS ${LLVM_LIT_DEPENDS} ${LLVM_ADDITIONAL_TEST_TARGETS}
    ARGS ${LLVM_LIT_EXTRA_ARGS}
    )
  if(TARGET check-runtimes)
    add_dependencies(check-all check-runtimes)
  endif()
  add_custom_target(test-depends
                    DEPENDS ${LLVM_LIT_DEPENDS} ${LLVM_ADDITIONAL_TEST_DEPENDS})
  set_target_properties(test-depends PROPERTIES FOLDER "Tests")
endif()

if (LLVM_INCLUDE_DOCS)
  add_subdirectory(docs)
endif()

add_subdirectory(cmake/modules)

# Do this last so that all lit targets have already been created.
if (LLVM_INCLUDE_UTILS)
  add_subdirectory(utils/llvm-lit)
endif()

if (NOT LLVM_INSTALL_TOOLCHAIN_ONLY)
  install(DIRECTORY include/llvm include/llvm-c
    DESTINATION include
    COMPONENT llvm-headers
    FILES_MATCHING
    PATTERN "*.def"
    PATTERN "*.h"
    PATTERN "*.td"
    PATTERN "*.inc"
    PATTERN "LICENSE.TXT"
    PATTERN ".svn" EXCLUDE
    )

  install(DIRECTORY ${LLVM_INCLUDE_DIR}/llvm ${LLVM_INCLUDE_DIR}/llvm-c
    DESTINATION include
    COMPONENT llvm-headers
    FILES_MATCHING
    PATTERN "*.def"
    PATTERN "*.h"
    PATTERN "*.gen"
    PATTERN "*.inc"
    # Exclude include/llvm/CMakeFiles/intrinsics_gen.dir, matched by "*.def"
    PATTERN "CMakeFiles" EXCLUDE
    PATTERN "config.h" EXCLUDE
    PATTERN ".svn" EXCLUDE
    )

  if (LLVM_INSTALL_MODULEMAPS)
    install(DIRECTORY include/llvm include/llvm-c
            DESTINATION include
            COMPONENT llvm-headers
            FILES_MATCHING
            PATTERN "module.modulemap"
            )
    install(FILES include/llvm/module.install.modulemap
            DESTINATION include/llvm
            COMPONENT llvm-headers
            RENAME "module.extern.modulemap"
            )
  endif(LLVM_INSTALL_MODULEMAPS)

  # Installing the headers needs to depend on generating any public
  # tablegen'd headers.
  add_custom_target(llvm-headers DEPENDS intrinsics_gen)
  set_target_properties(llvm-headers PROPERTIES FOLDER "Misc")

  if (NOT LLVM_ENABLE_IDE)
    add_llvm_install_targets(install-llvm-headers
                             DEPENDS llvm-headers
                             COMPONENT llvm-headers)
  endif()

  # Custom target to install all libraries.
  add_custom_target(llvm-libraries)
  set_target_properties(llvm-libraries PROPERTIES FOLDER "Misc")

  if (NOT LLVM_ENABLE_IDE)
    add_llvm_install_targets(install-llvm-libraries
                             DEPENDS llvm-libraries
                             COMPONENT llvm-libraries)
  endif()

  get_property(LLVM_LIBS GLOBAL PROPERTY LLVM_LIBS)
  if(LLVM_LIBS)
    list(REMOVE_DUPLICATES LLVM_LIBS)
    foreach(lib ${LLVM_LIBS})
      add_dependencies(llvm-libraries ${lib})
      if (NOT LLVM_ENABLE_IDE)
        add_dependencies(install-llvm-libraries install-${lib})
      endif()
    endforeach()
  endif()
endif()
