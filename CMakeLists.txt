# Main project build script
cmake_minimum_required(VERSION 3.16)

PROJECT(CataclysmBN)

SET(CMAKE_MODULE_PATH
    ${CMAKE_MODULE_PATH}
    ${CMAKE_SOURCE_DIR}/CMakeModules
)

INCLUDE(AstyleFormatSource)

SET(CMAKE_TLS_VERIFY ON)

# Build options
option(TILES        "Build graphical tileset version."   "OFF")
option(CURSES       "Build curses version."   "ON")
option(SOUND        "Support for in-game sounds & music."   "OFF")
option(BACKTRACE    "Support for printing stack backtraces on crash"   "ON")
option(LIBBACKTRACE "Print backtrace with libbacktrace."    "OFF")
option(USE_HOME_DIR "Use user's home directory for save files."   "ON")
set(LANGUAGES "" CACHE STRING "Compile localization files for specified languages. List of language ids separated by semicolon. Set to 'all' or leave empty to compile all, set to 'none' to compile none.")
option(DYNAMIC_LINKING "Use dynamic linking. Or use static to remove MinGW dependency instead."   "ON")
option(USE_UNITY_BUILD "Use unity build." "OFF")
option(USE_PCH_HEADER "Use precompiled PCH header." "OFF")
option(JSON_FORMAT  "Build JSON formatter" "OFF")
option(CATA_CCACHE  "Try to find and build with ccache" "ON")
option(CATA_CLANG_TIDY_PLUGIN "Build Cata's custom clang-tidy plugin" "OFF")
set(CATA_CLANG_TIDY_INCLUDE_DIR "" CACHE STRING "Path to internal clang-tidy headers required for plugin (e.g. ClangTidy.h)")
set(CATA_CHECK_CLANG_TIDY "" CACHE STRING "Path to check_clang_tidy.py for plugin tests")
set(GIT_BINARY       "" CACHE STRING "Git binary name or path.")
set(PREFIX           "" CACHE STRING "Location of Data & GFX directories")

include(CTest)

include(GetGitRevisionDescription)
git_describe(GIT_VERSION)

MESSAGE("\n * Cataclysm: Bright Nights is a roguelike set in a post-apocalyptic world.")
MESSAGE("   _________            __                   .__                                ")
MESSAGE("   \\_   ___ \\ _____   _/  |_ _____     ____  |  |   ___.__   ______  _____      ")
MESSAGE("   /    \\  \\/ \\__  \\  \\   __\\\\__  \\  _/ ___\\ |  |  <   |  | /  ___/ /     \\     ")
MESSAGE("   \\     \\____ / __ \\_ |  |   / __ \\_\\  \\___ |  |__ \\___  | \\___ \\ |  Y Y  \\    ")
MESSAGE("    \\______  /\(____  / |__|  \(____  / \\___  >|____/ / ____|/____  >|__|_|  /    ")
MESSAGE("           \\/      \\/             \\/      \\/        \\/          \\/       \\/     ")
MESSAGE("                                --= Bright Nights =--")

MESSAGE("\n * https://github.com/cataclysmbnteam/Cataclysm-BN\n")

MESSAGE(STATUS "${PROJECT} build environment -- \n")

MESSAGE(STATUS "Build realm is                : ${CMAKE_SYSTEM_NAME} ${CMAKE_HOST_SYSTEM_PROCESSOR}")

IF(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Debug)
ENDIF(NOT CMAKE_BUILD_TYPE)

ADD_DEFINITIONS(-DCMAKE)

if (NOT ${GIT_VERSION} MATCHES GIT-NOTFOUND)
    string(REPLACE "-NOTFOUND" "" GIT_VERSION ${GIT_VERSION})
    FILE(WRITE ${CMAKE_SOURCE_DIR}/src/version.h
        "// NOLINT(cata-header-guard)\n\#define VERSION \"${GIT_VERSION}\"\n")
    MESSAGE(STATUS "${PROJECT_NAME} build version is : ${GIT_VERSION}\n")
    ADD_DEFINITIONS(-DGIT_VERSION)
