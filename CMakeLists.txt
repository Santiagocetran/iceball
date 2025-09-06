cmake_minimum_required(VERSION 3.16)

project(iceball
    VERSION 0.2.1
    DESCRIPTION "Iceball - A game engine and game built upon the classic Ace of Spades experience"
    LANGUAGES C
)

# Set C standard
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

# Add necessary compiler definitions for modern systems
add_compile_definitions(_GNU_SOURCE)
add_compile_definitions(_USE_MATH_DEFINES)

# Set default build type if not specified
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build" FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

# Set build type specific flags
set(CMAKE_C_FLAGS_DEBUG "-g -O0 -DDEBUG")
set(CMAKE_C_FLAGS_RELEASE "-O3 -DNDEBUG")
set(CMAKE_C_FLAGS_RELWITHDEBINFO "-O2 -g -DNDEBUG")
set(CMAKE_C_FLAGS_MINSIZEREL "-Os -DNDEBUG")

# Set module path for custom find modules
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# Modern compiler feature detection and flags
if(CMAKE_C_COMPILER_ID STREQUAL "GNU" OR CMAKE_C_COMPILER_ID STREQUAL "Clang")
    # Keep debugging symbols even in Release builds
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -g")
    # Add common compiler flags
    add_compile_options(-fno-strict-aliasing -Wall -Wextra)
endif()

# Enable colored output for supported compilers
if(CMAKE_C_COMPILER_ID STREQUAL "GNU" AND CMAKE_C_COMPILER_VERSION VERSION_GREATER_EQUAL "4.9")
    add_compile_options(-fdiagnostics-color=always)
elseif(CMAKE_C_COMPILER_ID STREQUAL "Clang")
    add_compile_options(-fcolor-diagnostics)
endif()

# Windows-specific configuration
if(WIN32)
    if(MSVC)
        # MSVC-specific settings
        add_compile_definitions(_CRT_SECURE_NO_WARNINGS)
        # Handle filename.c as C++ for MSVC compatibility
        set_source_files_properties(src/filename.c PROPERTIES LANGUAGE CXX)
        # Set MSVC-specific flags
        set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} /W3")
        set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /O2")
    elseif(MINGW)
        # MinGW-specific settings
        add_compile_definitions(__MINGW32__)
    endif()
    
    # Windows-specific definitions
    add_compile_definitions(WIN32 _WIN32)
elseif(APPLE)
    # macOS-specific settings
    add_compile_definitions(APPLE __APPLE__)
    find_library(COCOA_LIBRARY Cocoa)
    find_library(IOKIT_LIBRARY IOKit)
    find_library(COREVIDEO_LIBRARY CoreVideo)
elseif(UNIX)
    # Unix/Linux-specific settings
    add_compile_definitions(UNIX __UNIX__)
    # Check for X11
    find_package(X11 QUIET)
    if(X11_FOUND)
        add_compile_definitions(HAVE_X11)
    endif()
endif()

# Set prefix paths for Windows builds
if(MINGW)
    set(CMAKE_PREFIX_PATH "dist/mingw/enet;dist/mingw/sdl2;dist/mingw/lua51;dist/mingw/sackit;dist/mingw/zlib" CACHE PATH "" FORCE)
elseif(MSVC)
    set(CMAKE_PREFIX_PATH "dist/msvc/enet;dist/msvc/sdl2;dist/msvc/lua51;dist/msvc/sackit;dist/msvc/zlib" CACHE PATH "" FORCE)
endif()

# Find required packages
find_package(ENet REQUIRED)
find_package(SDL2 REQUIRED)
find_package(ZLIB REQUIRED)
find_package(OpenGL REQUIRED)

# Find additional system libraries
find_package(Threads REQUIRED)
find_library(MATH_LIBRARY m)
find_library(DL_LIBRARY dl)

# Try to find LuaJIT first, fallback to regular Lua
find_package(LuaJIT QUIET)
if(LUAJIT_FOUND)
    add_compile_definitions(USE_LUAJIT)
    set(LUA_LIBRARIES ${LUA_LIBRARY} m)
else()
    find_package(Lua REQUIRED)
endif()

# Add sackit submodule
add_subdirectory(external/sackit)
set(sackit_LIBRARIES sackit)
set(sackit_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/external/sackit)

# Source file organization
file(GLOB LUA_FILES 
    src/lua* 
    src/external/bit.c
)

set(MAIN_FILES
    src/dsp.c
    src/img.c
    src/json.c
    src/logtxt.c
    src/main.c
    src/map.c
    src/model.c
    src/network.c
    src/path.c
    src/png.c
    src/random.c
    src/vecmath.c
    src/wav.c
)

set(GL_FILES
    src/gl/glad.c
    src/gl/render.c
    src/gl/render_img.c
)

# Organize source files in IDE
source_group("Lua" FILES ${LUA_FILES})
source_group("OpenGL" FILES ${GL_FILES})
source_group("Source" FILES ${MAIN_FILES})

# Main iceball executable
add_executable(iceball ${MAIN_FILES} ${LUA_FILES} ${GL_FILES})

# Set include directories for the target
target_include_directories(iceball PRIVATE
    include
    ${ENet_INCLUDE_DIRS}
    ${sackit_INCLUDE_DIRS}
    ${ZLIB_INCLUDE_DIRS}
    ${SDL2_INCLUDE_DIR}
    ${LUA_INCLUDE_DIR}
)

# Link libraries using modern target-based approach
target_link_libraries(iceball PRIVATE
    Threads::Threads
    ${MATH_LIBRARY}
    ${DL_LIBRARY}
    ${ENet_LIBRARIES}
    ${ZLIB_LIBRARIES}
    sackit
    ${LUA_LIBRARIES}
    ${SDL2_LIBRARIES}
    ${OPENGL_LIBRARIES}
)

