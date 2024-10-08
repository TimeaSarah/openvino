# Copyright (C) 2018-2024 Intel Corporation
# SPDX-License-Identifier: Apache-2.0
#

file(GLOB SOURCES "src/*.cpp" "include/*.hpp")
file(GLOB_RECURSE NPUW_SOURCES "npuw/*.cpp" "npuw/*.hpp")

set(NPU_PLUGIN_TARGET "openvino_intel_npu_plugin")
set(NPU_PLUGIN_ENGINE_SOURCE_FILE "src/plugin.cpp")
set(TARGET_NAME ${NPU_PLUGIN_TARGET})

#
# Plugin library
#

ov_add_plugin(NAME ${NPU_PLUGIN_TARGET}
    DEVICE_NAME ${NPU_DEVICE_NAME}
    SOURCES ${SOURCES} ${NPUW_SOURCES}
    VERSION_DEFINES_FOR ${NPU_PLUGIN_ENGINE_SOURCE_FILE}
)

set_target_properties(${TARGET_NAME} PROPERTIES INTERPROCEDURAL_OPTIMIZATION_RELEASE ${ENABLE_LTO})
ov_add_clang_format_target(${TARGET_NAME}_clang FOR_TARGETS ${TARGET_NAME})

if(ENABLE_DRIVER_COMPILER_ADAPTER)
    target_compile_definitions(${TARGET_NAME} PRIVATE ENABLE_DRIVER_COMPILER_ADAPTER)
    target_link_libraries(${TARGET_NAME}
        PRIVATE
            openvino_npu_driver_compiler_adapter
    )
endif()

if(ENABLE_ZEROAPI_BACKEND)
    target_compile_definitions(${TARGET_NAME} PRIVATE ENABLE_ZEROAPI_BACKEND)
    target_link_libraries(${TARGET_NAME}
        PRIVATE
            openvino_npu_level_zero_backend
    )
endif()

if(ENABLE_IMD_BACKEND)
    target_compile_definitions(${TARGET_NAME} PRIVATE ENABLE_IMD_BACKEND)
endif()

if(NOT BUILD_SHARED_LIBS)
    target_compile_definitions(${TARGET_NAME} PRIVATE OPENVINO_STATIC_LIBRARY)
endif()

target_link_libraries(${TARGET_NAME}
    PRIVATE
        openvino::npu_al
        openvino::npu_logger_utils
        openvino::runtime
        openvino::runtime::dev
)

target_include_directories(${TARGET_NAME}
    PRIVATE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    PRIVATE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/>
)
target_include_directories(${TARGET_NAME}
    SYSTEM PRIVATE
        $<TARGET_PROPERTY:LevelZero::LevelZero,INTERFACE_INCLUDE_DIRECTORIES>
        $<TARGET_PROPERTY:LevelZero::NPUExt,INTERFACE_INCLUDE_DIRECTORIES>
)

if(ENABLE_AVX2)
    ov_avx2_optimization_flags(avx2_flags)
    target_compile_options(${TARGET_NAME} PRIVATE "${avx2_flags}")
endif()

ov_add_api_validator_post_build_step(TARGET ${NPU_PLUGIN_TARGET})
