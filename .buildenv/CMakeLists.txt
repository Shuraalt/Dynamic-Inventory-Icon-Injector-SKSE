# This file configures the "build" of this SKSE plugin
# - how to compile the C++ plugin (in addition to settings found in CMakePresets.json)
# - where to output the resulting .dll file

# It's recommended to set a minimum CMake version.
# If you use CMake features from higher versions, update this to match.
cmake_minimum_required(VERSION 3.21)

# [BEGIN set_project_name]
# Set your project name. This will be the name of your SKSE .dll file.
project(DynamicInventoryIconInjector VERSION 0.0.1 LANGUAGES CXX)

file(GLOB_RECURSE sources CONFIGURE_DEPENDS ../src/*.cpp ../src/*.h ../api/DIII_API.h)

# [END set_project_name]

# #
# YOU DO NOT NEED TO EDIT ANYTHING BELOW HERE
# #

# If you're not using a mod manager, you probably want the SKSE plugin to go
# inside of your Skyrim "Data" folder.
#
# To do this automatically, set the `SKYRIM_FOLDER` environment variable
# to the path of your Skyrim Special Edition folder
if(DEFINED ENV{SKYRIM_FOLDER} AND IS_DIRECTORY "$ENV{SKYRIM_FOLDER}/Data")
    set(OUTPUT_FOLDER "$ENV{SKYRIM_FOLDER}/Data")
endif()

# If you're using Mod Organizer 2 or Vortex, you might want this to go inside
# of your "mods" folder, inside of a subfolder named "<your mod>".
#
# To do this automatically, set the `SKYRIM_MODS_FOLDER` environment variable
# to the path of your "mods" folder
if(DEFINED ENV{SKYRIM_MODS_FOLDER} AND IS_DIRECTORY "$ENV{SKYRIM_MODS_FOLDER}")
    set(OUTPUT_FOLDER "$ENV{SKYRIM_MODS_FOLDER}/${PROJECT_NAME}")
endif()

# If neither environment variable is set, no copy step is added and the built
# DLL stays in the build folder (build/release/ next to CMakePresets.json).

#include($ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake)

# CommonLibSSE-NG is fetched from source instead of using the old vcpkg port.
# The old port (CharmedBaryon/CommonLibSSE, last commit 2024-09) cannot read the
# format-5 Address Library used by Skyrim 1.7.99+, so any plugin built against it
# fails with "failed to open address library file" on 1.7.x.
include(FetchContent)
set(BUILD_TESTS OFF CACHE BOOL "" FORCE)
# PCH.h does `#undef ENABLE_SKYRIM_VR`, so build CommonLib without VR as well.
# Otherwise it would publicly define ENABLE_SKYRIM_VR and the two sides would disagree.
set(ENABLE_SKYRIM_VR OFF CACHE BOOL "" FORCE)
# Same trampoline option the old port enabled.
set(SKSE_SUPPORT_XBYAK ON CACHE BOOL "" FORCE)
# Not needed, and it vendors a MinHook component that could clash with vcpkg's minhook.
set(SKSE_SUPPORT_PATCH_SAFETY OFF CACHE BOOL "" FORCE)
FetchContent_Declare(
    CommonLibSSE
    GIT_REPOSITORY https://github.com/alandtse/CommonLibSSE-NG.git
    GIT_TAG v13.1.0
    GIT_SHALLOW TRUE
)
FetchContent_MakeAvailable(CommonLibSSE)
# add_commonlibsse_plugin() is only exposed through find_package or prebuilt builds,
# so include the helper explicitly when building the library from source.
include("${commonlibsse_SOURCE_DIR}/cmake/CommonLibSSE.cmake")

# Setup your SKSE plugin as an SKSE plugin!
add_commonlibsse_plugin(${PROJECT_NAME} SOURCES ${sources}) # <--- specifies plugin.cpp
target_compile_features(${PROJECT_NAME} PRIVATE cxx_std_23) # <--- use C++23 standard
target_precompile_headers(${PROJECT_NAME} PRIVATE ../PCH.h) # <--- PCH.h is required!

find_package(minhook CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE minhook::minhook)
find_package(boost_algorithm CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE Boost::algorithm)
find_package(xbyak CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE xbyak::xbyak)
find_package(jsoncpp CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE jsoncpp_static)

# When your SKSE .dll is compiled, this will automatically copy the .dll into your mods folder.
# Only works if you configure DEPLOY_ROOT above (or set the SKYRIM_MODS_FOLDER environment variable)
if(DEFINED OUTPUT_FOLDER)
    # If you specify an <OUTPUT_FOLDER> (including via environment variables)
    # then we'll copy your mod files into Skyrim or a mod manager for you!

    # Copy the SKSE plugin .dll files into the SKSE/Plugins/ folder
    set(DLL_FOLDER "${OUTPUT_FOLDER}/SKSE/Plugins")

    message(STATUS "SKSE plugin output folder: ${DLL_FOLDER}")

    add_custom_command(
        TARGET "${PROJECT_NAME}"
        POST_BUILD
        COMMAND "${CMAKE_COMMAND}" -E make_directory "${DLL_FOLDER}"
        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "$<TARGET_FILE:${PROJECT_NAME}>" "${DLL_FOLDER}/$<TARGET_FILE_NAME:${PROJECT_NAME}>"
        VERBATIM
    )

    # If you perform a "Debug" build, also copy .pdb file (for debug symbols)
    if(CMAKE_BUILD_TYPE STREQUAL "Debug" OR CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
        add_custom_command(
            TARGET "${PROJECT_NAME}"
            POST_BUILD
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "$<TARGET_PDB_FILE:${PROJECT_NAME}>" "${DLL_FOLDER}/$<TARGET_PDB_FILE_NAME:${PROJECT_NAME}>"
            VERBATIM
        )
    endif()
endif()
