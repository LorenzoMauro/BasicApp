#${PROJECT_NAME} CXX files
file(GLOB_RECURSE SOURCES "*.cpp")
file(GLOB_RECURSE HEADERS "*.h")
file(GLOB_RECURSE CUDA_FILES "*.cu")
file(GLOB_RECURSE CUDA_OPTIXIR_FILES "${CMAKE_CURRENT_SOURCE_DIR}/OptixIr/*.cu")
file(GLOB_RECURSE CUDA_PTX_FILES "${CMAKE_CURRENT_SOURCE_DIR}/Ptx/*.cu")

list(REMOVE_ITEM CUDA_FILES ${CUDA_OPTIXIR_FILES})
list(REMOVE_ITEM CUDA_FILES ${CUDA_PTX_FILES})

#Organize Files into Folder Like Filters (Useful when creating Visual Studio Solution)
assign_source_group(
    ${SOURCES}
    ${HEADERS}
    ${CUDA_COMPILE_FILES}
    ${PTX_COMPILE_FILES}
    ${OPTIXIR_COMPILE_FILES}
    ${OPTIXIR_MODULES}
    ${PTX_MODULES}
 )
 
 # Include Directories
list(APPEND INCLUDE_DIRS
    ${CMAKE_SOURCE_DIR}/${PROJECT_NAME}/src
    ${CMAKE_SOURCE_DIR}/assets/fonts
    ${CMAKE_CUDA_TOOLKIT_INCLUDE_DIRECTORIES}
    ${OPTIX_INCLUDE_DIR}
    ${Stb_INCLUDE_DIR}
)

# Cuda Compile Options
set(CUDA_COMPILE_OPTIONS
    --use_fast_math
    --relocatable-device-code=true
    -gencode=arch=compute_70,code=\"sm_70,compute_70\"
    --use-local-env
    -x cu
    --maxrregcount=0
    --machine 64
    --compile
    -cudart static
    --extended-lambda
    $<$<AND:$<CONFIG:Debug>>:-G>
    $<$<OR:$<CONFIG:Release>,$<CONFIG:RelWithDebInfo>,$<CONFIG:MinSizeRel>>:-O3>
)

# Create object library for OptiX files
# we want to create this target only if there are files to compile
if(CUDA_OPTIXIR_FILES)
    set(OPTIXIR_TARGET "OptixIR_Compilation")
    add_library(${OPTIXIR_TARGET} OBJECT ${CUDA_OPTIXIR_FILES})
    set_property(TARGET ${OPTIXIR_TARGET} PROPERTY CUDA_OPTIX_COMPILATION ON)
    target_include_directories(${OPTIXIR_TARGET} PRIVATE ${INCLUDE_DIRS})
    target_compile_options(${OPTIXIR_TARGET} PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:${CUDA_COMPILE_OPTIONS}>)
    set_target_properties(${OPTIXIR_TARGET} PROPERTIES
	VS_DEBUGGER_WORKING_DIRECTORY "$<TARGET_FILE_DIR:${PROJECT_NAME}>"
	CUDA_ARCHITECTURES OFF
	CUDA_SEPARABLE_COMPILATION ON
        POSITION_INDEPENDENT_CODE ON
	CUDA_VERBOSE_BUILD ON
    )
    target_compile_definitions(${OPTIXIR_TARGET} PRIVATE $<$<CONFIG:Debug>:DEBUG>)
    #ensure the files are copied even if the main target is not built
    add_custom_target(${OPTIXIR_TARGET}_copyFiles ALL
	COMMAND ${CMAKE_COMMAND} -E copy_directory
        "$<TARGET_FILE_DIR:${PROJECT_NAME}>/CMakeFiles/${OPTIXIR_TARGET}.dir/OptixIr/"
	"$<TARGET_FILE_DIR:${PROJECT_NAME}>/optixir/"
        COMMENT "Copying OptiX IR files to destination directory"
	DEPENDS ${OPTIXIR_TARGET}
    )
endif()

# Create object library for PTX files
if(CUDA_PTX_FILES)
    set(PTX_TARGET "PTX_Compilation")
    add_library(${PTX_TARGET} OBJECT ${CUDA_PTX_FILES})
    set_property(TARGET ${PTX_TARGET} PROPERTY CUDA_PTX_COMPILATION ON)
    target_include_directories(${PTX_TARGET} PRIVATE ${INCLUDE_DIRS})
    target_compile_options(${PTX_TARGET} PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:${CUDA_COMPILE_OPTIONS}>)
    set_target_properties(${PTX_TARGET} PROPERTIES
    	VS_DEBUGGER_WORKING_DIRECTORY "$<TARGET_FILE_DIR:${PROJECT_NAME}>"
	CUDA_ARCHITECTURES OFF
	CUDA_SEPARABLE_COMPILATION ON
	POSITION_INDEPENDENT_CODE ON
	CUDA_VERBOSE_BUILD ON
    )
    target_compile_definitions(${PTX_TARGET} PRIVATE $<$<CONFIG:Debug>:DEBUG>)
    add_custom_target(${PTX_TARGET}_copyFiles ALL
	COMMAND ${CMAKE_COMMAND} -E copy_directory
	"$<TARGET_FILE_DIR:${PROJECT_NAME}>/CMakeFiles/${PTX_TARGET}.dir/Ptx/"
	"$<TARGET_FILE_DIR:${PROJECT_NAME}>/ptx/"
	COMMENT "Copying PTX files to destination directory"
	DEPENDS ${PTX_TARGET}
    )
