cmake_minimum_required(VERSION 3.22.1)
message(STATUS "CMake version: ${CMAKE_VERSION}")

set(CMAKE_BUILD_TYPE Debug CACHE STRING "")
project(RealmCore)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/tools/cmake")

if(NOT DEFINED REALM_CORE_SUBMODULE_BUILD AND NOT CMAKE_SOURCE_DIR STREQUAL ${CMAKE_CURRENT_SOURCE_DIR})
    # Realm Core is being built as part of another project, likely an SDK
    set(REALM_CORE_SUBMODULE_BUILD ON)
endif()

# Include general CMake modules
include(GNUInstallDirs)
include(CheckIncludeFiles)
include(CheckSymbolExists)
include(CMakePushCheckState)
include(Utilities)
include(SpecialtyBuilds)
include(GetVersion)
include(CheckCXXCompilerFlag)
include(CheckCXXSourceCompiles)
include(CheckCXXSourceRuns)
include(AcquireRealmDependency)

include(CodeCoverage)

if(NOT DEFINED REALM_VERSION)
    # Get accurate git-describe version
    include(GetGitRevisionDescription)
    git_describe(REALM_VERSION)
endif()
set(PROJECT_VERSION ${DEP_VERSION})

# Project-wide build flags
if (NOT DEFINED CMAKE_CXX_STANDARD)
    set(CMAKE_CXX_STANDARD 17)
endif()
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

if(NOT APPLE)
    # TODO: Core APIs should always built for internal usage. But there seems to be issues with cocoa. Enable it only for Android for now.
    set(CMAKE_CXX_VISIBILITY_PRESET hidden)
elseif(APPLE)
    set(CMAKE_VISIBILITY_INLINES_HIDDEN 1)
endif()

function(check_if_cxx_compiler_flag_supported flag out_var)
    if(flag MATCHES "^-Wno-")
        # Compilers ignore unknown -Wno-foo flags, so look for -Wfoo instead.
        string(REPLACE "-Wno-" "-W" check_flag ${flag})
    else()
        set(check_flag ${flag})
    endif()

    check_cxx_compiler_flag(${check_flag} ${out_var})
endfunction()

function(add_cxx_flag_if_supported flag)
    check_if_cxx_compiler_flag_supported(${flag} HAVE_${flag})
    if(HAVE_${flag})
        add_compile_options($<$<COMPILE_LANGUAGE:CXX>:${flag}>)
    endif()
endfunction()

# Usage: append_source_file_compile_options(FILES <file1>[ <file2>...] FLAGS <flag1>[ <flag2>...])
function(append_source_file_compile_options)
    set(options)
    set(args)
    set(list_args FILES FLAGS)
    cmake_parse_arguments(append_options "${options}" "${args}" "${list_args}" ${ARGN})
    if (NOT append_options_FILES)
        message(FATAL_ERROR "Missing FILES argument for append_source_file_compile_options")
    endif()
    if (NOT append_options_FLAGS)
        message(FATAL_ERROR "Missing FLAGS argument for append_source_file_compile_options")
    endif()
    foreach(FLAG_ IN LISTS append_options_FLAGS)
        check_if_cxx_compiler_flag_supported(${FLAG_} HAVE${FLAG_})
        if (HAVE${FLAG_})
            foreach(SRC_ IN LISTS append_options_FILES)
                get_source_file_property(OLD_OPTS ${SRC_} COMPILE_OPTIONS)
                if (NOT OLD_OPTS)
                    set_source_files_properties(${SRC_} PROPERTIES COMPILE_OPTIONS "${FLAG_}")
                elseif(NOT "${FLAG_}" IN_LIST OLD_OPTS)
                    set_source_files_properties(${SRC_} PROPERTIES COMPILE_OPTIONS "${OLD_OPTS};${FLAG_}")
                endif()
            endforeach()
        endif()
    endforeach()
endfunction()

function(add_target_option_if_supported target option_scope)
    foreach(option ${ARGN})
        check_if_cxx_compiler_flag_supported(${option} HAVE_${option})
        if(HAVE_${option})
            target_compile_options(${target} ${option_scope} ${option})
        endif()
    endforeach()
endfunction()

