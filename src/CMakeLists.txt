
add_library(glfw ${GLFW_LIBRARY_TYPE}
                 "${GLFW_SOURCE_DIR}/include/GLFW/glfw3.h"
                 "${GLFW_SOURCE_DIR}/include/GLFW/glfw3native.h"
                 internal.h platform.h mappings.h
                 context.c init.c input.c monitor.c platform.c vulkan.c window.c
                 egl_context.c osmesa_context.c null_platform.h null_joystick.h
                 null_init.c null_monitor.c null_window.c null_joystick.c)

# The time, thread and module code is shared between all backends on a given OS,
# including the null backend, which still needs those bits to be functional
if (APPLE)
    target_sources(glfw PRIVATE cocoa_time.h cocoa_time.c posix_thread.h
                                posix_module.c posix_thread.c)
elseif (WIN32)
    target_sources(glfw PRIVATE win32_time.h win32_thread.h win32_module.c
                                win32_time.c win32_thread.c)
else()
    target_sources(glfw PRIVATE posix_time.h posix_thread.h posix_module.c
                                posix_time.c posix_thread.c)
endif()

if (GLFW_MAPPING)
    add_custom_target(update_mappings
        COMMAND "${CMAKE_COMMAND}" -P "${GLFW_SOURCE_DIR}/CMake/GenerateMappings.cmake" mappings.h.in mappings.h
        WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
        COMMENT "Updating gamepad mappings from upstream repository"
        SOURCES mappings.h.in "${GLFW_SOURCE_DIR}/CMake/GenerateMappings.cmake"
        VERBATIM)

    set_target_properties(update_mappings PROPERTIES FOLDER "GLFW3")

    source_group(src FILES mappings.h.in)
endif()

if (GLFW_BUILD_COCOA)
    enable_language(OBJC)
    target_compile_definitions(glfw PRIVATE _GLFW_COCOA)
    target_sources(glfw PRIVATE cocoa_platform.h cocoa_joystick.h cocoa_init.m
                                cocoa_joystick.m cocoa_monitor.m cocoa_window.m
                                nsgl_context.m)
endif()

if (GLFW_BUILD_WIN32)
    target_compile_definitions(glfw PRIVATE _GLFW_WIN32)
    target_sources(glfw PRIVATE win32_platform.h win32_joystick.h win32_init.c
                                win32_joystick.c win32_monitor.c win32_window.c
                                wgl_context.c)
endif()

if (GLFW_BUILD_X11)
    target_compile_definitions(glfw PRIVATE _GLFW_X11)
    target_sources(glfw PRIVATE x11_platform.h xkb_unicode.h x11_init.c
                                x11_monitor.c x11_window.c xkb_unicode.c
                                glx_context.c)
endif()

if (GLFW_BUILD_WAYLAND)
    target_compile_definitions(glfw PRIVATE _GLFW_WAYLAND)
    target_sources(glfw PRIVATE wl_platform.h wl_init.c
                                wl_monitor.c wl_window.c)
endif()

if (GLFW_BUILD_X11 OR GLFW_BUILD_WAYLAND)
    if (CMAKE_SYSTEM_NAME STREQUAL "Linux")
        target_sources(glfw PRIVATE linux_joystick.h linux_joystick.c)
    endif()
    target_sources(glfw PRIVATE posix_poll.h posix_poll.c)
endif()

