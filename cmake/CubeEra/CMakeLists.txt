function(add_bgfx_shader FILE FOLDER)
	get_filename_component(FILENAME "${FILE}" NAME_WE)
	string(SUBSTRING "${FILENAME}" 0 2 TYPE)
	if("${TYPE}" STREQUAL "fs")
		set(TYPE "FRAGMENT")
	elseif("${TYPE}" STREQUAL "vs")
		set(TYPE "VERTEX")
	elseif("${TYPE}" STREQUAL "cs")
		set(TYPE "COMPUTE")
	else()
		set(TYPE "")
	endif()

	if(NOT "${TYPE}" STREQUAL "")
		set(COMMON FILE ${FILE} ${TYPE} INCLUDES shader)
		set(OUTPUTS "")
		set(OUTPUTS_PRETTY "")

		if(WIN32)
			# dx11
			set(DX11_OUTPUT ${BGFX_DIR}/examples/runtime/shaders/dx11/${FILENAME}.bin)
			if(NOT "${TYPE}" STREQUAL "COMPUTE")
				_bgfx_shaderc_parse(
					DX11 ${COMMON} WINDOWS
					PROFILE s_5_0
					O 3
					OUTPUT ${DX11_OUTPUT}
				)
			else()
				_bgfx_shaderc_parse(
					DX11 ${COMMON} WINDOWS
					PROFILE s_5_0
					O 1
					OUTPUT ${DX11_OUTPUT}
				)
			endif()
			list(APPEND OUTPUTS "DX11")
			set(OUTPUTS_PRETTY "${OUTPUTS_PRETTY}DX11, ")
		endif()

		if(APPLE)
			# metal
			set(METAL_OUTPUT ${BGFX_DIR}/examples/runtime/shaders/metal/${FILENAME}.bin)
			_bgfx_shaderc_parse(METAL ${COMMON} OSX PROFILE metal OUTPUT ${METAL_OUTPUT})
			list(APPEND OUTPUTS "METAL")
			set(OUTPUTS_PRETTY "${OUTPUTS_PRETTY}Metal, ")
		endif()

		# essl
		if(NOT "${TYPE}" STREQUAL "COMPUTE")
			set(ESSL_OUTPUT ${BGFX_DIR}/examples/runtime/shaders/essl/${FILENAME}.bin)
			_bgfx_shaderc_parse(ESSL ${COMMON} ANDROID PROFILE 100_es OUTPUT ${ESSL_OUTPUT})
			list(APPEND OUTPUTS "ESSL")
			set(OUTPUTS_PRETTY "${OUTPUTS_PRETTY}ESSL, ")
		endif()

		# glsl
		set(GLSL_OUTPUT ${BGFX_DIR}/examples/runtime/shaders/glsl/${FILENAME}.bin)
		if(NOT "${TYPE}" STREQUAL "COMPUTE")
			_bgfx_shaderc_parse(GLSL ${COMMON} LINUX PROFILE 140 OUTPUT ${GLSL_OUTPUT})
		else()
			_bgfx_shaderc_parse(GLSL ${COMMON} LINUX PROFILE 430 OUTPUT ${GLSL_OUTPUT})
		endif()
		list(APPEND OUTPUTS "GLSL")
		set(OUTPUTS_PRETTY "${OUTPUTS_PRETTY}GLSL, ")

		# spirv
		if(NOT "${TYPE}" STREQUAL "COMPUTE")
			set(SPIRV_OUTPUT ${BGFX_DIR}/examples/runtime/shaders/spirv/${FILENAME}.bin)
			_bgfx_shaderc_parse(SPIRV ${COMMON} LINUX PROFILE spirv OUTPUT ${SPIRV_OUTPUT})
			list(APPEND OUTPUTS "SPIRV")
			set(OUTPUTS_PRETTY "${OUTPUTS_PRETTY}SPIRV")
			set(OUTPUT_FILES "")
			set(COMMANDS "")
		endif()

		foreach(OUT ${OUTPUTS})
			list(APPEND OUTPUT_FILES ${${OUT}_OUTPUT})
			list(APPEND COMMANDS COMMAND "bgfx::shaderc" ${${OUT}})
			get_filename_component(OUT_DIR ${${OUT}_OUTPUT} DIRECTORY)
			file(MAKE_DIRECTORY ${OUT_DIR})
		endforeach()

		file(RELATIVE_PATH PRINT_NAME ${BGFX_DIR}/examples ${FILE})
		add_custom_command(
			MAIN_DEPENDENCY ${FILE} OUTPUT ${OUTPUT_FILES} ${COMMANDS}
			COMMENT "Compiling shader ${PRINT_NAME} for ${OUTPUTS_PRETTY}"
		)
	endif()
