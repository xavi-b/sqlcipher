cmake_minimum_required(VERSION 3.16)

# Extract version from VERSION file
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/VERSION")
    file(READ "${CMAKE_CURRENT_SOURCE_DIR}/VERSION" SQLCIPHER_VERSION)
    string(STRIP "${SQLCIPHER_VERSION}" SQLCIPHER_VERSION)
else()
    set(SQLCIPHER_VERSION "3.49.2")
endif()

project(SQLCipher
    VERSION ${SQLCIPHER_VERSION}
    LANGUAGES C
    DESCRIPTION "SQLCipher is an SQLite extension that provides 256 bit AES encryption of database files"
)

# Set default build type if not specified
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build." FORCE)
endif()

# Build options
option(SQLCIPHER_ENABLE_SHARED "Build shared library" OFF)
option(SQLCIPHER_ENABLE_STATIC "Build static library" ON)
option(SQLCIPHER_ENABLE_CLI "Build command line interface" ON)
option(SQLCIPHER_ENABLE_TESTS "Build test programs" OFF)

# Library base name options (customizable output names)
set(SQLCIPHER_LIB_NAME "sqlcipher" CACHE STRING "Base name for library")
set(SQLCIPHER_CLI_NAME "sqlcipher" CACHE STRING "Name for CLI executable")

# Crypto backend selection (only one should be enabled)
set(SQLCIPHER_CRYPTO_BACKEND "openssl" CACHE STRING
    "Crypto backend: openssl, nss, libtomcrypt, commoncrypto")
set_property(CACHE SQLCIPHER_CRYPTO_BACKEND PROPERTY STRINGS
    "openssl" "nss" "libtomcrypt" "commoncrypto")

# Standard setup
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)

# Ensure C99 features are available
add_definitions(-D_GNU_SOURCE)
if(NOT WIN32)
    add_definitions(-D_POSIX_C_SOURCE=200809L)
endif()

# Platform-specific settings
if(WIN32)
    add_definitions(-DSQLITE_OS_WIN=1)
    if(MSVC)
        add_compile_options(/W3)
        add_definitions(-D_CRT_SECURE_NO_WARNINGS -D_CRT_SECURE_NO_DEPRECATE)
        # Force include stdint.h for MSVC (Visual Studio 2010+)
        add_compile_options(/FI stdint.h)
    endif()
else()
    add_definitions(-DSQLITE_OS_UNIX=1)
    if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_C_COMPILER_ID MATCHES "Clang")
        add_compile_options(-Wall -Wextra)
        # Ensure stdint.h and other C99 headers are available
        add_compile_options(-std=c99)
        # Force include stdint.h to ensure uint64_t types are available
        add_compile_options(-include stdint.h)
    endif()
endif()

# Find dependencies based on crypto backend
if(SQLCIPHER_CRYPTO_BACKEND STREQUAL "openssl")
    find_package(OpenSSL REQUIRED)
    set(CRYPTO_LIBRARIES OpenSSL::SSL OpenSSL::Crypto)
    add_definitions(-DSQLCIPHER_CRYPTO_OPENSSL=1)
elseif(SQLCIPHER_CRYPTO_BACKEND STREQUAL "nss")
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(NSS REQUIRED nss)
    set(CRYPTO_LIBRARIES ${NSS_LIBRARIES})
    include_directories(${NSS_INCLUDE_DIRS})
    add_definitions(-DSQLCIPHER_CRYPTO_NSS=1)
    link_directories(${NSS_LIBRARY_DIRS})
elseif(SQLCIPHER_CRYPTO_BACKEND STREQUAL "libtomcrypt")
    find_library(TOMCRYPT_LIBRARY tomcrypt)
    if(NOT TOMCRYPT_LIBRARY)
        message(FATAL_ERROR "LibTomCrypt library not found")
    endif()
    set(CRYPTO_LIBRARIES ${TOMCRYPT_LIBRARY})
    add_definitions(-DSQLCIPHER_CRYPTO_LIBTOMCRYPT=1)