ELSE (NOT ${GIT_VERSION} MATCHES GIT-NOTFOUND)
    MESSAGE("")
ENDIF(NOT ${GIT_VERSION} MATCHES GIT-NOTFOUND)

#OS Check Placeholders. Will be used for BINDIST
IF (${CMAKE_SYSTEM_NAME} MATCHES Linux)
    SET(_OS_LINUX_ 1)
ENDIF (${CMAKE_SYSTEM_NAME} MATCHES Linux)

IF (${CMAKE_SYSTEM_NAME} MATCHES FreeBSD)
    SET(_OS_FREEBSD_ 1)
ENDIF (${CMAKE_SYSTEM_NAME} MATCHES FreeBSD)

IF(${CMAKE_SYSTEM_NAME} MATCHES Darwin)
    SET(_OS_DARWIN_ 1)
ENDIF(${CMAKE_SYSTEM_NAME} MATCHES Darwin)

include(CheckCXXCompilerFlag)

#FIXME: Add dest build choice: m32 for 32 bit or m64 for 64 bit version
#add_definitions("-m32")
#SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -m32")
#SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -m32")
#SET(CMAKE_SHARED_LIBRARY_C_FLAGS "${CMAKE_SHARED_LIBRARY_C_FLAGS} -m32")
#SET(CMAKE_SHARED_LIBRARY_CXX_FLAGS "${CMAKE_SHARED_LIBRARY_CXX_FLAGS} -m32")

IF (NOT DYNAMIC_LINKING)
    SET(CMAKE_FIND_LIBRARY_SUFFIXES ".a;.dll.a")
    SET(BUILD_SHARED_LIBRARIES OFF)
    check_cxx_compiler_flag (-static HAVE_STATIC_FLAG)
    IF(HAVE_STATIC_FLAG)
      SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -static")
    ENDIF(HAVE_STATIC_FLAG)
    # Workaround for cmake link library guesser
    SET(CMAKE_EXE_LINK_DYNAMIC_C_FLAGS)       # remove -Wl,-Bdynamic
    SET(CMAKE_EXE_LINK_DYNAMIC_CXX_FLAGS)
    SET(CMAKE_SHARED_LIBRARY_C_FLAGS)         # remove -fPIC
    SET(CMAKE_SHARED_LIBRARY_CXX_FLAGS)
    SET(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS)    # remove -rdynamic
    SET(CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS)
ELSE (NOT DYNAMIC_LINKING)
    IF(MINGW AND NOT CMAKE_CXX_COMPILER_ID MATCHES "Clang")
      # Avoid depending on MinGW runtime DLLs
      SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -static-libgcc -static-libstdc++")
    ENDIF(MINGW AND NOT CMAKE_CXX_COMPILER_ID MATCHES "Clang")
ENDIF (NOT DYNAMIC_LINKING)

# System specific actions
IF (${CMAKE_SYSTEM_NAME} MATCHES Linux OR ${CMAKE_SYSTEM_NAME} MATCHES FreeBSD)
    IF(NOT DATA_PREFIX)
      SET( DATA_PREFIX ${CMAKE_INSTALL_PREFIX}/share/cataclysm-bn)
    ENDIF(NOT DATA_PREFIX)
    IF(NOT BIN_PREFIX)
      SET( BIN_PREFIX ${CMAKE_INSTALL_PREFIX}/bin)
    ENDIF(NOT BIN_PREFIX)
    IF(NOT DESKTOP_ENTRY_PATH)
      SET( DESKTOP_ENTRY_PATH ${CMAKE_INSTALL_PREFIX}/share/applications)
    ENDIF(NOT DESKTOP_ENTRY_PATH)
    IF(NOT PIXMAPS_ENTRY_PATH)
      SET( PIXMAPS_ENTRY_PATH ${CMAKE_INSTALL_PREFIX}/share/icons/hicolor)
    ENDIF(NOT PIXMAPS_ENTRY_PATH)
    IF(NOT PIXMAPS_UNITY_ENTRY_PATH)
      SET( PIXMAPS_UNITY_ENTRY_PATH ${CMAKE_INSTALL_PREFIX}/share/icons/ubuntu-mono-dark)
    ENDIF(NOT PIXMAPS_UNITY_ENTRY_PATH)
    IF(NOT MANPAGE_ENTRY_PATH)
      SET( MANPAGE_ENTRY_PATH ${CMAKE_INSTALL_PREFIX}/share/man)
    ENDIF(NOT MANPAGE_ENTRY_PATH)