endfunction()


function(add_source ARG_NAME)
	# Parse arguments
	cmake_parse_arguments(ARG "COMMON" "" "DIRECTORIES;SOURCES" ${ARGN})

	# Get all source files
	list(APPEND ARG_DIRECTORIES "${PROJECT_SOURCE_DIR}/src")
	set(SOURCES "")
	set(SHADERS "")
	foreach(DIR ${ARG_DIRECTORIES})
		if(APPLE)
			file(GLOB GLOB_SOURCES ${DIR}/*.mm)
			list(APPEND SOURCES ${GLOB_SOURCES})
		endif()
		file(GLOB GLOB_SOURCES ${DIR}/*.c ${DIR}/*.cpp ${DIR}/*.h ${DIR}/*.sc)
		list(APPEND SOURCES ${GLOB_SOURCES})
		file(GLOB GLOB_SHADERS ${DIR}/*.sc)
		list(APPEND SHADERS ${GLOB_SHADERS})
	endforeach()

	# Add target
	if(ANDROID)
		add_library(${ARG_NAME} SHARED ${SOURCES})
	else()
		add_executable(${ARG_NAME} WIN32 ${SOURCES})
	endif()
	if(NOT BGFX_INSTALL_EXAMPLES)
		set_property(TARGET ${ARG_NAME} PROPERTY EXCLUDE_FROM_ALL ON)
	endif()
	target_link_libraries(${ARG_NAME} PUBLIC example-common)
	configure_debugging(${ARG_NAME} WORKING_DIR runtime)
	if(MSVC)
		set_target_properties(${ARG_NAME} PROPERTIES LINK_FLAGS "/ENTRY:\"mainCRTStartup\"")
	endif()
	if(BGFX_CUSTOM_TARGETS)
		add_dependencies(examples ${ARG_NAME})
	endif()
	if(IOS)
		set_target_properties(
			${ARG_NAME}
			PROPERTIES MACOSX_BUNDLE ON
					   MACOSX_BUNDLE_GUI_IDENTIFIER ${ARG_NAME}
					   MACOSX_BUNDLE_BUNDLE_VERSION 0
					   MACOSX_BUNDLE_SHORT_VERSION_STRING 0
					   XCODE_ATTRIBUTE_CODE_SIGN_IDENTITY "iPhone Developer"
		)
	endif()
	target_compile_definitions(
		${ARG_NAME}
		PRIVATE "-D_CRT_SECURE_NO_WARNINGS" #
				"-D__STDC_FORMAT_MACROS" #
				"-DENTRY_CONFIG_IMPLEMENT_MAIN=1" #
	)

	# Configure shaders
	if(NOT ARG_COMMON
	   AND NOT IOS
	   AND NOT EMSCRIPTEN
	   AND NOT ANDROID
	)
		foreach(SHADER ${SHADERS})
			add_bgfx_shader(${SHADER} ${ARG_NAME})
		endforeach()
		source_group("Shader Files" FILES ${SHADERS})
	endif()

	if(NOT ARG_COMMON AND EMSCRIPTEN)
		set_target_properties(
			example-${ARG_NAME}
			PROPERTIES LINK_FLAGS
					   "-s PRECISE_F32=1 -s TOTAL_MEMORY=268435456 -s ENVIRONMENT=web --memory-init-file 1 --emrun"
					   SUFFIX ".html"
		)
	endif()

	# Directory name
	set_target_properties(${ARG_NAME} PROPERTIES FOLDER "CubeEra")
endfunction()

add_source(
	CubeEra
)