elseif(SQLCIPHER_CRYPTO_BACKEND STREQUAL "commoncrypto")
    if(NOT APPLE)
        message(FATAL_ERROR "CommonCrypto is only available on Apple platforms")
    endif()
    find_library(COMMONCRYPTO_LIBRARY CommonCrypto)
    if(NOT COMMONCRYPTO_LIBRARY)
        message(FATAL_ERROR "CommonCrypto framework not found")
    endif()
    set(CRYPTO_LIBRARIES ${COMMONCRYPTO_LIBRARY})
    add_definitions(-DSQLCIPHER_CRYPTO_CC=1)
else()
    message(FATAL_ERROR "Invalid crypto backend: ${SQLCIPHER_CRYPTO_BACKEND}")
endif()

# Core SQLite/SQLCipher compiler definitions
add_definitions(
    -DSQLITE_HAS_CODEC=1
    -DSQLITE_EXTRA_INIT=sqlcipher_extra_init
    -DSQLITE_EXTRA_SHUTDOWN=sqlcipher_extra_shutdown
    -DSQLITE_TEMP_STORE=2
    -DSQLITE_THREADSAFE=1
)

# System libraries
set(SYSTEM_LIBRARIES)
if(NOT WIN32)
    list(APPEND SYSTEM_LIBRARIES m pthread dl)
    if(NOT APPLE)
        list(APPEND SYSTEM_LIBRARIES rt)
    endif()
endif()

# Source files
set(SQLCIPHER_SOURCES
    src/alter.c
    src/analyze.c
    src/attach.c
    src/auth.c
    src/backup.c
    src/bitvec.c
    src/btmutex.c
    src/btree.c
    src/build.c
    src/callback.c
    src/complete.c
    src/ctime.c
    src/date.c
    src/dbpage.c
    src/dbstat.c
    src/delete.c
    src/expr.c
    src/fault.c
    src/fkey.c
    src/func.c
    src/global.c
    src/hash.c
    src/insert.c
    src/json.c
    src/legacy.c
    src/loadext.c
    src/main.c
    src/malloc.c
    src/mem0.c
    src/mem1.c
    src/mem2.c
    src/mem3.c
    src/mem5.c
    src/memdb.c
    src/memjournal.c
    src/mutex.c
    src/mutex_noop.c
    src/mutex_unix.c
    src/mutex_w32.c
    src/notify.c
    src/os.c
    src/os_kv.c
    src/os_unix.c
    src/os_win.c
    src/pager.c
    src/pcache.c
    src/pcache1.c
    src/pragma.c
    src/prepare.c
    src/printf.c
    src/random.c
    src/resolve.c
    src/rowset.c
    src/select.c
    src/status.c
    src/table.c
    src/threads.c
    src/tokenize.c
    src/treeview.c
    src/trigger.c
    src/update.c
    src/upsert.c
    src/utf.c
    src/util.c
    src/vacuum.c
    src/vdbe.c
    src/vdbeapi.c
    src/vdbeaux.c
    src/vdbeblob.c
    src/vdbemem.c
    src/vdbesort.c
    src/vdbetrace.c
    src/vdbevtab.c
    src/vtab.c
    src/wal.c
    src/walker.c
    src/where.c
    src/wherecode.c
    src/whereexpr.c
    src/window.c
    # SQLCipher sources
    src/sqlcipher.c
)

# Add appropriate crypto backend source
if(SQLCIPHER_CRYPTO_BACKEND STREQUAL "openssl")
    list(APPEND SQLCIPHER_SOURCES src/crypto_openssl.c)
elseif(SQLCIPHER_CRYPTO_BACKEND STREQUAL "nss")
    list(APPEND SQLCIPHER_SOURCES src/crypto_nss.c)
elseif(SQLCIPHER_CRYPTO_BACKEND STREQUAL "libtomcrypt")
    list(APPEND SQLCIPHER_SOURCES src/crypto_libtomcrypt.c)