ENDIF (${CMAKE_SYSTEM_NAME} MATCHES Linux OR ${CMAKE_SYSTEM_NAME} MATCHES FreeBSD)

IF (${CMAKE_SYSTEM_NAME} MATCHES Windows)
    IF(NOT DATA_PREFIX)
      SET(DATA_PREFIX ${CMAKE_INSTALL_PREFIX})
    ENDIF(NOT DATA_PREFIX)
    IF(NOT BIN_PREFIX)
      SET(BIN_PREFIX ${CMAKE_INSTALL_PREFIX})
    ENDIF(NOT BIN_PREFIX)
ENDIF (${CMAKE_SYSTEM_NAME} MATCHES Windows)


MESSAGE(STATUS "${PROJECT_NAME} build options --\n")

# Preset variables
IF(NOT LANGUAGES)
    SET(LANGUAGES all)
ENDIF(NOT LANGUAGES)

IF (GIT_BINARY)
    SET(GIT_EXECUTABLE ${GIT_BINARY})
ELSE (GIT_BINARY)
    FIND_PACKAGE(Git)
    IF (NOT GIT_FOUND)
      MESSAGE(WARNING "Git binary not found. Build version will be set to NULL. Install Git package or use -DGIT_BINARY to set path to git binary.")
    ENDIF (NOT GIT_FOUND)
ENDIF (GIT_BINARY)

IF (PREFIX)
    ADD_DEFINITIONS(-DPREFIX=${PREFIX})
ENDIF (PREFIX)

# Can't compile curses and tiles build's at same time
IF(TILES)
    SET(CURSES OFF)
ENDIF(TILES)

# Set build types and display info
IF(CMAKE_BUILD_TYPE STREQUAL Debug)
    MESSAGE("\n")
    MESSAGE(STATUS "Build ${PROJECT} in development mode --\n")
    MESSAGE(STATUS "Binaries will be located in: " ${CMAKE_SOURCE_DIR})
    SET(CMAKE_VERBOSE_MAKEFILE ON)
    # Since CataclysmDDA does not respect PREFIX for development builds
    # and has funny path handlers, we should create resulting Binaries
    # in the source directory
    SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY
     ${CMAKE_SOURCE_DIR}
      CACHE PATH "Single Directory for all Executables."
    )
    SET(BIN_PREFIX ${CMAKE_SOURCE_DIR})
ELSE (CMAKE_BUILD_TYPE STREQUAL Debug)
    MESSAGE(STATUS "CMAKE_INSTALL_PREFIX          : ${CMAKE_INSTALL_PREFIX}")
    MESSAGE(STATUS "BIN_PREFIX                    : ${BIN_PREFIX}")
    MESSAGE(STATUS "DATA_PREFIX                   : ${DATA_PREFIX}")
    MESSAGE(STATUS "DESKTOP_ENTRY_PATH            : ${DESKTOP_ENTRY_PATH}")
    MESSAGE(STATUS "PIXMAPS_ENTRY_PATH            : ${PIXMAPS_ENTRY_PATH}")
    MESSAGE(STATUS "PIXMAPS_UNITY_ENTRY_PATH      : ${PIXMAPS_UNITY_ENTRY_PATH}")
    MESSAGE(STATUS "MANPAGE_ENTRY_PATH            : ${MANPAGE_ENTRY_PATH}\n")
    ADD_DEFINITIONS(-DRELEASE)
    # Use PREFIX as storage of data,gfx, etc.. Useful only on *nix OS.
    IF (PREFIX AND NOT WIN32)
      ADD_DEFINITIONS(-DDATA_DIR_PREFIX)
    ENDIF (PREFIX AND NOT WIN32)
