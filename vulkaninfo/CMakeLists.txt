# CMakeLists.txt file for building Vulkaninfo

set(VULKANINFO_INCLUDE_DIRS
    ${CMAKE_CURRENT_BINARY_DIR}
    ${CMAKE_CURRENT_BINARY_DIR/..}
   )

if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    include(FindPkgConfig)
    option(BUILD_WSI_XCB_SUPPORT "Build XCB WSI support" ON)
    option(BUILD_WSI_XLIB_SUPPORT "Build Xlib WSI support" ON)
    option(BUILD_WSI_WAYLAND_SUPPORT "Build Wayland WSI support" ON)
    option(BUILD_WSI_MIR_SUPPORT "Build Mir WSI support" ON)
    set(VULKANINFO_WSI_SELECTION "XCB" CACHE STRING "Select WSI target for vulkaninfo (XCB, XLIB, WAYLAND, MIR, DISPLAY)")

    if (BUILD_WSI_XCB_SUPPORT)
        find_package(XCB REQUIRED)
    endif()

    if (BUILD_WSI_XLIB_SUPPORT)
        find_package(X11 REQUIRED)
    endif()
    
    if (BUILD_WSI_WAYLAND_SUPPORT)
        find_package(Wayland REQUIRED)
    endif()

    if (BUILD_WSI_MIR_SUPPORT)
        find_package(Mir REQUIRED)
    endif()

    if (VULKANINFO_WSI_SELECTION STREQUAL "XCB")
        if (NOT BUILD_WSI_XCB_SUPPORT)
            message( FATAL_ERROR "Selected XCB for vulkaninfo build but not building Xcb support" )
        endif()
        set(VULKANINFO_INCLUDE_DIRS
            ${XCB_INCLUDE_DIRS}
            ${VULKANINFO_INCLUDE_DIRS}
        )
        link_libraries(${XCB_LIBRARIES})
        add_definitions(-DVK_USE_PLATFORM_XCB_KHR)
    elseif (VULKANINFO_WSI_SELECTION STREQUAL "XLIB")
        if (NOT BUILD_WSI_XLIB_SUPPORT)
            message( FATAL_ERROR "Selected XLIB for vulkaninfo build but not building Xlib support" )
        endif()
        set(VULKANINFO_INCLUDE_DIRS
            ${XLIB_INCLUDE_DIRS}
            ${VULKANINFO_INCLUDE_DIRS}
        )
        link_libraries(${XLIB_LIBRARIES})
        add_definitions(-DVK_USE_PLATFORM_XLIB_KHR)
    elseif (VULKANINFO_WSI_SELECTION STREQUAL "WAYLAND")
        if (NOT BUILD_WSI_WAYLAND_SUPPORT)
            message( FATAL_ERROR "Selected Wayland for vulkaninfo build but not building Wayland support" )
        endif()
        set(VULKANINFO_INCLUDE_DIRS
            ${WAYLAND_CLIENT_INCLUDE_DIRS}
            ${VULKANINFO_INCLUDE_DIRS}
        )
        link_libraries(${WAYLAND_CLIENT_LIBRARIES})
        add_definitions(-DVK_USE_PLATFORM_WAYLAND_KHR)
    elseif (VULKANINFO_WSI_SELECTION STREQUAL "MIR")
        if (NOT BUILD_WSI_MIR_SUPPORT)
            message( FATAL_ERROR "Selected MIR for vulkaninfo build but not building Mir support" )
        endif()
        set(VULKANINFO_INCLUDE_DIRS
            ${MIR_INCLUDE_DIRS}
            ${VULKANINFO_INCLUDE_DIRS}
        )
        link_libraries(${MIR_LIBRARIES})
        add_definitions(-DVK_USE_PLATFORM_MIR_KHR)
    endif()
endif()

include_directories(
    ${VULKANINFO_INCLUDE_DIRS}
    )

if(WIN32)
    add_executable(vulkaninfo vulkaninfo.c vulkaninfo.rc)
elseif(APPLE)
    add_executable(vulkaninfo vulkaninfo.c
        ${CMAKE_CURRENT_SOURCE_DIR}/macOS/vulkaninfo/metal_view.m
        ${CMAKE_CURRENT_SOURCE_DIR}/macOS/vulkaninfo/metal_view.h
        )
else()
    add_executable(vulkaninfo vulkaninfo.c)
endif()

# Predefine loader root as a cmake cache variable for cmake-gui
set(LOADER_REPO_ROOT "LOADER-NOTFOUND" CACHE PATH "Absolute path to the root of the loader repository")

if(LOADER_REPO_ROOT)
    message(STATUS "Using user-supplied path to locate Vulkan")
    if(WIN32)
        set (LOADER_SEARCH_PATHS
            "${LOADER_REPO_ROOT}/${BUILDTGT_DIR}/loader/${DEBUG_DECORATION}"
            "${LOADER_REPO_ROOT}/${BUILDTGT_DIR}/loader/${RELEASE_DECORATION}"
            )
    else()
        set (LOADER_SEARCH_PATHS
            "${LOADER_REPO_ROOT}/${BUILDTGT_DIR}/loader"
            )
    endif()

    find_library(LIBVK NAMES vulkan vulkan-1
        HINTS ${LOADER_SEARCH_PATHS}
        )
    message(STATUS "Found Vulkan: ${LIBVK}")
    if(APPLE)
        target_link_libraries(vulkaninfo ${LIBVK} "-framework AppKit -framework QuartzCore")
        target_include_directories(vulkaninfo PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/macOS/vulkaninfo)
    else()
        target_link_libraries(vulkaninfo ${LIBVK})
    endif()
 else()
    message(STATUS "Using find_package to locate Vulkan")
    find_package(Vulkan)
    set (LIBVK "Vulkan::Vulkan")
    if(APPLE)
        target_link_libraries(vulkaninfo ${LIBVK} "-framework AppKit -framework QuartzCore")
        target_include_directories(vulkaninfo PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/macOS/vulkaninfo)
    else()
        target_link_libraries(vulkaninfo ${LIBVK})
    endif()
endif()

if(APPLE)
    set_target_properties(vulkaninfo PROPERTIES
        INSTALL_RPATH "@loader_path/../lib"
    )
    install(TARGETS vulkaninfo DESTINATION ${CMAKE_CURRENT_BINARY_DIR})
endif()

# Create vulkaninfo application bundle for MacOS
if(APPLE)
    include(${CMAKE_CURRENT_SOURCE_DIR}/macOS/vulkaninfo.cmake)
endif()

if(WIN32)
    target_compile_definitions(vulkaninfo PUBLIC -DVK_USE_PLATFORM_WIN32_KHR -DWIN32_LEAN_AND_MEAN -D_CRT_SECURE_NO_WARNINGS)

    # Use static MSVCRT libraries
    foreach(configuration in CMAKE_C_FLAGS_DEBUG CMAKE_C_FLAGS_MINSIZEREL CMAKE_C_FLAGS_RELEASE CMAKE_C_FLAGS_RELWITHDEBINFO
                             CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELEASE CMAKE_CXX_FLAGS_RELWITHDEBINFO)
        if(${configuration} MATCHES "/MD")
            string(REGEX REPLACE "/MD" "/MT" ${configuration} "${${configuration}}")
        endif()
    endforeach()

    file(COPY vulkaninfo.vcxproj.user DESTINATION ${CMAKE_BINARY_DIR}/demos)
elseif(APPLE)
    add_definitions(-DVK_USE_PLATFORM_MACOS_MVK)
endif()

if(${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
    install(TARGETS vulkaninfo RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

