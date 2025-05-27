cmake_minimum_required(VERSION 3.16)

#=============================
# Project / Package metadata
#=============================
project(secp256k1
  # The package (a.k.a. release) version is based on semantic versioning 2.0.0 of
  # the API. All changes in experimental modules are treated as
  # backwards-compatible and therefore at most increase the minor version.
  VERSION 0.1.0
  DESCRIPTION "Optimized C library for ECDSA signatures and secret/public key operations on curve secp256k1."
  HOMEPAGE_URL "https://github.com/chaintope/secp256k1"
  LANGUAGES C
)
enable_testing()
list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)

if(CMAKE_VERSION VERSION_LESS 3.21)
  # Emulates CMake 3.21+ behavior.
  if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
    set(PROJECT_IS_TOP_LEVEL ON)
    set(${PROJECT_NAME}_IS_TOP_LEVEL ON)
  else()
    set(PROJECT_IS_TOP_LEVEL OFF)
    set(${PROJECT_NAME}_IS_TOP_LEVEL OFF)
  endif()
endif()

# The library version is based on libtool versioning of the ABI. The set of
# rules for updating the version can be found here:
# https://www.gnu.org/software/libtool/manual/html_node/Updating-version-info.html
# All changes in experimental modules are treated as if they don't affect the
# interface and therefore only increase the revision.
set(${PROJECT_NAME}_LIB_VERSION_CURRENT 0)
set(${PROJECT_NAME}_LIB_VERSION_REVISION 1)
set(${PROJECT_NAME}_LIB_VERSION_AGE 0)

#=============================
# Language setup
#=============================
set(CMAKE_C_STANDARD 90)
set(CMAKE_C_EXTENSIONS OFF)

#=============================
# Configurable options
#=============================
option(BUILD_SHARED_LIBS "Build shared libraries." ON)
option(SECP256K1_DISABLE_SHARED "Disable shared library. Overrides BUILD_SHARED_LIBS." OFF)
if(SECP256K1_DISABLE_SHARED)
  set(BUILD_SHARED_LIBS OFF)
endif()

option(SECP256K1_INSTALL "Enable installation." ${PROJECT_IS_TOP_LEVEL})

## Modules

# We declare all options before processing them, to make sure we can express
# dependencies while processing.
option(SECP256K1_ENABLE_MODULE_ECDH "Enable ECDH module." ON)
option(SECP256K1_ENABLE_MODULE_RECOVERY "Enable ECDSA pubkey recovery module." OFF)
option(SECP256K1_ENABLE_MODULE_SCHNORR "Enable schnorrsig module." ON)

# Processing must be done in a topological sorting of the dependency graph
# (dependent module first).

if(SECP256K1_ASM STREQUAL "arm32")
  enable_language(ASM)
  include(CheckArm32Assembly)
  check_arm32_assembly()
  if(HAVE_ARM32_ASM)
    add_compile_definitions(USE_EXTERNAL_ASM=1)
  else()
    message(FATAL_ERROR "ARM32 assembly requested but not available.")
  endif()
elseif(SECP256K1_ASM)
  include(CheckX86_64Assembly)
  check_x86_64_assembly()
  if(HAVE_X86_64_ASM)
    set(SECP256K1_ASM "x86_64")
    add_compile_definitions(USE_ASM_X86_64=1)
  elseif(SECP256K1_ASM STREQUAL "AUTO")
    set(SECP256K1_ASM "OFF")
  else()
    message(FATAL_ERROR "x86_64 assembly requested but not available.")
  endif()
endif()

option(SECP256K1_EXPERIMENTAL "Allow experimental configuration options." OFF)
if(NOT SECP256K1_EXPERIMENTAL)
  if(SECP256K1_ASM STREQUAL "arm32")
    message(FATAL_ERROR "ARM32 assembly is experimental. Use -DSECP256K1_EXPERIMENTAL=ON to allow.")
  endif()
endif()

option(SECP256K1_BUILD_BENCHMARK "Build benchmarks." ON)
option(SECP256K1_BUILD_TESTS "Build tests." ON)

# Redefine configuration flags.
# We leave assertions on.
string(REGEX REPLACE "-DNDEBUG[ \t\r\n]*" "" CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELWITHDEBINFO}")
string(REGEX REPLACE "-DNDEBUG[ \t\r\n]*" "" CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE}")
string(REGEX REPLACE "-DNDEBUG[ \t\r\n]*" "" CMAKE_C_FLAGS_MINSIZEREL "${CMAKE_C_FLAGS_MINSIZEREL}")
# Prefer -O2 optimization level. (-O3 is CMake's default for Release for many compilers.)
string(REGEX REPLACE "-O3( |$)" "-O2\\1" CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE}")