ENDIF (CMAKE_BUILD_TYPE STREQUAL Debug)
    MESSAGE(STATUS "GIT_BINARY                    : ${GIT_EXECUTABLE}")
    MESSAGE(STATUS "DYNAMIC_LINKING               : ${DYNAMIC_LINKING}")

    MESSAGE(STATUS "TILES                         : ${TILES}")
    MESSAGE(STATUS "CURSES                        : ${CURSES}")
    MESSAGE(STATUS "SOUND                         : ${SOUND}")
    MESSAGE(STATUS "BACKTRACE                     : ${BACKTRACE}")
    MESSAGE(STATUS "USE_HOME_DIR                  : ${USE_HOME_DIR}\n")

    MESSAGE(STATUS "UNITY_BUILD                   : ${USE_UNITY_BUILD}")
    MESSAGE(STATUS "PCH_HEADER                    : ${USE_PCH_HEADER}")

    MESSAGE(STATUS "LANGUAGES                     : ${LANGUAGES}\n")

    MESSAGE(STATUS "See INSTALL file for details and more info --\n")

IF(MSVC)
    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
      add_definitions(-D_AMD64_)
    else()
      add_definitions(-D_X86_)
    endif()
ELSE()
    SET(CATA_WARNINGS
        "-Werror -Wall -Wextra \
       -Wformat-signedness \
       -Wlogical-op \
       -Wmissing-declarations \
       -Wmissing-noreturn \
       -Wnon-virtual-dtor \
       -Wold-style-cast \
       -Woverloaded-virtual \
       -Wpedantic \
       -Wsuggest-override \
       -Wunused-macros \
       -Wzero-as-null-pointer-constant \
       -Wno-unknown-warning-option \
       -Wno-range-loop-analysis")
    IF (NOT ${CMAKE_SYSTEM_NAME} MATCHES Windows)
        SET(CATA_WARNINGS "${CATA_WARNINGS} -Wredundant-decls")
    ENDIF()
    SET(CATA_OTHER_FLAGS "${CATA_OTHER_FLAGS} -fsigned-char")
    # Compact the whitespace in the warning string
    string(REGEX REPLACE "[\t ]+" " " CATA_WARNINGS "${CATA_WARNINGS}")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CATA_WARNINGS} ${CATA_OTHER_FLAGS}")
    SET(CMAKE_CXX_FLAGS_DEBUG "-Og -g")
ENDIF()

SET(CMAKE_CXX_STANDARD 17)

# Force out-of-source build
IF(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
    MESSAGE(FATAL_ERROR
      "This project requires an out of source build. Remove the file 'CMakeCache.txt' found in this directory before continuing, create a separate build
       directory and run 'cmake [options] <srcs>' from there.\nSee INSTALL file for details and more info\n"
    )
ENDIF(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})

#SET(THREADS_USE_PTHREADS_WIN32 True)
SET(CMAKE_THREAD_PREFER_PTHREAD True)
FIND_PACKAGE(Threads REQUIRED)