# Dedicated server executable (no OpenGL)
add_executable(iceball-dedi EXCLUDE_FROM_ALL ${MAIN_FILES} ${LUA_FILES})

# Set include directories for dedicated server
target_include_directories(iceball-dedi PRIVATE
    include
    ${ENet_INCLUDE_DIRS}
    ${sackit_INCLUDE_DIRS}
    ${ZLIB_INCLUDE_DIRS}
    ${LUA_INCLUDE_DIR}
)

# Link libraries for dedicated server (no OpenGL/SDL2)
target_link_libraries(iceball-dedi PRIVATE
    Threads::Threads
    ${MATH_LIBRARY}
    ${DL_LIBRARY}
    ${ENet_LIBRARIES}
    ${ZLIB_LIBRARIES}
    ${LUA_LIBRARIES}
)

# Set compile definitions for dedicated server
target_compile_definitions(iceball-dedi PRIVATE DEDI)

# Function to copy runtime dependencies on Windows
function(copy_runtime_dependency target_name source_path)
    add_custom_command(TARGET ${target_name} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "${PROJECT_SOURCE_DIR}/${source_path}"
            $<TARGET_FILE_DIR:${target_name}>
        COMMENT "Copying ${source_path} to output directory"
    )
endfunction()

# Copy Windows runtime dependencies
if(MINGW)
    copy_runtime_dependency(iceball "dist/mingw/sdl2/bin/libSDL2.dll")
    copy_runtime_dependency(iceball "dist/mingw/lua51/bin/liblua.dll")
    copy_runtime_dependency(iceball "dist/mingw/zlib/bin/libzlib.dll")
elseif(MSVC)
    copy_runtime_dependency(iceball "dist/msvc/sdl2/bin/SDL2.dll")
    copy_runtime_dependency(iceball "dist/msvc/lua51/bin/lua.dll")
    copy_runtime_dependency(iceball "dist/msvc/zlib/bin/zlib.dll")
endif()

# Build options
option(BUILD_DEDICATED_SERVER "Build dedicated server executable" ON)
option(BUILD_TESTS "Build test suite" OFF)
option(ENABLE_DEBUG_SYMBOLS "Enable debug symbols in Release builds" ON)
option(ENABLE_SANITIZERS "Enable address and undefined behavior sanitizers" OFF)
option(ENABLE_COVERAGE "Enable code coverage reporting" OFF)

# Set debug symbols for Release builds if requested
if(ENABLE_DEBUG_SYMBOLS AND CMAKE_BUILD_TYPE STREQUAL "Release")
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -g")
endif()

# Enable sanitizers if requested
if(ENABLE_SANITIZERS AND CMAKE_C_COMPILER_ID STREQUAL "GNU")
    add_compile_options(-fsanitize=address -fsanitize=undefined)
    add_link_options(-fsanitize=address -fsanitize=undefined)
endif()

# Enable code coverage if requested
if(ENABLE_COVERAGE AND CMAKE_C_COMPILER_ID STREQUAL "GNU")
    add_compile_options(--coverage)
    add_link_options(--coverage)
endif()

# Install data files
install(DIRECTORY pkg/ DESTINATION share/iceball
    FILES_MATCHING PATTERN "*.lua" PATTERN "*.json" PATTERN "*.png" 
    PATTERN "*.tga" PATTERN "*.wav" PATTERN "*.it" PATTERN "*.kv6" 
    PATTERN "*.pmf" PATTERN "*.frag" PATTERN "*.vert"
)

# Install documentation
install(FILES README.md LICENCE-GPLv3.txt DESTINATION share/doc/iceball)

# Package configuration
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/iceball-config-version.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY AnyNewerVersion
)

# Set package variables
set(ICEBALL_INCLUDE_DIRS "${CMAKE_CURRENT_SOURCE_DIR}/include")
set(ICEBALL_LIBRARIES "iceball::iceball")

# Export targets for use by other projects
install(TARGETS iceball
    EXPORT iceballTargets
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
)

if(BUILD_DEDICATED_SERVER)
    install(TARGETS iceball-dedi
        EXPORT iceballTargets
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib
    )
endif()

# Create and install config files
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/iceball-config.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/iceball-config.cmake"
    @ONLY
)

install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/iceball-config.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/iceball-config-version.cmake"
    DESTINATION lib/cmake/iceball
)

install(EXPORT iceballTargets
    FILE iceballTargets.cmake
    DESTINATION lib/cmake/iceball
)

# Print build configuration summary
message(STATUS "")
message(STATUS "=== Iceball Build Configuration ===")
message(STATUS "Version: ${PROJECT_VERSION}")
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")
message(STATUS "C compiler: ${CMAKE_C_COMPILER}")
message(STATUS "C standard: ${CMAKE_C_STANDARD}")
message(STATUS "Dedicated server: ${BUILD_DEDICATED_SERVER}")
message(STATUS "Tests: ${BUILD_TESTS}")
message(STATUS "Debug symbols: ${ENABLE_DEBUG_SYMBOLS}")
message(STATUS "Sanitizers: ${ENABLE_SANITIZERS}")
message(STATUS "Code coverage: ${ENABLE_COVERAGE}")
if(LUAJIT_FOUND)
    message(STATUS "Lua implementation: LuaJIT")
else()
    message(STATUS "Lua implementation: Lua")
endif()
if(X11_FOUND)
    message(STATUS "X11 support: Yes")
else()
    message(STATUS "X11 support: No")
endif()
message(STATUS "==================================")
message(STATUS "")
