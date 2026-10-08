cmake_minimum_required(VERSION 3.15)
project(Cake)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(KISSFFT_TEST OFF CACHE BOOL "Disable KissFFT tests" FORCE)

set(KISSFFT_STATIC ON CACHE BOOL "Build static library" FORCE)

set(FE_EXCLUDE_JOLT ON CACHE BOOL "Disable JOLT" FORCE)


#add_subdirectory(${CMAKE_SOURCE_DIR}/../FenixEngine/src/Engine ${CMAKE_BINARY_DIR}/out/Engine)

set(ENGINE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../FenixEngine")


if(NOT TARGET Engine)
    message(STATUS "Engine target not found. Looking for engine source...")
    
    if(EXISTS "${ENGINE_DIR}/CMakeLists.txt")
        add_subdirectory("${ENGINE_DIR}" FenixEngine)
    # elif(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/FenixEngine/CMakeLists.txt")
    #     add_subdirectory("${CMAKE_CURRENT_SOURCE_DIR}/FenixEngine" FenixEngine)
    else()
        message(FATAL_ERROR "FenixEngine not found!")
    endif()
endif()

set(IMGUI_DIR ${ENGINE_DIR}/external/imgui)

file(GLOB_RECURSE IMGUI_SOURCE_FILES
	${IMGUI_DIR}/imgui.cpp
	${IMGUI_DIR}/backends/imgui_impl_sdl3.cpp
	${IMGUI_DIR}/backends/imgui_impl_sdlgpu3.cpp
	${IMGUI_DIR}/backends/imgui_impl_sdlrenderer3.cpp
	${IMGUI_DIR}/backends/imgui_impl_opengl3.cpp
	# ${IMGUI_DIR}/backends/imgui_impl_vulkan.cpp
	${IMGUI_DIR}/imgui_tables.cpp
	${IMGUI_DIR}/imgui_widgets.cpp
	${IMGUI_DIR}/imgui_demo.cpp
	${IMGUI_DIR}/imgui_draw.cpp
)

add_executable(Cake
	WIN32
	${IMGUI_SOURCE_FILES}
	main.cpp
)

target_include_directories(Cake PRIVATE
	include
	${ENGINE_DIR}
	${ENGINE_DIR}/include
	${ENGINE_DIR}/external
	${ENGINE_DIR}/external/imgui
	${ENGINE_DIR}/external/kissfft
	${ENGINE_DIR}/external/SDL/include
	${ENGINE_DIR}/src/Engine
)

target_link_libraries(Cake PRIVATE
	Engine
)

target_precompile_headers(Cake PRIVATE
	"$<$<COMPILE_LANGUAGE:CXX>:${ENGINE_DIR}/src/Engine/pch.hpp>"
)

if(NOT WIN32 AND NOT EMSCRIPTEN)
find_package(PkgConfig REQUIRED)
pkg_check_modules(PIPEWIRE REQUIRED libpipewire-0.3)
target_include_directories(Cake PRIVATE ${PIPEWIRE_INCLUDE_DIRS})
target_link_libraries(Cake PRIVATE ${PIPEWIRE_LIBRARIES})
target_compile_options(Cake PRIVATE ${PIPEWIRE_CFLAGS_OTHER})

find_package(X11 REQUIRED)

target_link_libraries(Cake PRIVATE X11::X11)
endif()

set_property(TARGET Cake PROPERTY VS_DEBUGGER_WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})

add_custom_command(TARGET Cake POST_BUILD
	COMMAND ${CMAKE_COMMAND} -E copy_directory
		${CMAKE_CURRENT_SOURCE_DIR}/resources
		$<TARGET_FILE_DIR:Cake>/resources
	COMMENT "Copying Cake resources to build directory..."
)
	
if(MSVC)
	target_link_options(Cake PRIVATE $<$<CONFIG:Debug>:/DEBUG>)

	add_compile_options(/RTC1)

	
	set(USE_STATIC_MSVC_RUNTIME_LIBRARY OFF CACHE BOOL "Use Static MSVC++" FORCE)
	set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")

	set(KISSFFT_TEST OFF CACHE BOOL "Disable KissFFT tests" FORCE)

	set(KISSFFT_STATIC ON CACHE BOOL "Build static library" FORCE)