# Define custom "Coverage" build type.
set(CMAKE_C_FLAGS_COVERAGE "${CMAKE_C_FLAGS_RELWITHDEBINFO} -O0 -DCOVERAGE=1 --coverage" CACHE STRING
  "Flags used by the C compiler during \"Coverage\" builds."
  FORCE
)
set(CMAKE_EXE_LINKER_FLAGS_COVERAGE "${CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO} --coverage" CACHE STRING
  "Flags used for linking binaries during \"Coverage\" builds."
  FORCE
)
set(CMAKE_SHARED_LINKER_FLAGS_COVERAGE "${CMAKE_SHARED_LINKER_FLAGS_RELWITHDEBINFO} --coverage" CACHE STRING
  "Flags used by the shared libraries linker during \"Coverage\" builds."
  FORCE
)
mark_as_advanced(
  CMAKE_C_FLAGS_COVERAGE
  CMAKE_EXE_LINKER_FLAGS_COVERAGE
  CMAKE_SHARED_LINKER_FLAGS_COVERAGE
)

if(PROJECT_IS_TOP_LEVEL)
  get_property(is_multi_config GLOBAL PROPERTY GENERATOR_IS_MULTI_CONFIG)
  set(default_build_type "RelWithDebInfo")
  if(is_multi_config)
    set(CMAKE_CONFIGURATION_TYPES "${default_build_type}" "Release" "Debug" "MinSizeRel" "Coverage" CACHE STRING
      "Supported configuration types."
      FORCE
    )
  else()
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY
      STRINGS "${default_build_type}" "Release" "Debug" "MinSizeRel" "Coverage"
    )
    if(NOT CMAKE_BUILD_TYPE)
      message(STATUS "Setting build type to \"${default_build_type}\" as none was specified")
      set(CMAKE_BUILD_TYPE "${default_build_type}" CACHE STRING
        "Choose the type of build."
        FORCE
      )
    endif()
  endif()
endif()

include(TryAppendCFlags)
try_append_c_flags(-Wall) # GCC >= 2.95 and probably many other compilers.


# Keep the following commands ordered lexicographically.
try_append_c_flags(-pedantic)
try_append_c_flags(-Wcast-align) # GCC >= 2.95.
try_append_c_flags(-Wcast-align=strict) # GCC >= 8.0.
#try_append_c_flags(-Wconditional-uninitialized) # Clang >= 3.0 only.
try_append_c_flags(-Wextra) # GCC >= 3.4, this is the newer name of -W, which we don't use because older GCCs will warn about unused functions.
try_append_c_flags(-Wnested-externs)
try_append_c_flags(-Wno-long-long) # GCC >= 3.0, -Wlong-long is implied by -pedantic.
try_append_c_flags(-Wno-overlength-strings) # GCC >= 4.2, -Woverlength-strings is implied by -pedantic.
try_append_c_flags(-Wno-unused-function) # GCC >= 3.0, -Wunused-function is implied by -Wall.
try_append_c_flags(-Wreserved-identifier) # Clang >= 13.0 only.
try_append_c_flags(-Wshadow)
try_append_c_flags(-Wstrict-prototypes)
try_append_c_flags(-Wundef)


set(SECP256K1_APPEND_CFLAGS "" CACHE STRING "Compiler flags that are appended to the command line after all other flags added by the build system. This variable is intended for debugging and special builds.")
if(SECP256K1_APPEND_CFLAGS)
  # Appending to this low-level rule variable is the only way to
  # guarantee that the flags appear at the end of the command line.
  string(APPEND CMAKE_C_COMPILE_OBJECT " ${SECP256K1_APPEND_CFLAGS}")
endif()

set(SECP256K1_APPEND_LDFLAGS "" CACHE STRING "Linker flags that are appended to the command line after all other flags added by the build system. This variable is intended for debugging and special builds.")
if(SECP256K1_APPEND_LDFLAGS)
  # Appending to this low-level rule variable is the only way to
  # guarantee that the flags appear at the end of the command line.
  string(APPEND CMAKE_C_CREATE_SHARED_LIBRARY " ${SECP256K1_APPEND_LDFLAGS}")
  string(APPEND CMAKE_C_LINK_EXECUTABLE " ${SECP256K1_APPEND_LDFLAGS}")
endif()

if(NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY)
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)
endif()
if(NOT CMAKE_LIBRARY_OUTPUT_DIRECTORY)
  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
endif()
if(NOT CMAKE_ARCHIVE_OUTPUT_DIRECTORY)
  set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
endif()

# Generate config file
set(USE_NUM_GMP 0)
set(USE_NUM_NONE 1)
set(USE_FIELD_INV_BUILTIN 1)
set(USE_SCALAR_INV_BUILTIN 1)

configure_file(
  ${PROJECT_SOURCE_DIR}/src/libsecp256k1-config.h.cmake.in
  ${PROJECT_BINARY_DIR}/libsecp256k1-config.h
  @ONLY
)

# Add project include directories
include_directories(
  ${PROJECT_SOURCE_DIR}
  ${PROJECT_SOURCE_DIR}/include
  ${PROJECT_BINARY_DIR}  # For generated config file
)

add_subdirectory(src)