if (GLFW_BUILD_WAYLAND)
    include(CheckIncludeFiles)
    include(CheckFunctionExists)
    check_function_exists(memfd_create HAVE_MEMFD_CREATE)
    if (HAVE_MEMFD_CREATE)
        target_compile_definitions(glfw PRIVATE HAVE_MEMFD_CREATE)
    endif()

    find_program(WAYLAND_SCANNER_EXECUTABLE NAMES wayland-scanner)
    if (NOT WAYLAND_SCANNER_EXECUTABLE)
        message(FATAL_ERROR "Failed to find wayland-scanner")
    endif()

    macro(generate_wayland_protocol protocol_file)
        set(protocol_path "${GLFW_SOURCE_DIR}/deps/wayland/${protocol_file}")

        string(REGEX REPLACE "\\.xml$" "-client-protocol.h" header_file ${protocol_file})
        string(REGEX REPLACE "\\.xml$" "-client-protocol-code.h" code_file ${protocol_file})

        add_custom_command(OUTPUT ${header_file}
            COMMAND "${WAYLAND_SCANNER_EXECUTABLE}" client-header "${protocol_path}" ${header_file}
            DEPENDS "${protocol_path}"
            VERBATIM)

        add_custom_command(OUTPUT ${code_file}
            COMMAND "${WAYLAND_SCANNER_EXECUTABLE}" private-code "${protocol_path}" ${code_file}
            DEPENDS "${protocol_path}"
            VERBATIM)

        target_sources(glfw PRIVATE ${header_file} ${code_file})
    endmacro()

    generate_wayland_protocol("wayland.xml")
    generate_wayland_protocol("viewporter.xml")
    generate_wayland_protocol("xdg-shell.xml")
    generate_wayland_protocol("idle-inhibit-unstable-v1.xml")
    generate_wayland_protocol("pointer-constraints-unstable-v1.xml")
    generate_wayland_protocol("relative-pointer-unstable-v1.xml")
    generate_wayland_protocol("fractional-scale-v1.xml")
    generate_wayland_protocol("xdg-activation-v1.xml")
    generate_wayland_protocol("xdg-decoration-unstable-v1.xml")
endif()