# Check for build types and libraries
IF(TILES)
    # Find SDL, SDL_ttf & SDL_image for graphical install
    MESSAGE(STATUS "Searching for SDL2 library --")
    FIND_PACKAGE(SDL2)
    IF(NOT SDL2_FOUND)
      MESSAGE(FATAL_ERROR
          "This project requires SDL2 to be installed to be compiled in graphical mode. Please install the SDL2 development libraries, or try compiling
           without the -DTILES=1 for a text only compilation.\nSee INSTALL file for details and more info\n"
      )
    ENDIF(NOT SDL2_FOUND)

    IF (NOT DYNAMIC_LINKING)
      # SDL, SDL_Image, SDL_ttf deps are required for static build
      MESSAGE(STATUS "Searching for SDL deps libraries --")
      FIND_PACKAGE(Freetype REQUIRED)
      FIND_PACKAGE(PNG REQUIRED)
      FIND_PACKAGE(JPEG REQUIRED)
      FIND_PACKAGE(ZLIB REQUIRED)
      FIND_PACKAGE(BZip2 REQUIRED)
    ENDIF (NOT DYNAMIC_LINKING)

    MESSAGE(STATUS "Searching for SDL2_TTF library --")
    FIND_PACKAGE(SDL2_ttf)
    IF(NOT SDL2_TTF_FOUND)
      MESSAGE(FATAL_ERROR
        "This project requires SDL2_ttf to be installed to be compiled in graphical mode. Please install the SDL2_ttf development libraries, or try
         compiling without the -DTILES=1 for a text only compilation.\nSee INSTALL file for details and moreinfo\n"
      )
    ENDIF(NOT SDL2_TTF_FOUND)

    MESSAGE(STATUS "Searching for SDL2_image library --\n")
    FIND_PACKAGE(SDL2_image)
    IF(NOT SDL2_IMAGE_FOUND)
      MESSAGE(FATAL_ERROR
        "This project requires SDL2_image to be installed to be compiled in graphical mode. Please install the SDL2_image development libraries, or try
         compiling without the -DTILES=1 for a text only compilation.\nSee INSTALL file for details and more info\n"
      )
    ENDIF(NOT SDL2_IMAGE_FOUND)
    ADD_DEFINITIONS(-DTILES)
ENDIF(TILES)

IF(CURSES)
    # Find the ncurses library for a text based compile
    MESSAGE(STATUS "Searching for Curses library --\n")
    SET(CURSES_NEED_NCURSES TRUE)
    SET(CURSES_NEED_WIDE TRUE)
    FIND_PACKAGE(Curses)
    IF(NOT CURSES_FOUND)
      MESSAGE(FATAL_ERROR
        "This project requires ncurses to be installed to be compiled in text only mode. Please install the ncurses development libraries, or try compiling
         with the -DTILES=1 for a graphical compilation.\nSee INSTALL file for details and more info\n"
      )
    ENDIF(NOT CURSES_FOUND)
ENDIF(CURSES)

IF(SOUND)
    # You need TILES to be able to use SOUND
    IF(NOT TILES)
      MESSAGE(FATAL_ERROR
        "You must enable graphical support with -DTILES=1 to be able to enable sound support.\nSee INSTALL file for details and more info\n"
      )
    ENDIF(NOT TILES)

    # Sound requires SDL_mixer library
    MESSAGE(STATUS "Searching for SDL2_mixer library --\n")
    FIND_PACKAGE(SDL2_mixer)
    IF(NOT SDL2_MIXER_FOUND)
      MESSAGE(FATAL_ERROR
        "You need the SDL2_mixer development library to be able to compile with sound enabled.\nSee INSTALL file for details and more info\n"
      )
    ENDIF(NOT SDL2_MIXER_FOUND)
ENDIF(SOUND)

IF(BACKTRACE)
    ADD_DEFINITIONS(-DBACKTRACE)
    IF(LIBBACKTRACE)
      ADD_DEFINITIONS(-DLIBBACKTRACE)
    ENDIF(LIBBACKTRACE)
ENDIF(BACKTRACE)

# Ok. Now create build and install recipes
IF(USE_HOME_DIR)
    ADD_DEFINITIONS(-DUSE_HOME_DIR)
ENDIF(USE_HOME_DIR)

add_subdirectory(src)
add_subdirectory(data)
add_subdirectory(lang)
if (NOT MSVC)
    add_subdirectory(src/chkjson)
endif()
add_subdirectory(tests)
if (JSON_FORMAT)
    add_subdirectory(tools/format)
endif()
if (CATA_CLANG_TIDY_PLUGIN)
    add_subdirectory(tools/clang-tidy-plugin)
endif()

CONFIGURE_FILE(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
    IMMEDIATE @ONLY
)

ADD_CUSTOM_TARGET(uninstall
    "${CMAKE_COMMAND}"
    -P "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
)

find_program(CCACHE_FOUND ccache)
if(CCACHE_FOUND AND CATA_CCACHE)
    set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
    set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ccache)
endif()

if(USE_UNITY_BUILD)
    set_property(GLOBAL PROPERTY UNITY_BUILD ON)
endif()
# vim:noet
