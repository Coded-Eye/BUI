include(FetchContent)

FetchContent_Declare(
    odin_c_bindgen
    GIT_REPOSITORY https://github.com/karl-zylinski/odin-c-bindgen.git
    GIT_TAG        main
)

FetchContent_MakeAvailable(odin_c_bindgen)

set(ODIN_BINDGEN_SRC_DIR "${odin_c_bindgen_SOURCE_DIR}")
set(ODIN_OUTPUT_DIR "${CMAKE_CURRENT_BINARY_DIR}")

if(WIN32)
    set(BINDGEN_EXE "${ODIN_BINDGEN_SRC_DIR}/bindgen.exe")
else()
    set(BINDGEN_EXE "${ODIN_BINDGEN_SRC_DIR}/bindgen.bin")
endif()

add_custom_command(
    OUTPUT "${BINDGEN_EXE}"
    COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_SOURCE_DIR}/bindings/odin/windows/libclang.dll" "${ODIN_BINDGEN_SRC_DIR}/libclang.dll"
    COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_SOURCE_DIR}/bindings/odin/windows/libclang.lib" "${ODIN_BINDGEN_SRC_DIR}/libclang/libclang.lib"
    COMMAND odin build ${ODIN_BINDGEN_SRC_DIR}/src -out:${BINDGEN_EXE}
    WORKING_DIRECTORY "${ODIN_BINDGEN_SRC_DIR}"
    COMMENT "Compiling automatically fetched odin-c-bindgen tool..."
    VERBATIM
)

file(MAKE_DIRECTORY "${ODIN_OUTPUT_DIR}")

add_custom_command(
    OUTPUT "${ODIN_OUTPUT_DIR}/BUI_kit.h"
    COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_SOURCE_DIR}/kit/include/BUI_kit.h" "${ODIN_OUTPUT_DIR}/BUI_kit.h"
    DEPENDS "${CMAKE_SOURCE_DIR}/kit/include/BUI_kit.h"
    COMMENT "Staging BUI_kit.h for binding generator..."
    VERBATIM
)

