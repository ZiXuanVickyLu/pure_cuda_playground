cmake_minimum_required(VERSION 3.26 FATAL_ERROR)
project(CUDAPlayGround C CXX CUDA)

# set C++17 standard supported, the cutlass library requires C++17
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
if(MSVC)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /Zc:__cplusplus")
	# for passing correct __cplusplus macro to the compiler
endif()

# set at least CUDA 11.0 standard supported, this will overwriting CUDA compiler flags
# the cutlass library requires C++17 standard
set(CMAKE_CUDA_STANDARD 17)

#update the CUDA_ARCHITECTURES with cmake script to determine， Auto-detect GPU architecture, sets ${CUDA_ARCHS}
include("cmake/AutoDetectCudaArch.cmake")

#add default constexpr support for device-enabled constructor and CUDA lambda.
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} --extended-lambda --expt-relaxed-constexpr")
#update cmake module path
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/"  ${CMAKE_MODULE_PATH} CACHE PATH "cache module path")

# since cutlass has long path for its object files, we need to set this to a larger value to avoid warning or potential error
set(CMAKE_OBJECT_PATH_MAX 500)

# define Pathfrom
if(LINUX)
	add_compile_definitions(CP_PLATFORM_LINUX)
	message(STATUS "STATUS: Platform Linux")
elseif(WIN32 OR MSVC)
	add_compile_definitions(CP_PLATFORM_WINDOWS)
	message(STATUS "STATUS: Platform Windows")
endif()

#set build directory structure
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib/${CMAKE_BUILD_TYPE}" CACHE PATH "" FORCE)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib/${CMAKE_BUILD_TYPE}" CACHE PATH "" FORCE)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin/${CMAKE_BUILD_TYPE}" CACHE PATH "" FORCE)
set(THIRD_PARTY_DIR "${CMAKE_CURRENT_SOURCE_DIR}/ext")

# add cuda
find_package(CUDA REQUIRED)
if(CUDA_FOUND)
    include_directories(${CUDA_INCLUDE_DIRS})
    link_directories(${CUDA_TOOLKIT_ROOT_DIR}/lib64)
    link_libraries(${CUDA_LIBRARIES})
else()
    message("ERROR: Cuda Not Found! [Required]")
endif()

#add complier command for e.g.,vscode-based ide and GNU. Clion and VS is not necessary
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# add thrust (and with cub)
find_package(Thrust REQUIRED CONFIG)
thrust_create_target(Thrust)

#================================================================== build submodules ====================================================================
# nvpro_core configuration
# look for nvpro_core 1) as a sub-folder 2) at some other locations
# this cannot be put anywhere else since we still didn't find setup.cmake yet
if(NOT BASE_DIRECTORY)
	find_path(BASE_DIRECTORY
			NAMES nvpro_core/cmake/setup.cmake
			PATHS ${THIRD_PARTY_DIR}
			REQUIRED
			DOC "Directory containing nvpro_core"
	)
endif()
## Various functions and macros REQUIRED
if(EXISTS ${BASE_DIRECTORY}/nvpro_core/cmake/setup.cmake)
	include(${BASE_DIRECTORY}/nvpro_core/cmake/setup.cmake)
	include(${BASE_DIRECTORY}/nvpro_core/cmake/utilities.cmake)
else()
	message(FATAL_ERROR "could not find base directory, please set BASE_DIRECTORY to folder containing nvpro_core")
endif()

#Eigen configuration
set(BUILD_TESTING OFF CACHE BOOL "disable building eigen tests" FORCE)

#cereal configuration
#disable performance comparison between boost, which required boost library otherwise
set(BUILD_SANDBOX OFF CACHE BOOL "disable building sandbox of cereal" FORCE)
set(SKIP_PERFORMANCE_COMPARISON ON CACHE BOOL "disable performance comparison between boost" FORCE)

#add submodules
include_directories("${THIRD_PARTY_DIR}/trove")		# this is an experimental repo for GPU vectorization data-structure
add_subdirectory("${THIRD_PARTY_DIR}/eigen")		# this is for basic linear algebraic routine, branch v3.4.0
add_subdirectory("${THIRD_PARTY_DIR}/spdlog")		# this is for spdlog
add_subdirectory("${THIRD_PARTY_DIR}/fmt")		# this is for formating
add_subdirectory("${THIRD_PARTY_DIR}/nvpro_core") 	# this is for basic graphical visualization
add_subdirectory("${THIRD_PARTY_DIR}/cereal") 		# this is for serialization
add_subdirectory("${THIRD_PARTY_DIR}/cutlass")		# this is an nvidia gemm and more template library 
add_subdirectory("${THIRD_PARTY_DIR}/googletest")	# this is for unit test
#================================================================== build submodules done ===============================================================

#option library
option(CP_LIBRARY_CORE				"Enable building the core library"	ON)

#option compile
option(CP_ENABLE_ASSERTS_OPT			"Enable assertion"			ON)

#precompile macro
if(CMAKE_BUILD_TYPE MATCHES "Debug")
	add_compile_definitions(CP_DEBUG_MODE)
	message(STATUS
	"STATUS: Enable Debug Mode")

	if(CP_ENABLE_ASSERTS_OPT)
		add_compile_definitions(CP_ENABLE_ASSERTS)
		message(STATUS
		"STATUS: Enable CP_ENABLE_ASSERTS")
	endif()
endif()

#add asset path
set(CP_ASSET_PATH "${CMAKE_CURRENT_SOURCE_DIR}/asset/")

message(STATUS
	"STATUS: Set Asset to " ${CP_ASSET_PATH})
add_compile_definitions(CP_ASSET_PATH="${CP_ASSET_PATH}")

# add src
set(CP_INCLUDE "${CMAKE_CURRENT_SOURCE_DIR}/include")
set(CP_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/src")
include_directories("${CP_INCLUDE}")
include_directories("${CP_SOURCES}")
add_subdirectory(src)

option(CP_TEST			"Enable building tests" ON)
#add test?
if(CP_TEST)
    add_subdirectory(test)
endif()

include(printConfig)