elseif(SQLCIPHER_CRYPTO_BACKEND STREQUAL "commoncrypto")
    list(APPEND SQLCIPHER_SOURCES src/crypto_cc.c)
endif()

# Add include directories for extensions
include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    ${CMAKE_CURRENT_BINARY_DIR}
)

# Generate required header files
# Find TCL for header generation
find_program(TCLSH_EXECUTABLE tclsh tclsh8.6 tclsh8.5 tclsh8.4)

# Build mksourceid tool for sqlite3.h generation
add_executable(mksourceid ${CMAKE_CURRENT_SOURCE_DIR}/tool/mksourceid.c)
set_target_properties(mksourceid PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
)

# Generate sqlite3.h from sqlite.h.in (contains SQLCipher function declarations)
if(TCLSH_EXECUTABLE)
    add_custom_command(
        OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/sqlite3.h
        COMMAND ${TCLSH_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/tool/mksqlite3h.tcl ${CMAKE_CURRENT_SOURCE_DIR} -o ${CMAKE_CURRENT_BINARY_DIR}/sqlite3.h
        DEPENDS
            ${CMAKE_CURRENT_SOURCE_DIR}/src/sqlite.h.in
            ${CMAKE_CURRENT_SOURCE_DIR}/VERSION
            ${CMAKE_CURRENT_SOURCE_DIR}/manifest
            mksourceid
            ${CMAKE_CURRENT_SOURCE_DIR}/tool/mksqlite3h.tcl
        COMMENT "Generating sqlite3.h from sqlite.h.in"
    )
else()
    # Fallback: copy sqlite.h.in to sqlite3.h
    add_custom_command(
        OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/sqlite3.h
        COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/src/sqlite.h.in ${CMAKE_CURRENT_BINARY_DIR}/sqlite3.h
        DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/sqlite.h.in
        COMMENT "Copying sqlite.h.in to sqlite3.h (fallback)"
    )
endif()

# Build lemon parser generator
add_executable(lemon ${CMAKE_CURRENT_SOURCE_DIR}/tool/lemon.c)
set_target_properties(lemon PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
)

# Build mkkeywordhash tool
add_executable(mkkeywordhash ${CMAKE_CURRENT_SOURCE_DIR}/tool/mkkeywordhash.c)
set_target_properties(mkkeywordhash PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
)

# Generate parse.c and parse.h from parse.y
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/parse.c ${CMAKE_CURRENT_BINARY_DIR}/parse.h
    COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/src/parse.y ${CMAKE_CURRENT_BINARY_DIR}/parse.y
    COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/tool/lempar.c ${CMAKE_CURRENT_BINARY_DIR}/lempar.c
    COMMAND $<TARGET_FILE:lemon> ${CMAKE_CURRENT_BINARY_DIR}/parse.y
    DEPENDS lemon ${CMAKE_CURRENT_SOURCE_DIR}/src/parse.y ${CMAKE_CURRENT_SOURCE_DIR}/tool/lempar.c
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "Generating parse.c and parse.h from parse.y"
)

# Generate keywordhash.h
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/keywordhash.h
    COMMAND $<TARGET_FILE:mkkeywordhash> > ${CMAKE_CURRENT_BINARY_DIR}/keywordhash.h
    DEPENDS mkkeywordhash
    COMMENT "Generating keywordhash.h"
        )

    # Generate opcodes.h (requires TCL)