set(SJSON_FILE "${ODIN_OUTPUT_DIR}/bindgen.sjson")
set(SJSON_CONTENT
"package_name = \"bui_odin\"
inputs = [\"./BUI_kit.h\"]
output_folder = \"BUI\"
imports_file = \"libs.odin\""
)
file(WRITE "${SJSON_FILE}" "${SJSON_CONTENT}")

set(ODIN_LIB_FILE "${ODIN_OUTPUT_DIR}/libs.odin")
set(IMPORT_CONTENT
"when ODIN_OS == .Windows {
	@(extra_linker_flags=\"/NODEFAULTLIB:libcpmtd /NODEFAULTLIB:libcmtd /NODEFAULTLIB:libcpmt /NODEFAULTLIB:libcmt\")
	foreign import lib {
		\"./bui_kit.lib\",
		\"./bui_core.lib\",
		\"./bui_compat.lib\",
		\"./bui_api.lib\",
		\"./bui_gpu.lib\",

		\"system:msvcprtd.lib\", // Standard C++ Library (DLL - Debug)
		\"system:vcruntimed.lib\",// Visual C++ Runtime (DLL - Debug)
		\"system:ucrtd.lib\",    // Universal C Runtime (DLL - Debug)

		\"system:user32.lib\",     // Windowing, cursors, inputs, messages
		\"system:gdi32.lib\",      // Device contexts, fonts, basic graphics
		\"system:kernel32.lib\",   // Core OS services (threads, memory, file I/O)
		\"system:comctl32.lib\",   // Native extended UI controls and event tracking
		\"system:shell32.lib\",    // Shell API (folders, drag & drop)
		\"system:ole32.lib\",      // COM/OLE component initialization
		\"system:uuid.lib\",       // Globally Unique Identifiers definitions
		\"system:advapi32.lib\",   // Registry management and security APIs
		\"system:shlwapi.lib\",    // Shell Lightweight API (string processing, paths)
		\"system:winmm.lib\",      // Windows Multimedia API (high-resolution timers)

	}
} else when ODIN_OS == .Linux {
	@(extra_linker_flags=\"-Wl,--start-group -lstdc++ -Wl,--end-group\")
	foreign import lib {
		\"./libbui_kit.a\",
		\"./libbui_core.a\",
		\"./libbui_compat.a\",
		\"./libbui_api.a\",
		\"./libbui_gpu.a\",
	}
}"
)
file(WRITE "${ODIN_LIB_FILE}" "${IMPORT_CONTENT}")

set(FINAL_LIB_ODIN "${ODIN_OUTPUT_DIR}/BUI/BUI_kit.odin")

if(WIN32)
add_custom_command(
    OUTPUT "${FINAL_LIB_ODIN}"
    COMMAND ${BINDGEN_EXE} "."
    COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_BINARY_DIR}/gpu/Debug/bui_gpu.lib" "${ODIN_OUTPUT_DIR}/BUI/bui_gpu.lib"
    COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_BINARY_DIR}/compat/Debug/bui_compat.lib" "${ODIN_OUTPUT_DIR}/BUI/bui_compat.lib"
    COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_BINARY_DIR}/api/Debug/bui_api.lib" "${ODIN_OUTPUT_DIR}/BUI/bui_api.lib"
    COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_BINARY_DIR}/core/Debug/bui_core.lib" "${ODIN_OUTPUT_DIR}/BUI/bui_core.lib"
    COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_BINARY_DIR}/kit/Debug/bui_kit.lib" "${ODIN_OUTPUT_DIR}/BUI/bui_kit.lib"
    DEPENDS "${BINDGEN_EXE}" "${ODIN_OUTPUT_DIR}/BUI_kit.h" "${SJSON_FILE}" "${ODIN_LIB_FILE}"
    WORKING_DIRECTORY "${ODIN_OUTPUT_DIR}"
    COMMENT "Generating active bindings and collecting required library files into BUI package folder..."
    VERBATIM
)
else()
add_custom_command(
    OUTPUT "${FINAL_LIB_ODIN}"
    COMMAND ${BINDGEN_EXE} "."
    COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_BINARY_DIR}/gpu/libbui_gpu.a" "${ODIN_OUTPUT_DIR}/BUI/libbui_gpu.a"
    COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_BINARY_DIR}/compat/libbui_compat.a" "${ODIN_OUTPUT_DIR}/BUI/libbui_compat.a"
    COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_BINARY_DIR}/api/libbui_api.a" "${ODIN_OUTPUT_DIR}/BUI/libbui_api.a"
    COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_BINARY_DIR}/core/libbui_core.a" "${ODIN_OUTPUT_DIR}/BUI/libbui_core.a"
    COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_BINARY_DIR}/kit/libbui_kit.a" "${ODIN_OUTPUT_DIR}/BUI/libbui_kit.a"
    DEPENDS "${BINDGEN_EXE}" "${ODIN_OUTPUT_DIR}/BUI_kit.h" "${SJSON_FILE}" "${ODIN_LIB_FILE}"
    WORKING_DIRECTORY "${ODIN_OUTPUT_DIR}"
    COMMENT "Generating active bindings and collecting required library files into BUI package folder..."
    VERBATIM
)
endif()

add_custom_target(bui_odin_bindings ALL
    DEPENDS "${FINAL_LIB_ODIN}"
)

if(TARGET bui_kit)
    add_dependencies(bui_odin_bindings bui_kit)
endif()
if(TARGET bui_gpu)
    add_dependencies(bui_odin_bindings bui_gpu)
endif()
if(TARGET bui_core)
    add_dependencies(bui_odin_bindings bui_core)
endif()
if(TARGET bui_api)
    add_dependencies(bui_odin_bindings bui_api)
endif()
if(TARGET bui_compat)
    add_dependencies(bui_odin_bindings bui_compat)
endif()