function(use_faster_linker)
    # If a linker has already been set, don't override.
    if ("${CMAKE_EXE_LINKER_FLAGS}" MATCHES "-fuse-ld=")
        return()
    endif()

    foreach(LINKER lld gold) # lld > gold > default
        # there is no link equivalent to check_cxx_compiler_flag()
        set(CMAKE_REQUIRED_LINK_OPTIONS "-fuse-ld=${LINKER}")
        check_cxx_source_compiles("int main() {}" HAVE_LINKER_${LINKER})
        if(HAVE_LINKER_${LINKER})
            message(STATUS "Using linker ${LINKER}")
            set(REALM_USE_FAST_LINKER "-fuse-ld=${LINKER}" PARENT_SCOPE)
            return()
        endif()
    endforeach()
endfunction()

# Set global warnings settings
if(MSVC)
    add_compile_options(/W3 /D_CRT_SECURE_NO_WARNINGS /D_SCL_SECURE_NO_WARNINGS)

    # Don't complain about unary minus on unsigned resulting in positive number.
    add_compile_options(/wd4146)

    # We use these in our AtomicSharedPtr implementation and it can't move
    # to atomic<shared_ptr<T>> because NotificationToken relies on movability.
    add_compile_options(/D_SILENCE_CXX20_OLD_SHARED_PTR_ATOMIC_SUPPORT_DEPRECATION_WARNING)

    # Enable __cplusplus macro
    add_compile_options(/Zc:__cplusplus)
else()
    add_compile_options(-Wall -Wextra -Wempty-body -Wparentheses -Wunknown-pragmas -Wunreachable-code -Wunused-parameter -Wno-missing-field-initializers)
    # TODO: Remove this when fixed
    if(ANDROID)
        add_compile_options(-Wno-uninitialized)
    elseif(${CMAKE_CXX_COMPILER_ID} MATCHES ".*[Cc]lang")
        # FIXME: Re-introduce -Wold-style-cast
        add_compile_options(-Wunreachable-code -Wshorten-64-to-32 -Wconditional-uninitialized -Wextra-semi -Wno-nested-anon-types -Wdocumentation -Wthread-safety -Wthread-safety-negative -Wmissing-prototypes)
    endif()

    # By default GCC warns that it changed how it passes bitfields by value as
    # function arguments on arm64 every time this happens. This warning is just
    # an FYI and not pointing out a problem, so just disable it.
    add_cxx_flag_if_supported(-Wno-psabi)

    add_cxx_flag_if_supported(-Wpartial-availability)

    # This warning is too agressive. It warns about moves that are not redundant on older
    # compilers that we still support. It is also harmless, unlike pessimizing moves.
    add_cxx_flag_if_supported(-Wno-redundant-move)

    # For backwards compatibility libc++ public headers have some transitive
    # includes which they don't use. Disabling this improves build times a bit.
    add_compile_definitions(_LIBCPP_REMOVE_TRANSITIVE_INCLUDES)

    # Ninja buffers output so we need to tell the compiler to use colors even though stdout isn't a tty.
    if("${CMAKE_GENERATOR}" STREQUAL "Ninja")
        add_cxx_flag_if_supported(-fdiagnostics-color)
    endif()

    use_faster_linker()
endif()

if(ANDROID)
    # Optimize for size vs. performance for Android. The has the following implications:
    # - Add `-ffunction-sections` and `-fdata-sections`. This requires that `-Wl,-gc-sections` are used when creating
    #   the final .so file.
    # - `-fstrict-aliasing` is inherited from NDK r10e.
    # - `-fomit-frame-pointer` is inherited from NDK r10e.
    # - On some architectures char is unsigned by default. Make it signed
    # - Compile with -Oz in Release because on Android we favor code size over performance
    #
    add_compile_options(-fdata-sections -ffunction-sections -fomit-frame-pointer -fsigned-char -fstrict-aliasing -funwind-tables -no-canonical-prefixes $<$<CONFIG:Release>:-Oz>)
endif()

if(EMSCRIPTEN AND NOT REALM_CORE_SUBMODULE_BUILD)
    add_compile_options(-fwasm-exceptions)
    add_link_options(-fwasm-exceptions -sALLOW_MEMORY_GROWTH=1)
    set(CMAKE_EXECUTABLE_SUFFIX ".html")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --emrun")
endif()