if(TCLSH_EXECUTABLE)
    # Create CMake script to generate opcodes.h
    set(OPCODES_CMAKE_SCRIPT ${CMAKE_CURRENT_BINARY_DIR}/generate_opcodes.cmake)
    file(WRITE ${OPCODES_CMAKE_SCRIPT}
        "# Generate opcodes.h\n"
        "execute_process(\n"
        "    COMMAND \"${CMAKE_COMMAND}\" -E cat \"${CMAKE_CURRENT_BINARY_DIR}/parse.h\" \"${CMAKE_CURRENT_SOURCE_DIR}/src/vdbe.c\"\n"
        "    COMMAND \"${TCLSH_EXECUTABLE}\" \"${CMAKE_CURRENT_SOURCE_DIR}/tool/mkopcodeh.tcl\"\n"
        "    OUTPUT_FILE \"${CMAKE_CURRENT_BINARY_DIR}/opcodes.h\"\n"
        "    RESULT_VARIABLE opcodes_result\n"
        ")\n"
        "if(NOT opcodes_result EQUAL 0)\n"
        "    message(FATAL_ERROR \"Failed to generate opcodes.h\")\n"
        "endif()\n"
    )

    add_custom_command(
        OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/opcodes.h
        COMMAND ${CMAKE_COMMAND} -P ${OPCODES_CMAKE_SCRIPT}
        DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/parse.h ${CMAKE_CURRENT_SOURCE_DIR}/src/vdbe.c ${CMAKE_CURRENT_SOURCE_DIR}/tool/mkopcodeh.tcl
        COMMENT "Generating opcodes.h"
    )

    # Create CMake script to generate opcodes.c
    set(OPCODES_C_CMAKE_SCRIPT ${CMAKE_CURRENT_BINARY_DIR}/generate_opcodes_c.cmake)
    file(WRITE ${OPCODES_C_CMAKE_SCRIPT}
        "# Generate opcodes.c\n"
        "execute_process(\n"
        "    COMMAND \"${TCLSH_EXECUTABLE}\" \"${CMAKE_CURRENT_SOURCE_DIR}/tool/mkopcodec.tcl\" \"${CMAKE_CURRENT_BINARY_DIR}/opcodes.h\"\n"
        "    OUTPUT_FILE \"${CMAKE_CURRENT_BINARY_DIR}/opcodes.c\"\n"
        "    RESULT_VARIABLE opcodes_c_result\n"
        ")\n"
        "if(NOT opcodes_c_result EQUAL 0)\n"
        "    message(FATAL_ERROR \"Failed to generate opcodes.c\")\n"
        "endif()\n"
    )

    add_custom_command(
        OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/opcodes.c
        COMMAND ${CMAKE_COMMAND} -P ${OPCODES_C_CMAKE_SCRIPT}
        DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/opcodes.h ${CMAKE_CURRENT_SOURCE_DIR}/tool/mkopcodec.tcl
        COMMENT "Generating opcodes.c"
    )
    set(GENERATED_OPCODES_SOURCES ${CMAKE_CURRENT_BINARY_DIR}/opcodes.c)
else()
    message(WARNING "TCL not found - opcodes.h and opcodes.c will not be generated. Some functionality may be missing.")
    set(GENERATED_OPCODES_SOURCES "")
endif()

# Add generated sources to the source list
list(APPEND SQLCIPHER_SOURCES
    ${CMAKE_CURRENT_BINARY_DIR}/parse.c
    ${GENERATED_OPCODES_SOURCES}
)

# Create custom target for generated files
add_custom_target(generate_headers
    DEPENDS
    ${CMAKE_CURRENT_BINARY_DIR}/sqlite3.h
    ${CMAKE_CURRENT_BINARY_DIR}/parse.h
    ${CMAKE_CURRENT_BINARY_DIR}/keywordhash.h
    ${CMAKE_CURRENT_BINARY_DIR}/opcodes.h
)