# Static precomputation for eliptic curve mutliplication
option(SECP256K1_ECMULT_STATIC_PRECOMPUTATION "Precompute libsecp256k1's eliptic curve mutliplication tables" ON)
if(SECP256K1_ECMULT_STATIC_PRECOMPUTATION)
  set(USE_ECMULT_STATIC_PRECOMPUTATION 1)

  include(NativeExecutable)
  add_native_executable(gen_context src/gen_context.c)
  target_include_directories(gen_context 
    PRIVATE
    ${PROJECT_SOURCE_DIR}
    include
  )
  target_compile_options(gen_context PRIVATE -Wall -Wextra -Wno-unused-function)

  add_custom_command(
    OUTPUT ${PROJECT_SOURCE_DIR}/src/ecmult_static_context.h
    COMMAND gen_context
    DEPENDS gen_context
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
    COMMENT "Generating ecmult_static_context.h"
  )
  add_custom_target(gen_context_h DEPENDS ${PROJECT_SOURCE_DIR}/src/ecmult_static_context.h)
  add_dependencies(secp256k1 gen_context_h)
endif()

message("\n")
message("secp256k1 configure summary")
message("===========================")
message("Build artifacts:")
if(BUILD_SHARED_LIBS)
  set(library_type "Shared")
else()
  set(library_type "Static")
endif()

message("  library type ........................ ${library_type}")
message("Optional modules:")
message("  ECDH ................................ ${SECP256K1_ENABLE_MODULE_ECDH}")
message("  ECDSA pubkey recovery ............... ${SECP256K1_ENABLE_MODULE_RECOVERY}")
message("  schnorrsig .......................... ${SECP256K1_ENABLE_MODULE_SCHNORR}")
message("Optional binaries:")
message("  benchmark ........................... ${SECP256K1_BUILD_BENCHMARK}")
message("  noverify_tests ...................... ${SECP256K1_BUILD_TESTS}")
set(tests_status "${SECP256K1_BUILD_TESTS}")
if(CMAKE_BUILD_TYPE STREQUAL "Coverage")
  set(tests_status OFF)
endif()
message("  tests ............................... ${tests_status}")
message("")
if(CMAKE_CROSSCOMPILING)
  set(cross_status "TRUE, for ${CMAKE_SYSTEM_NAME}, ${CMAKE_SYSTEM_PROCESSOR}")
else()
  set(cross_status "FALSE")
endif()
message("Cross compiling ....................... ${cross_status}")
message("Valgrind .............................. ${SECP256K1_VALGRIND}")
get_directory_property(definitions COMPILE_DEFINITIONS)
string(REPLACE ";" " " definitions "${definitions}")
message("Preprocessor defined macros ........... ${definitions}")
message("C compiler ............................ ${CMAKE_C_COMPILER_ID} ${CMAKE_C_COMPILER_VERSION}, ${CMAKE_C_COMPILER}")
message("CFLAGS ................................ ${CMAKE_C_FLAGS}")
get_directory_property(compile_options COMPILE_OPTIONS)
string(REPLACE ";" " " compile_options "${compile_options}")
message("Compile options ....................... " ${compile_options})
if(NOT is_multi_config)
  message("Build type:")
  message(" - CMAKE_BUILD_TYPE ................... ${CMAKE_BUILD_TYPE}")
  string(TOUPPER "${CMAKE_BUILD_TYPE}" build_type)
  message(" - CFLAGS ............................. ${CMAKE_C_FLAGS_${build_type}}")
  message(" - LDFLAGS for executables ............ ${CMAKE_EXE_LINKER_FLAGS_${build_type}}")
  message(" - LDFLAGS for shared libraries ....... ${CMAKE_SHARED_LINKER_FLAGS_${build_type}}")
else()
  message("Supported configurations .............. ${CMAKE_CONFIGURATION_TYPES}")
  message("RelWithDebInfo configuration:")
  message(" - CFLAGS ............................. ${CMAKE_C_FLAGS_RELWITHDEBINFO}")
  message(" - LDFLAGS for executables ............ ${CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO}")
  message(" - LDFLAGS for shared libraries ....... ${CMAKE_SHARED_LINKER_FLAGS_RELWITHDEBINFO}")
  message("Debug configuration:")
  message(" - CFLAGS ............................. ${CMAKE_C_FLAGS_DEBUG}")
  message(" - LDFLAGS for executables ............ ${CMAKE_EXE_LINKER_FLAGS_DEBUG}")
  message(" - LDFLAGS for shared libraries ....... ${CMAKE_SHARED_LINKER_FLAGS_DEBUG}")
endif()
if(SECP256K1_APPEND_CFLAGS)
  message("SECP256K1_APPEND_CFLAGS ............... ${SECP256K1_APPEND_CFLAGS}")
endif()
if(SECP256K1_APPEND_LDFLAGS)
  message("SECP256K1_APPEND_LDFLAGS .............. ${SECP256K1_APPEND_LDFLAGS}")
endif()
message("")
if(SECP256K1_EXPERIMENTAL)
  message(
    "  ******\n"
    "  WARNING: experimental build\n"
    "  Experimental features do not have stable APIs or properties, and may not be safe for production use.\n"
    "  ******\n"
  )
endif()