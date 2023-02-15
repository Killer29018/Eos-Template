cmake_minimum_required (VERSION 3.18)

find_program(glslc_exec NAMES glslc HINTS Vulkan::glslc)
set (CMAKE_CXX_STANDARD 20)

project (Template)

file (GLOB_RECURSE Files CONFIGURE_DEPENDS "src/*.cpp")

set (outputDirectory "${PROJECT_SOURCE_DIR}/bin")
if (CMAKE_BUILD_TYPE MATCHES Debug)
    message ("Debug build")
    set (outputDirectory "${outputDirectory}/debug")
elseif (CMAKE_BUILD_TYPE MATCHES Release)
    message ("Release build")
    set (outputDirectory "${outputDirectory}/release")
endif ()
set (CMAKE_RUNTIME_OUTPUT_DIRECTORY ${outputDirectory})

add_subdirectory (vendor)

add_executable(${PROJECT_NAME} ${Files})

target_include_directories(${PROJECT_NAME} PRIVATE "vendor/Eos/eos")
target_link_libraries(${PROJECT_NAME}
    PRIVATE
        "-std=c++2a -g -Wall -Wextra -Wpedantic"
    PUBLIC 
        Eos
    )

if (WIN32)
    target_compile_definitions(${PROJECT_NAME} PUBLIC EOS_PLATFORM_WINDOWS)
elseif (UNIX)
    target_compile_definitions(${PROJECT_NAME} PUBLIC EOS_PLATFORM_LINUX)
endif ()

set (InputRes "${PROJECT_SOURCE_DIR}/res")
set (OutputRes "${outputDirectory}/res")

if (NOT EXISTS ${OutputRes}/Shaders)
    add_custom_command(
        TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND
        ${CMAKE_COMMAND} -E make_directory
        ${OutputRes}/Shaders
        )
endif()

file (GLOB_RECURSE Shaders CONFIGURE_DEPENDS
    ${InputRes}/Shaders/*.vert
    ${InputRes}/Shaders/*.frag
    ${InputRes}/Shaders/*.comp)

foreach (Shader ${Shaders})
    get_filename_component (File ${Shader} NAME)

    # Compile Shaders
    add_custom_command(
        TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND
        ${glslc_exec}
        ${InputRes}/Shaders/${File}
        -o ${InputRes}/Shaders/${File}.spv
        COMMENT "Compiled ${InputRes}/Shaders/${File} to ${InputRes}/Shaders/${File}.spv")

    # Copy Shaders
    add_custom_command(
        TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND
        cmake -E copy
        ${InputRes}/Shaders/${File}.spv
        ${OutputRes}/Shaders/
        COMMENT "Copied ${InputRes}/Shaders/${File}.spv to ${OutputRes}/Shaders/${File}.spv")
endforeach ()