endif()


# Main Executable Target
add_executable(${PROJECT_NAME} main.cpp)

target_sources(${PROJECT_NAME} PRIVATE  ${SOURCES} ${HEADERS} ${CUDA_FILES} )

if(CUDA_OPTIXIR_FILES)
    add_dependencies(${PROJECT_NAME} ${OPTIXIR_TARGET})
    add_dependencies(${PROJECT_NAME} ${OPTIXIR_TARGET}_copyFiles)
endif()
if(CUDA_PTX_FILES)
	add_dependencies(${PROJECT_NAME} ${PTX_TARGET})
	add_dependencies(${PROJECT_NAME} ${PTX_TARGET}_copyFiles)
endif()

target_include_directories(${PROJECT_NAME} PRIVATE ${INCLUDE_DIRS})

target_compile_options(${PROJECT_NAME} PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:${CUDA_COMPILE_OPTIONS}>)

target_compile_definitions(${PROJECT_NAME} PRIVATE $<$<CONFIG:Debug>:DEBUG>)

set_target_properties(${PROJECT_NAME} PROPERTIES
    VS_DEBUGGER_WORKING_DIRECTORY "$<TARGET_FILE_DIR:${PROJECT_NAME}>"
	#CUDA_ARCHITECTURES OFF
	CUDA_SEPARABLE_COMPILATION ON
    POSITION_INDEPENDENT_CODE ON
	CUDA_VERBOSE_BUILD ON
)

target_link_libraries(${PROJECT_NAME} PRIVATE 
  imgui::imgui
  spdlog::spdlog_header_only
  glfw
  glad::glad
  CUDA::cudart_static
  CUDA::cuda_driver
  CUDA::nvtx3
  CUDA::cufft
  CUDA::curand
  CUDA::cublas
  CUDA::cublasLt
)

# Post Build Commands
#add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
#                   COMMAND xcopy /D /Y /F "\"${CMAKE_SOURCE_DIR}/assets/ImGui.ini\"" "\"$<TARGET_FILE_DIR:${PROJECT_NAME}>/assets/\\\""
#                   COMMENT "Copying ImGui.ini from repo assets if newer..."
#)
#add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
#                   COMMAND if exist "\"$<TARGET_FILE_DIR:${PROJECT_NAME}>/assets/ImGui.ini\"" (xcopy /D /Y /F "\"$<TARGET_FILE_DIR:${PROJECT_NAME}>/assets/ImGui.ini\"" "\"${CMAKE_SOURCE_DIR}/assets/\\\"")
#                   COMMENT "Copying ImGui.ini from target folder to repo assets if newer..."
#)
#
#add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
#                   COMMAND ${CMAKE_COMMAND} -E copy_directory "\"${CMAKE_SOURCE_DIR}/assets/textures\"" "\"$<TARGET_FILE_DIR:${PROJECT_NAME}>/assets/textures\""
#                   COMMENT "Copying textures from repo assets..."
#)
#
#add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
#                   COMMAND ${CMAKE_COMMAND} -E copy_directory "\"${CMAKE_SOURCE_DIR}/assets/models\"" "\"$<TARGET_FILE_DIR:${PROJECT_NAME}>/assets/models\""
#                   COMMENT "Copying models from repo assets..."
#)
#
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
				   COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_BINARY_DIR}/../compile_commands.json ${PROJECT_SOURCE_DIR}
                   COMMENT "Copying compile_commands.json to root for non-msvc text editors..."
)


#install Target                   
set(CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}/install" CACHE PATH "Default install path" FORCE)

install(DIRECTORY "$<TARGET_FILE_DIR:${PROJECT_NAME}>/ptx" DESTINATION .)

install(DIRECTORY "$<TARGET_FILE_DIR:${PROJECT_NAME}>/optixir" DESTINATION .)

install(DIRECTORY "$<TARGET_FILE_DIR:${PROJECT_NAME}>/assets" DESTINATION .)

install(TARGETS ${PROJECT_NAME} RUNTIME DESTINATION .)