endif()

# set_target_properties(Cake PROPERTIES SUFFIX ".scr")




# ---------------------------------------------------------------
# GLSL -> SPIR-V shader compilation
# ---------------------------------------------------------------
find_program(GLSLC glslc HINTS
	$ENV{VULKAN_SDK}/Bin
	$ENV{VULKAN_SDK}/bin
)

if(GLSLC)
	message(STATUS "Found glslc: ${GLSLC}")

	set(SHADER_SRC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/resources/shaders)
	set(SHADER_OUT_DIR ${CMAKE_CURRENT_BINARY_DIR}/resources/shaders)
	file(MAKE_DIRECTORY ${SHADER_OUT_DIR})

	set(SHADER_SOURCES
		${SHADER_SRC_DIR}/VertexShader_vk.glsl
		${SHADER_SRC_DIR}/FragmentShader_vk.glsl
	)

	foreach(SHADER ${SHADER_SOURCES})
		get_filename_component(SHADER_NAME ${SHADER} NAME_WE)

		if(${SHADER_NAME} MATCHES ".*Fragment.*")
			set(SHADER_STAGE frag)
		else()
			set(SHADER_STAGE vert)
		endif()

		set(SPIRV_OUTPUT ${SHADER_OUT_DIR}/${SHADER_NAME}.spv)

		add_custom_command(
			OUTPUT ${SPIRV_OUTPUT}
			COMMAND ${GLSLC} -fshader-stage=${SHADER_STAGE} --target-env=vulkan1.0 -DFE_VULKAN ${SHADER} -o ${SPIRV_OUTPUT}
			DEPENDS ${SHADER}
			COMMENT "Compiling ${SHADER_NAME} -> ${SHADER_NAME}.spv"
			VERBATIM
		)

		list(APPEND SPIRV_OUTPUTS ${SPIRV_OUTPUT})
	endforeach()

	add_custom_target(SPIRV_ShadersCake ALL DEPENDS ${SPIRV_OUTPUTS})
	add_dependencies(Cake SPIRV_ShadersCake)
else()
	message(WARNING "glslc not found — shader compilation skipped. Set VULKAN_SDK environment variable.")
endif()



# Emscripten shi


include(${CMAKE_SOURCE_DIR}/EmscriptenConfig.cmake)
if(EMSCRIPTEN)
	setup_emscripten_target(Cake
		RESOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/resources
	)

	# Publish the runnable Foxcraft build to the FenixWeb Pages docs folder.
	# Release emscripten builds only; the custom Foxcraft.html is never touched.
	if(CMAKE_BUILD_TYPE STREQUAL "Release")
		set(CAKE_PAGES_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../FenixWeb/docs/Cake)
		set(CAKE_OUT_DIR $<TARGET_FILE_DIR:Cake>)

		add_custom_command(TARGET Cake POST_BUILD
			COMMAND ${CMAKE_COMMAND} -E copy
				${CAKE_OUT_DIR}/Cake.wasm ${CAKE_PAGES_DIR}/Cake.wasm
			COMMAND ${CMAKE_COMMAND} -E copy
				${CAKE_OUT_DIR}/Cake.js ${CAKE_PAGES_DIR}/Cake.js
			COMMAND ${CMAKE_COMMAND} -E copy
				${CAKE_OUT_DIR}/Cake.data ${CAKE_PAGES_DIR}/Cake.data
			COMMENT "Publishing CAKE build artifacts to ${CAKE_PAGES_DIR} (Release only)"
		)
	endif()
endif()

# Dear ImGui is a vendored third-party library. Its .cpp files define
# IMGUI_DEFINE_MATH_OPERATORS immediately before including imgui.h, which
# imgui_internal.h hard-errors on unless that define came first. A unity blob
# that pulls a game/engine file including imgui.h ahead of them breaks every
# ImGui file after it, so keep each ImGui file in its own translation unit.
set_source_files_properties(${IMGUI_SOURCE_FILES} PROPERTIES
	SKIP_UNITY_BUILD_INCLUSION ON
)