if (WIN32 AND GLFW_BUILD_SHARED_LIBRARY)
    configure_file(glfw.rc.in glfw.rc @ONLY)
    target_sources(glfw PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/glfw.rc")
endif()

if (UNIX AND GLFW_BUILD_SHARED_LIBRARY)
    # On Unix-like systems, shared libraries can use the soname system.
    set(GLFW_LIB_NAME glfw)
else()
    set(GLFW_LIB_NAME glfw3)
endif()
set(GLFW_LIB_NAME_SUFFIX "")

set_target_properties(glfw PROPERTIES
                      OUTPUT_NAME ${GLFW_LIB_NAME}
                      VERSION ${GLFW_VERSION_MAJOR}.${GLFW_VERSION_MINOR}
                      SOVERSION ${GLFW_VERSION_MAJOR}
                      POSITION_INDEPENDENT_CODE ON
                      C_STANDARD 99
                      C_EXTENSIONS OFF
                      DEFINE_SYMBOL _GLFW_BUILD_DLL
                      FOLDER "GLFW3")

target_include_directories(glfw PUBLIC
                           "$<BUILD_INTERFACE:${GLFW_SOURCE_DIR}/include>"
                           "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>")
target_include_directories(glfw PRIVATE
                           "${GLFW_SOURCE_DIR}/src"
                           "${GLFW_BINARY_DIR}/src")
target_link_libraries(glfw PRIVATE Threads::Threads)

if (GLFW_BUILD_WIN32)
    list(APPEND glfw_PKG_LIBS "-lgdi32")
endif()

if (GLFW_BUILD_COCOA)
    target_link_libraries(glfw PRIVATE "-framework Cocoa"
                                       "-framework IOKit"
                                       "-framework CoreFoundation"
                                       "-framework QuartzCore")

    set(glfw_PKG_DEPS "")
    set(glfw_PKG_LIBS "-framework Cocoa -framework IOKit -framework CoreFoundation -framework QuartzCore")
endif()

if (GLFW_BUILD_WAYLAND)
    include(FindPkgConfig)

    pkg_check_modules(Wayland REQUIRED
        wayland-client>=0.2.7
        wayland-cursor>=0.2.7
        wayland-egl>=0.2.7
        xkbcommon>=0.5.0)

    target_include_directories(glfw PRIVATE ${Wayland_INCLUDE_DIRS})

    if (NOT CMAKE_SYSTEM_NAME STREQUAL "Linux")
        find_package(EpollShim)
        if (EPOLLSHIM_FOUND)
            target_include_directories(glfw PRIVATE ${EPOLLSHIM_INCLUDE_DIRS})
            target_link_libraries(glfw PRIVATE ${EPOLLSHIM_LIBRARIES})
        endif()
    endif()
endif()

if (GLFW_BUILD_X11)
    find_package(X11 REQUIRED)
    target_include_directories(glfw PRIVATE "${X11_X11_INCLUDE_PATH}")

    # Check for XRandR (modern resolution switching and gamma control)
    if (NOT X11_Xrandr_INCLUDE_PATH)
        message(FATAL_ERROR "RandR headers not found; install libxrandr development package")
    endif()
    target_include_directories(glfw PRIVATE "${X11_Xrandr_INCLUDE_PATH}")

    # Check for Xinerama (legacy multi-monitor support)
    if (NOT X11_Xinerama_INCLUDE_PATH)
        message(FATAL_ERROR "Xinerama headers not found; install libxinerama development package")
    endif()
    target_include_directories(glfw PRIVATE "${X11_Xinerama_INCLUDE_PATH}")

    # Check for Xkb (X keyboard extension)
    if (NOT X11_Xkb_INCLUDE_PATH)
        message(FATAL_ERROR "XKB headers not found; install X11 development package")
    endif()
    target_include_directories(glfw PRIVATE "${X11_Xkb_INCLUDE_PATH}")

    # Check for Xcursor (cursor creation from RGBA images)
    if (NOT X11_Xcursor_INCLUDE_PATH)
        message(FATAL_ERROR "Xcursor headers not found; install libxcursor development package")
    endif()
    target_include_directories(glfw PRIVATE "${X11_Xcursor_INCLUDE_PATH}")

    # Check for XInput (modern HID input)
    if (NOT X11_Xi_INCLUDE_PATH)
        message(FATAL_ERROR "XInput headers not found; install libxi development package")
    endif()
    target_include_directories(glfw PRIVATE "${X11_Xi_INCLUDE_PATH}")

    # Check for X Shape (custom window input shape)
    if (NOT X11_Xshape_INCLUDE_PATH)
        message(FATAL_ERROR "X Shape headers not found; install libxext development package")
    endif()
    target_include_directories(glfw PRIVATE "${X11_Xshape_INCLUDE_PATH}")
endif()

if (UNIX AND NOT APPLE)
    find_library(RT_LIBRARY rt)
    mark_as_advanced(RT_LIBRARY)
    if (RT_LIBRARY)
        target_link_libraries(glfw PRIVATE "${RT_LIBRARY}")
        list(APPEND glfw_PKG_LIBS "-lrt")
    endif()

    find_library(MATH_LIBRARY m)
    mark_as_advanced(MATH_LIBRARY)
    if (MATH_LIBRARY)
        target_link_libraries(glfw PRIVATE "${MATH_LIBRARY}")
        list(APPEND glfw_PKG_LIBS "-lm")
    endif()

    if (CMAKE_DL_LIBS)
        target_link_libraries(glfw PRIVATE "${CMAKE_DL_LIBS}")
        list(APPEND glfw_PKG_LIBS "-l${CMAKE_DL_LIBS}")
    endif()
endif()

if (WIN32)
    if (GLFW_USE_HYBRID_HPG)
        target_compile_definitions(glfw PRIVATE _GLFW_USE_HYBRID_HPG)
    endif()
endif()

# Enable a reasonable set of warnings
# NOTE: The order matters here, Clang-CL matches both MSVC and Clang
if (MSVC)
    target_compile_options(glfw PRIVATE "/W3")
elseif (CMAKE_C_COMPILER_ID STREQUAL "GNU" OR
        CMAKE_C_COMPILER_ID STREQUAL "Clang" OR
        CMAKE_C_COMPILER_ID STREQUAL "AppleClang")

    target_compile_options(glfw PRIVATE "-Wall")
endif()

if (GLFW_BUILD_WIN32)
    target_compile_definitions(glfw PRIVATE UNICODE _UNICODE)
endif()

# Workaround for the MS CRT deprecating parts of the standard library
if (MSVC OR CMAKE_C_SIMULATE_ID STREQUAL "MSVC")
    target_compile_definitions(glfw PRIVATE _CRT_SECURE_NO_WARNINGS)
endif()

# Workaround for -std=c99 on Linux disabling _DEFAULT_SOURCE (POSIX 2008 and more)
if (CMAKE_SYSTEM_NAME STREQUAL "Linux")
    target_compile_definitions(glfw PRIVATE _DEFAULT_SOURCE)
endif()

if (GLFW_BUILD_SHARED_LIBRARY)
    if (WIN32)
        if (MINGW)
            # Remove the lib prefix on the DLL (but not the import library)
            set_target_properties(glfw PROPERTIES PREFIX "")

            # Add a suffix to the import library to avoid naming conflicts
            set_target_properties(glfw PROPERTIES IMPORT_SUFFIX "dll.a")
        else()
            # Add a suffix to the import library to avoid naming conflicts
            set_target_properties(glfw PROPERTIES IMPORT_SUFFIX "dll.lib")
        endif()
        set (GLFW_LIB_NAME_SUFFIX "dll")

        target_compile_definitions(glfw INTERFACE GLFW_DLL)
    endif()

    if (MINGW)
        # Enable link-time exploit mitigation features enabled by default on MSVC
        include(CheckCCompilerFlag)
        include(CMakePushCheckState)

        # Compatibility with data execution prevention (DEP)
        cmake_push_check_state()
        set(CMAKE_REQUIRED_FLAGS "-Wl,--nxcompat")
        check_c_compiler_flag("" _GLFW_HAS_DEP)
        if (_GLFW_HAS_DEP)
            target_link_libraries(glfw PRIVATE "-Wl,--nxcompat")
        endif()
        cmake_pop_check_state()

        # Compatibility with address space layout randomization (ASLR)
        cmake_push_check_state()
        set(CMAKE_REQUIRED_FLAGS "-Wl,--dynamicbase")
        check_c_compiler_flag("" _GLFW_HAS_ASLR)
        if (_GLFW_HAS_ASLR)
            target_link_libraries(glfw PRIVATE "-Wl,--dynamicbase")
        endif()
        cmake_pop_check_state()

        # Compatibility with 64-bit address space layout randomization (ASLR)
        cmake_push_check_state()
        set(CMAKE_REQUIRED_FLAGS "-Wl,--high-entropy-va")
        check_c_compiler_flag("" _GLFW_HAS_64ASLR)
        if (_GLFW_HAS_64ASLR)
            target_link_libraries(glfw PRIVATE "-Wl,--high-entropy-va")
        endif()
        cmake_pop_check_state()
    endif()

    if (UNIX)
        # Hide symbols not explicitly tagged for export from the shared library
        target_compile_options(glfw PRIVATE "-fvisibility=hidden")
    endif()
endif()

list(JOIN glfw_PKG_DEPS " " deps)
list(JOIN glfw_PKG_LIBS " " libs)

set(GLFW_PKG_CONFIG_REQUIRES_PRIVATE "${deps}" CACHE INTERNAL
    "GLFW pkg-config Requires.private")
set(GLFW_PKG_CONFIG_LIBS_PRIVATE "${libs}" CACHE INTERNAL
    "GLFW pkg-config Libs.private")

configure_file("${GLFW_SOURCE_DIR}/CMake/glfw3.pc.in" glfw3.pc @ONLY)

if (GLFW_INSTALL)
    install(TARGETS glfw
            EXPORT glfwTargets
            RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
            ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
            LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}")
endif()

#--------------------------------------------------------------------
# Define solution tree organization
#--------------------------------------------------------------------

# Collect all source and header files in the current directory
file(GLOB GLFW_SRC_FILES "*.c" "*.cpp" "*.mm" "*.m")
file(GLOB GLFW_HEADER_FILES "*.h" "*.hpp"
    "${GLFW_SOURCE_DIR}/include/GLFW/glfw3.h"
    "${GLFW_SOURCE_DIR}/include/GLFW/glfw3native.h"
)

# Group them in the IDE
source_group(src FILES ${GLFW_SRC_FILES})
source_group(include FILES ${GLFW_HEADER_FILES})