set(CMAKE_DEBUG_POSTFIX "-dbg")
set(CMAKE_MINSIZEDEBUG_POSTFIX "-dbg")

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if(CMAKE_SYSTEM_NAME MATCHES "^Windows")
    add_compile_definitions(
        WIN32_LEAN_AND_MEAN # include minimal Windows.h for faster builds
        UNICODE # prefer Unicode variants of Windows APIs over ANSI variants
        _UNICODE # prefer Unicode variants of C runtime APIs over ANSI variants
    )
    if(NOT WINDOWS_STORE)
        # for regular Windows target Windows 8.1 as the minimum version
        # see https://docs.microsoft.com/en-us/windows/win32/WinProg/using-the-windows-headers
        add_compile_definitions(
            _WIN32_WINNT=0x0603
            WINVER=0x603
            NTDDI_VERSION=0x06030000
        )
    endif()
endif()

if(MSVC)
    add_compile_options(
        /MP # Enable multi-processor compilation
    )
    if(WINDOWS_STORE)
        # Removing LNK4075 warnings for debug UWP builds
        # "LNK4075: ignoring '/INCREMENTAL' due to '/OPT:ICF' specification"
        # "LNK4075: ignoring '/INCREMENTAL' due to '/OPT:REF' specification"

        # Optional verification checks since we don't know existing contents of variables below
        string(REPLACE "/OPT:ICF " "/OPT:NOICF " CMAKE_EXE_LINKER_FLAGS_DEBUG "${CMAKE_EXE_LINKER_FLAGS_DEBUG}")
        string(REPLACE "/OPT:REF " "/OPT:NOREF " CMAKE_EXE_LINKER_FLAGS_DEBUG "${CMAKE_EXE_LINKER_FLAGS_DEBUG}")
        string(REPLACE "/INCREMENTAL:YES " "/INCREMENTAL:NO " CMAKE_EXE_LINKER_FLAGS_DEBUG "${CMAKE_EXE_LINKER_FLAGS_DEBUG}")
        string(REPLACE "/INCREMENTAL " "/INCREMENTAL:NO " CMAKE_EXE_LINKER_FLAGS_DEBUG "${CMAKE_EXE_LINKER_FLAGS_DEBUG}")

        # Mandatory
        set(CMAKE_MODULE_LINKER_FLAGS_DEBUG  "${CMAKE_MODULE_LINKER_FLAGS_DEBUG} /INCREMENTAL:NO /OPT:NOREF /OPT:NOICF")
        set(CMAKE_EXE_LINKER_FLAGS_DEBUG     "${CMAKE_EXE_LINKER_FLAGS_DEBUG} /INCREMENTAL:NO /OPT:NOREF /OPT:NOICF")
        set(CMAKE_SHARED_LINKER_FLAGS_DEBUG  "${CMAKE_SHARED_LINKER_FLAGS_DEBUG} /INCREMENTAL:NO /OPT:NOREF /OPT:NOICF")
    else()
        if(NOT DEFINED CMAKE_MSVC_RUNTIME_LIBRARY)
            set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
        endif()
        string(REGEX REPLACE "/RTC(su|[1su])" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
    endif()

    # Let the compiler optimize debug builds ever so slightly.
    # /Ob1: This allows the compiler to inline functions marked __inline.
    # /Oi:  This enables (faster) intrinsic functions.
    string(REPLACE "/Ob0" "/Ob1" CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
    add_compile_options($<$<CONFIG:Debug>:/Oi>)

    # Prevent warnings abount non-existing .pdb files
    add_link_options($<$<CONFIG:Debug>:/ignore:4099>)
endif()

if(UNIX)
    cmake_push_check_state(RESET)
    # Enable access to large file APIs, but don't make them the default.
    add_compile_definitions("_LARGEFILE_SOURCE" "_LARGEFILE64_SOURCE")
    set(CMAKE_REQUIRED_DEFINITIONS "-D_LARGEFILE_SOURCE" "-D_LARGEFILE64_SOURCE")
    # Use readdir64 if available.
    check_symbol_exists(readdir64 "dirent.h" REALM_HAVE_READDIR64)
    cmake_reset_check_state()
    set(CMAKE_REQUIRED_DEFINITIONS "-D_POSIX_C_SOURCE=200112L")
    check_symbol_exists(posix_fallocate "fcntl.h" REALM_HAVE_POSIX_FALLOCATE)
    if(REALM_HAVE_POSIX_FALLOCATE)
        add_compile_definitions("_POSIX_C_SOURCE=200112L")
    endif()
    cmake_pop_check_state()
endif()

# Options (passed to CMake)
option(REALM_ENABLE_SYNC "Enable synchronized realms." ON)
option(REALM_BUILD_TEST_CLIENT "Build the test client" OFF)
option(REALM_ENABLE_ASSERTIONS "Enable assertions in release mode." OFF)
option(REALM_ENABLE_ALLOC_SET_ZERO "Zero all allocations." OFF)
if(NOT EMSCRIPTEN)
    option(REALM_ENABLE_ENCRYPTION "Enable encryption." ON)
endif()
option(REALM_ENABLE_MEMDEBUG "Add additional memory checks" OFF)
option(REALM_VALGRIND "Tell the test suite we are running with valgrind" OFF)
option(REALM_SYNC_MULTIPLEXING "Enables/disables sync session multiplexing by default" ON)
set(REALM_MAX_BPNODE_SIZE "1000" CACHE STRING "Max B+ tree node size.")
option(REALM_ENABLE_GEOSPATIAL "Enable geospatial types and queries." ON)
option(REALM_APP_SERVICES "Enable the default app services implementation." ON)

# Find dependencies
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)

# Threads library is pthread-compatible
# Check for these explicitly because older versions of musl define only one or the other
if(CMAKE_USE_PTHREADS_INIT)
    check_symbol_exists(pthread_getname_np "pthread.h" REALM_HAVE_PTHREAD_GETNAME)
    check_symbol_exists(pthread_setname_np "pthread.h" REALM_HAVE_PTHREAD_SETNAME)
endif()

find_package(Backtrace)
set(REALM_HAVE_BACKTRACE ${Backtrace_FOUND})

if(REALM_ENABLE_SYNC)
    option(REALM_FORCE_OPENSSL "Always use OpenSSL for SSL needs, regardless of target platform." OFF)
    if(CMAKE_SYSTEM_NAME MATCHES "^Windows|Linux|Android|OHOS")
        set(REALM_NEEDS_OPENSSL TRUE)
    endif()
    if(REALM_NEEDS_OPENSSL OR REALM_FORCE_OPENSSL)
        option(REALM_INCLUDE_CERTS "Include a list of trust certificates in the build for OpenSSL certificate verification" ON)
    endif()
    elseif(REALM_ENABLE_ENCRYPTION AND CMAKE_SYSTEM_NAME MATCHES "Linux|Android|OHOS")
    set(REALM_NEEDS_OPENSSL TRUE)
endif()

if(REALM_NEEDS_OPENSSL OR REALM_FORCE_OPENSSL)
    option(REALM_USE_SYSTEM_OPENSSL "Look for an external OpenSSL installation instead of using prebuilt one." OFF)
    set(_REALM_USE_OPENSSL_DEFAULT_VERIFY_PATHS ON)
    if(NOT REALM_USE_SYSTEM_OPENSSL AND (ANDROID OR WIN32 OR CMAKE_SYSTEM_NAME STREQUAL "Linux"))
        # Use our own prebuilt OpenSSL
        realm_acquire_dependency(openssl ${DEP_OPENSSL_VERSION} OPENSSL_CMAKE_INCLUDE_FILE)

        include(${OPENSSL_CMAKE_INCLUDE_FILE})
        set(_REALM_USE_OPENSSL_DEFAULT_VERIFY_PATHS OFF)
    endif()

    find_package(OpenSSL REQUIRED)
    set(REALM_HAVE_OPENSSL ON)
    option(REALM_USE_SYSTEM_OPENSSL_PATHS "Use the system OpenSSL certificate store (specified by the OPENSSLDIR environment variable) at runtime for TLS handshake." ${_REALM_USE_OPENSSL_DEFAULT_VERIFY_PATHS})
    string(REGEX MATCH "^([0-9]+)\\.([0-9]+)" OPENSSL_VERSION_MAJOR_MINOR "${OPENSSL_VERSION}")
elseif(APPLE)
    set(REALM_HAVE_SECURE_TRANSPORT "1")
endif()

# Use Zlib for Sync, but allow integrators to override it
# Don't use find_library(ZLIB) on Apple platforms - it hardcodes the path per platform,
# so for an iOS build it'll use the path from the Device plaform, which is an error on Simulator.
# Just use -lz and let Xcode figure it out
# Emscripten does provide Zlib, but it doesn't work with find_package and is handled specially
if(NOT APPLE AND NOT EMSCRIPTEN AND NOT TARGET ZLIB::ZLIB)
    if(WIN32 OR (CMAKE_SYSTEM_NAME STREQUAL "Linux" AND REALM_LINUX_TOOLCHAIN))
        find_package(ZLIB)
        if (NOT ZLIB_FOUND)
            realm_acquire_dependency(zlib ${DEP_ZLIB_VERSION} ZLIB_CMAKE_INCLUDE_FILE)
            include(${ZLIB_CMAKE_INCLUDE_FILE})
        endif()
    elseif(ANDROID)
        # On Android FindZLIB chooses the static libz over the dynamic one, but this leads to issues
        # (see https://github.com/android/ndk/issues/1179)
        # We want to link against the stub library instead of statically linking anyway,
        # so we hack find_library to only consider shared object libraries when looking for libz
        set(_CMAKE_FIND_LIBRARY_SUFFIXES_orig ${CMAKE_FIND_LIBRARY_SUFFIXES})
        set(CMAKE_FIND_LIBRARY_SUFFIXES .so)
    endif()
    find_package(ZLIB REQUIRED)
    if(ANDROID)
        set(CMAKE_FIND_LIBRARY_SUFFIXES ${_CMAKE_FIND_LIBRARY_SUFFIXES_orig})
    endif()
endif()

# Store configuration in header file
configure_file(src/realm/util/config.h.in src/realm/util/config.h)

# Configure source code to use right version number
configure_file(src/realm/version_numbers.hpp.in src/realm/version_numbers.hpp)

set(DEPRECATED_CONFIG_FILE "${RealmCore_SOURCE_DIR}/src/realm/util/config.h")
if(EXISTS "${DEPRECATED_CONFIG_FILE}")
    message(FATAL_ERROR "${DEPRECATED_CONFIG_FILE} exists in the source directory, and will take precedence over the generated configuration in the build directory. Please remove this file before continuing. Alternatively, you can also clean your realm-core to remove this and other stale files: git clean -xfd")
endif()

set(JSON_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/json)

# Tell the build system where to find the sources (and generated sources)
include_directories(src)
include_directories(${CMAKE_CURRENT_BINARY_DIR}/src) # For generated files (like config.h)

add_subdirectory(src)
add_subdirectory(bindgen)

# Install the licence and changelog files
install(FILES LICENSE CHANGELOG.md DESTINATION "doc/realm" COMPONENT devel)

# Only prepare test/install/package targets if we're not a submodule
if(REALM_CORE_SUBMODULE_BUILD)
    return()
endif()

# Make the project importable from the build directory
set(REALM_EXPORTED_TARGETS
        Storage
        QueryParser
        ObjectStore
        RealmFFI
        RealmFFIStatic
        )
if(REALM_ENABLE_SYNC)
    list(APPEND REALM_EXPORTED_TARGETS Sync)
endif()
export(TARGETS ${REALM_EXPORTED_TARGETS} NAMESPACE Realm:: FILE RealmTargets.cmake)
configure_file(tools/cmake/RealmConfig.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/RealmConfig.cmake @ONLY)
configure_file(tools/cmake/AcquireRealmDependency.cmake ${CMAKE_CURRENT_BINARY_DIR}/AcquireRealmDependency.cmake @ONLY)

# Make the project importable from the install directory
install(EXPORT realm
        NAMESPACE Realm::
        FILE RealmTargets.cmake
        DESTINATION share/cmake/Realm
        COMPONENT devel
        )

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/RealmConfig.cmake
        DESTINATION share/cmake/Realm
        COMPONENT devel
        )

install(FILES tools/cmake/AcquireRealmDependency.cmake
        DESTINATION share/cmake/Realm
        COMPONENT devel
        )

if(NOT REALM_BUILD_LIB_ONLY AND NOT REALM_NO_TESTS)
    enable_testing()
    add_subdirectory(test)
endif()

if (REALM_BUILD_TEST_CLIENT)
    add_subdirectory(test/client)
endif()

# CPack
set(CPACK_GENERATOR "TGZ")
set(CPACK_PACKAGE_VERSION ${REALM_VERSION})
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)

# Generators that don't support build-time configuration selection.
# Others like Xcode and Visual Studio can determine the package name
# based on the build configuration at build-time.
set(_single_config_generators
    Ninja
    "Unix Makefiles"
)
if(CMAKE_GENERATOR IN_LIST _single_config_generators)
    set(CPACK_PACKAGE_NAME "realm-${CMAKE_BUILD_TYPE}")
else()
    set(CPACK_PACKAGE_NAME "realm-\${CPACK_BUILD_CONFIG}")
endif()

include(CPack)
cpack_add_component(runtime DEPENDS runtime)
cpack_add_component(devel DEPENDS devel)