# Check if sources exist
foreach(src_file ${SQLCIPHER_SOURCES})
    if(NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/${src_file}")
        message(WARNING "Source file ${src_file} not found")
    endif()
endforeach()

# Shared library
if(SQLCIPHER_ENABLE_SHARED)
    add_library(${SQLCIPHER_LIB_NAME} SHARED ${SQLCIPHER_SOURCES})

    # Ensure generated headers are built first
    add_dependencies(${SQLCIPHER_LIB_NAME} generate_headers)

    target_link_libraries(${SQLCIPHER_LIB_NAME}
        ${CRYPTO_LIBRARIES}
        ${SYSTEM_LIBRARIES}
    )

    set_target_properties(${SQLCIPHER_LIB_NAME} PROPERTIES
        VERSION ${PROJECT_VERSION}
        SOVERSION ${PROJECT_VERSION_MAJOR}
        OUTPUT_NAME ${SQLCIPHER_LIB_NAME}
        POSITION_INDEPENDENT_CODE ON
    )

    if(WIN32)
        set_target_properties(${SQLCIPHER_LIB_NAME} PROPERTIES
            WINDOWS_EXPORT_ALL_SYMBOLS ON
        )
    endif()

    # Set rpath for Unix-like systems
    if(NOT WIN32)
        set_target_properties(${SQLCIPHER_LIB_NAME} PROPERTIES
            INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib"
            INSTALL_RPATH_USE_LINK_PATH TRUE
        )
    endif()
endif()

# Static library
if(SQLCIPHER_ENABLE_STATIC)
    add_library(${SQLCIPHER_LIB_NAME}_static STATIC ${SQLCIPHER_SOURCES})

    add_dependencies(${SQLCIPHER_LIB_NAME}_static generate_headers)

    target_link_libraries(${SQLCIPHER_LIB_NAME}_static
        ${CRYPTO_LIBRARIES}
        ${SYSTEM_LIBRARIES}
    )

    set_target_properties(${SQLCIPHER_LIB_NAME}_static PROPERTIES
        OUTPUT_NAME ${SQLCIPHER_LIB_NAME}
        POSITION_INDEPENDENT_CODE ON
    )
endif()

# Command line interface
if(SQLCIPHER_ENABLE_CLI)
    # Generate shell.c from shell.c.in or use it directly
    set(SHELL_C_IN "${CMAKE_CURRENT_SOURCE_DIR}/src/shell.c.in")
    set(SHELL_C "${CMAKE_CURRENT_BINARY_DIR}/shell.c")

    if(EXISTS "${SHELL_C_IN}")
        # Try to find tclsh for generating shell.c
        find_program(TCLSH_EXECUTABLE tclsh tclsh8.6 tclsh8.5 tclsh8.4)

        if(TCLSH_EXECUTABLE AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/tool/mkshellc.tcl")
            # Generate shell.c using the original TCL script (preferred method)
            add_custom_command(
                OUTPUT ${SHELL_C}
                COMMAND ${TCLSH_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/tool/mkshellc.tcl ${SHELL_C}
                DEPENDS ${SHELL_C_IN} ${CMAKE_CURRENT_SOURCE_DIR}/tool/mkshellc.tcl
                WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
                COMMENT "Generating shell.c using TCL script (full-featured)"
            )
            set(CLI_SOURCES ${SHELL_C})
            message(STATUS "Using TCL to generate shell.c (recommended)")
        else()
            # Cross-platform fallback: copy shell.c.in to shell.c
            # Note: shell.c.in is valid C code and will work on all platforms
            add_custom_command(
                OUTPUT ${SHELL_C}
                COMMAND ${CMAKE_COMMAND} -E copy ${SHELL_C_IN} ${SHELL_C}
                DEPENDS ${SHELL_C_IN}
                COMMENT "Using shell.c.in directly (cross-platform fallback)"
            )
            set(CLI_SOURCES ${SHELL_C})
            if(NOT TCLSH_EXECUTABLE)
                message(STATUS "TCL not found - using fallback method for shell.c")
                message(STATUS "  To get full shell features, install TCL: https://www.tcl.tk/")
            endif()
        endif()
    else()
        message(WARNING "CLI source ${SHELL_C_IN} not found, skipping CLI build")
        set(SQLCIPHER_ENABLE_CLI OFF)
    endif()
endif()

if(SQLCIPHER_ENABLE_CLI)
    add_executable(${SQLCIPHER_CLI_NAME}_bin ${CLI_SOURCES})

    # Link with shared library if available, otherwise static
    if(SQLCIPHER_ENABLE_SHARED)
        target_link_libraries(${SQLCIPHER_CLI_NAME}_bin ${SQLCIPHER_LIB_NAME})
    elseif(SQLCIPHER_ENABLE_STATIC)
        target_link_libraries(${SQLCIPHER_CLI_NAME}_bin ${SQLCIPHER_LIB_NAME}_static)
        target_link_libraries(${SQLCIPHER_CLI_NAME}_bin ${CRYPTO_LIBRARIES} ${SYSTEM_LIBRARIES})
    endif()

    # Add shell-specific definitions
    target_compile_definitions(${SQLCIPHER_CLI_NAME}_bin PRIVATE
        -DSQLITE_ENABLE_EXPLAIN_COMMENTS=1
        -DSQLITE_ENABLE_UNKNOWN_SQL_FUNCTION=1
        -DSQLITE_ENABLE_STMTVTAB=1
        -DSQLITE_ENABLE_DBPAGE_VTAB=1
        -DSQLITE_ENABLE_DBSTAT_VTAB=1
        -DSQLITE_ENABLE_BYTECODE_VTAB=1
        -DSQLITE_ENABLE_OFFSET_SQL_FUNC=1
        -DSQLITE_DQS=0
    )

    # Add readline support if available
    find_package(PkgConfig QUIET)
    if(PkgConfig_FOUND)
        pkg_check_modules(READLINE readline)
        if(READLINE_FOUND)
            target_compile_definitions(${SQLCIPHER_CLI_NAME}_bin PRIVATE -DHAVE_READLINE=1)
            target_link_libraries(${SQLCIPHER_CLI_NAME}_bin ${READLINE_LIBRARIES})
            target_include_directories(${SQLCIPHER_CLI_NAME}_bin PRIVATE ${READLINE_INCLUDE_DIRS})
        endif()
    endif()

    set_target_properties(${SQLCIPHER_CLI_NAME}_bin PROPERTIES
        OUTPUT_NAME ${SQLCIPHER_CLI_NAME}
        POSITION_INDEPENDENT_CODE ON
    )
endif()

# Installation
include(GNUInstallDirs)

if(SQLCIPHER_ENABLE_SHARED)
    install(TARGETS ${SQLCIPHER_LIB_NAME}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    )
endif()

if(SQLCIPHER_ENABLE_STATIC)
    install(TARGETS ${SQLCIPHER_LIB_NAME}_static
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    )
endif()

if(SQLCIPHER_ENABLE_CLI)
    install(TARGETS ${SQLCIPHER_CLI_NAME}_bin
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    )
endif()

# Install headers
install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/sqlite3.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    RENAME ${SQLCIPHER_LIB_NAME}.h
)

# Create pkg-config file
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/sqlite3.pc.in"
    "${CMAKE_CURRENT_BINARY_DIR}/sqlcipher.pc"
    @ONLY
)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/sqlcipher.pc"
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig"
)

# Tests (if enabled)
if(SQLCIPHER_ENABLE_TESTS)
    enable_testing()
    # Add test targets here if needed
endif()

# Summary
message(STATUS "")
message(STATUS "SQLCipher ${PROJECT_VERSION} Configuration Summary:")
message(STATUS "  Crypto Backend: ${SQLCIPHER_CRYPTO_BACKEND}")
message(STATUS "  Build shared library: ${SQLCIPHER_ENABLE_SHARED}")
message(STATUS "  Build static library: ${SQLCIPHER_ENABLE_STATIC}")
message(STATUS "  Build CLI: ${SQLCIPHER_ENABLE_CLI}")
message(STATUS "  Library name: ${SQLCIPHER_LIB_NAME}")
message(STATUS "  Header name: ${SQLCIPHER_LIB_NAME}.h")
message(STATUS "  CLI name: ${SQLCIPHER_CLI_NAME}")
message(STATUS "  Install prefix: ${CMAKE_INSTALL_PREFIX}")
message(STATUS "")