# Copyright(c) 2016 - 2019 Federico Bolelli, Costantino Grana, Michele Cancilla, Lorenzo Baraldi and Roberto Vezzani
# All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are met :
#
# *Redistributions of source code must retain the above copyright notice, this
# list of conditions and the following disclaimer.
#
# * Redistributions in binary form must reproduce the above copyright notice,
# this list of conditions and the following disclaimer in the documentation
# and / or other materials provided with the distribution.
#
# * Neither the name of YACCLAB nor the names of its
# contributors may be used to endorse or promote products derived from
# this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
# AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
# DISCLAIMED.IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
# FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
# DAMAGES(INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
# SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
# CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
# OR TORT(INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

cmake_minimum_required(VERSION 3.13.0)


# ----------------------------------------------------------------------------
#  Base variables
# ----------------------------------------------------------------------------

set (ProjectName "YACCLAB")
project (${ProjectName})

# ----------------------------------------------------------------------------
#  YACCLAB and CMAKE variables
# ----------------------------------------------------------------------------
set(CMAKE_CONFIGURATION_TYPES "Debug;Release" CACHE STRING "Configs" FORCE)
set(CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}" CACHE PATH "Install prefix" FORCE)

set(YACCLAB_ENABLE_CUDA OFF CACHE BOOL "Enable CUDA support if checked")
set(YACCLAB_ENABLE_3D OFF CACHE BOOL "Enable 3D support if checked")
set(YACCLAB_DOWNLOAD_DATASET OFF CACHE BOOL "If flagged the YACCLAB dataset will be automatically downloaded")
set(YACCLAB_DOWNLOAD_DATASET_3D OFF CACHE BOOL "If flagged the YACCLAB 3D dataset will be automatically downloaded")
set(YACCLAB_INPUT_DATASET_PATH "${CMAKE_INSTALL_PREFIX}/input" CACHE PATH "'input' dataset path") 
set(YACCLAB_OUTPUT_RESULTS_PATH "${CMAKE_INSTALL_PREFIX}/output" CACHE PATH "'output' results path") 
set(YACCLAB_OS "${CMAKE_SYSTEM_NAME} ${CMAKE_SYSTEM_VERSION}" CACHE INTERNAL STRING)
set(YACCLAB_IS_TRAVIS_CI OFF CACHE INTERNAL "This variable is used to handle travis-ci parameters, do not change its value")

if(MSVC)
  set(CMAKE_USE_RELATIVE_PATHS ON CACHE INTERNAL "" FORCE)
endif()


# ----------------------------------------------------------------------------
#  Check for CUDA support
# ----------------------------------------------------------------------------
if(YACCLAB_ENABLE_CUDA)
	include(CheckLanguage)
	check_language(CUDA)	
	if(CMAKE_CUDA_COMPILER)
		enable_language(CUDA)
		add_definitions(-DYACCLAB_WITH_CUDA)
	else()
		message(WARNING "No CUDA support")
	endif()
endif()


# ----------------------------------------------------------------------------
#  Check for c++14 support
# ----------------------------------------------------------------------------

if(DEFINED CMAKE_BUILD_TYPE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS ${CMAKE_CONFIGURATION_TYPES})
endif()

message(STATUS "Compiler ID: ${CMAKE_CXX_COMPILER_ID}")
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang|AppleClang" OR CMAKE_C_COMPILER_ID MATCHES "GNU|Clang|AppleClang")
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "" FORCE)
    if(NOT CMAKE_CXX_FLAGS MATCHES "-m32" AND NOT CMAKE_CXX_FLAGS MATCHES "-m64")
        set(CMAKE_CXX_FLAGS "-m32 ${CMAKE_CXX_FLAGS}" CACHE STRING "32-bit compiled" FORCE)
    endif()
    if(NOT CMAKE_C_FLAGS MATCHES "-m32" AND NOT CMAKE_C_FLAGS MATCHES "-m64")
	set(CMAKE_C_FLAGS "-m32 ${CMAKE_C_FLAGS}" CACHE STRING "32-bit compiled" FORCE)
    endif()
endif()

# The following commands would serve to simplify the check on the compiler version but do not yet seem to work perfectly
set(CMAKE_CXX_STANDARD_REQUIRED YES) # Force the following command to generate an error when it fails
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_EXTENSIONS NO)
   
#if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang|AppleClang")
#    include(CheckCXXCompilerFlag)
#    CHECK_CXX_COMPILER_FLAG("-std=c++14" COMPILER_SUPPORTS_CXX14)
#
#	if(COMPILER_SUPPORTS_CXX14 AND NOT CMAKE_CXX_FLAGS MATCHES "-std=c\\+\\+14")
#		set(CMAKE_CXX_FLAGS "-std=c++14 ${CMAKE_CXX_FLAGS}" CACHE STRING "Enable c++14" FORCE)
#	elseif(NOT COMPILER_SUPPORTS_CXX14)
#		message(FATAL_ERROR "The compiler ${CMAKE_CXX_COMPILER} has no C++14 support. Please use a different C++ compiler.")
#	endif()
#	
#	message(STATUS "C++ flags: ${CMAKE_CXX_FLAGS}")
#	message(STATUS "C flags: ${CMAKE_C_FLAGS}")
#elseif(MSVC)
#		if(MSVC_VERSION LESS 1900)
#			message(FATAL_ERROR "The compiler ${CMAKE_CXX_COMPILER} has no C++14 support. Please use a different C++ compiler.")
#		endif()
#else()
#	message(STATUS "We are not able to check if ${CMAKE_CXX_COMPILER} compiler has C++14 support. YACCLAB may not work properly.")
#endif()


# ----------------------------------------------------------------------------
#  Build & install layouts
# ----------------------------------------------------------------------------
set(YACCLAB_EXECUTABLE_OUTPUT_PATH "${CMAKE_INSTALL_PREFIX}" CACHE PATH "Output directory for applications")


# ----------------------------------------------------------------------------
# Solution folders
# ----------------------------------------------------------------------------

if(ENABLE_SOLUTION_FOLDERS)
  set_property(GLOBAL PROPERTY USE_FOLDERS ON)
  set_property(GLOBAL PROPERTY PREDEFINED_TARGETS_FOLDER "CMakeTargets")
endif()


# ----------------------------------------------------------------------------
# OpenCV 
# ----------------------------------------------------------------------------

set(OpenCV_REQUIRED_PACKAGES "core;imgcodecs;imgproc" CACHE STRING "OpenCV packages required by YACCLAB are already set")
string(FIND "${OpenCV_REQUIRED_PACKAGES}" "cudafeatures2d" cudafeatures2d_pos)
if (YACCLAB_ENABLE_CUDA AND CMAKE_CUDA_COMPILER AND ${cudafeatures2d_pos} EQUAL -1) 
	set(OpenCV_REQUIRED_PACKAGES "${OpenCV_REQUIRED_PACKAGES};cudafeatures2d" CACHE STRING "OpenCV packages required by YACCLAB are already set" FORCE)
endif()
FIND_PACKAGE(OpenCV REQUIRED ${OpenCV_REQUIRED_PACKAGES})
include_directories( ${OpenCV_INCLUDE_DIRS} )

if(MSVC)
	if("${OpenCV_STATIC}" STREQUAL "ON" OR "${OpenCV_SHARED}" STREQUAL "OFF")
		message(STATUS "Setting static runtime for MSVC")
		foreach(flag_var CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
			string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
		endforeach(flag_var)
		if (YACCLAB_ENABLE_CUDA AND CMAKE_CUDA_COMPILER)
			foreach(flag_var CMAKE_CUDA_FLAGS_DEBUG CMAKE_CUDA_FLAGS_RELEASE CMAKE_CUDA_FLAGS_MINSIZEREL CMAKE_CUDA_FLAGS_RELWITHDEBINFO) 
				string(REGEX REPLACE "-MD" "-MT" ${flag_var} "${${flag_var}}")
			endforeach(flag_var)
		endif()
	endif()
endif()

# ----------------------------------------------------------------------------
# Utility
# ----------------------------------------------------------------------------
message(STATUS "OS: ${YACCLAB_OS}, OpenCV: ${OpenCV_VERSION}")
add_definitions(-DOPENCV_VERSION_MAJOR=${OpenCV_VERSION_MAJOR})
add_definitions(-DYACCLAB_OS="${YACCLAB_OS}")

# ----------------------------------------------------------------------------
# Project
# ----------------------------------------------------------------------------

include_directories("${CMAKE_SOURCE_DIR}/include")
include_directories("${CMAKE_SOURCE_DIR}/cuda/include")
include_directories("${CMAKE_INSTALL_PREFIX}") # To avoid linking error when new file is added to the project directly from Visual Studio IDE

#SET_SOURCE_FILES_PROPERTIES("${CMAKE_SOURCE_DIR}/src/labeling_bolelli_2019.cc" PROPERTIES COMPILE_FLAGS "-O1")
#file(GLOB CUDAFILES ${CUDAFILES})
#if (YACCLAB_ENABLE_CUDA AND CMAKE_CUDA_COMPILER)
#	file(GLOB CUDAFILES ${CUDAFILES} "cuda/src/*.cu")
#	file(GLOB SRCFILES ${SRCFILES} "cuda/src/*.cc" "cuda/src/*.cpp" "cuda/src/*.c")
#	file(GLOB HFILES ${HFILES} "cuda/include/*.h" "cuda/include/*.hpp")
#endif()

add_definitions(-D_CRT_SECURE_NO_WARNINGS) #To suppress 'fopen' opencv warning/bug  


# ----------------------------------------------------------------------------
# Configuration file
# ----------------------------------------------------------------------------

file(READ "${CMAKE_SOURCE_DIR}/doc/config_2d.yaml" YACCLAB_CONFIG_2D)

if(YACCLAB_ENABLE_CUDA AND CMAKE_CUDA_COMPILER)
	file(READ "${CMAKE_SOURCE_DIR}/doc/config_cuda_2d.yaml" YACCLAB_CONFIG_CUDA_2D)
	if(YACCLAB_ENABLE_3D)
		file(READ "${CMAKE_SOURCE_DIR}/doc/config_cuda_3d.yaml" YACCLAB_CONFIG_CUDA_3D)
	endif()
endif()

if(YACCLAB_ENABLE_3D)
	file(READ "${CMAKE_SOURCE_DIR}/doc/config_3d.yaml" YACCLAB_CONFIG_3D)
endif()

if(YACCLAB_IS_TRAVIS_CI)	
	configure_file("${CMAKE_SOURCE_DIR}/doc/config_travis.yaml" "${CMAKE_INSTALL_PREFIX}/config.yaml")
else()
	if (NOT EXISTS "${CMAKE_INSTALL_PREFIX}/config.yaml")
		configure_file("${CMAKE_SOURCE_DIR}/doc/config_base.yaml" "${CMAKE_INSTALL_PREFIX}/config.yaml")
	endif()
endif()
#if (YACCLAB_ENABLE_CUDA AND CMAKE_CUDA_COMPILER)
#	source_group("Cuda Files" FILES ${CUDAFILES})
#endif()


if(YACCLAB_ENABLE_CUDA AND CMAKE_CUDA_COMPILER)
	cuda_add_executable(${ProjectName} "${CMAKE_INSTALL_PREFIX}/config.yaml")
else()
	add_executable(${ProjectName} "${CMAKE_INSTALL_PREFIX}/config.yaml")
endif()
set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT ${ProjectName}) # Set YACCLAB as startup group instead of ALL_BUILD
target_link_libraries (${ProjectName}  ${OpenCV_LIBS})

# ----------------------------------------------------------------------------
# Output and datasests
# ----------------------------------------------------------------------------

# Set YACCLAB as startup group instead of ALL_BUILD
#set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT YACCLAB)

#target_link_libraries (${ProjectName} ${OpenCV_LIBS})

add_subdirectory(src)
add_subdirectory(include)
if(YACCLAB_ENABLE_CUDA AND CMAKE_CUDA_COMPILER)
    add_subdirectory(cuda/src)
    add_subdirectory(cuda/include)
endif()

source_group("Include Files" REGULAR_EXPRESSION "inc$")
source_group("Header Files" REGULAR_EXPRESSION "h$|hpp$")
source_group("Source Files" REGULAR_EXPRESSION "cc$|cpp$|c$")
if(YACCLAB_ENABLE_CUDA AND CMAKE_CUDA_COMPILER)
    source_group("Cuda Files" REGULAR_EXPRESSION "cu$")
endif()
# Creation of the output directory
if (NOT EXISTS "${YACCLAB_OUTPUT_RESULTS_PATH}")
		file(MAKE_DIRECTORY ${YACCLAB_OUTPUT_RESULTS_PATH})
endif()

# Download and unpack dataset if necessary
if(YACCLAB_DOWNLOAD_DATASET)
    # Creation of input directory
    if (NOT EXISTS "${YACCLAB_INPUT_DATASET_PATH}")
        file(MAKE_DIRECTORY ${YACCLAB_INPUT_DATASET_PATH})
    endif()

    if (EXISTS "${YACCLAB_INPUT_DATASET_PATH}/dataset.zip")
        message(STATUS "${YACCLAB_INPUT_DATASET_PATH}/dataset.zip already exists, download skipped!")
    else()
        message(STATUS "Downloading dataset...")
        file(DOWNLOAD http://imagelab.ing.unimore.it/files/YACCLAB_dataset_new.zip "${YACCLAB_INPUT_DATASET_PATH}/dataset.zip" STATUS status SHOW_PROGRESS)
        list( GET status 0 error_code )
        if( error_code )
            message(ERROR " Cmake is not able to download the YACCLAB dataset. \nPlease, retry or download it manually and put all unzipped dataset directories in the 'input' path:\n'" ${YACCLAB_INPUT_DATASET_PATH} "'. \n(The dataset is available at http://imagelab.ing.unimore.it/files/YACCLAB_dataset_new.zip).")
            if(EXISTS "${YACCLAB_INPUT_DATASET_PATH}/dataset.zip")
                file(REMOVE "${YACCLAB_INPUT_DATASET_PATH}/dataset.zip")
            endif()
        else()
            message(STATUS "Downloading done.")
        endif()

        if(EXISTS "${YACCLAB_INPUT_DATASET_PATH}/dataset.zip")
            message(STATUS "Unpacking (it may take a while)...")
            execute_process(COMMAND ${CMAKE_COMMAND} -E tar xvf "${YACCLAB_INPUT_DATASET_PATH}/dataset.zip" WORKING_DIRECTORY "${YACCLAB_INPUT_DATASET_PATH}" OUTPUT_QUIET)
            message(STATUS "Unpacking done.")
        endif()
    endif()

endif()

if (YACCLAB_DOWNLOAD_DATASET_3D)
	# Creation of input directory
	if (NOT EXISTS "${YACCLAB_INPUT_DATASET_PATH}")
		file(MAKE_DIRECTORY ${YACCLAB_INPUT_DATASET_PATH})
	endif()

	if (EXISTS "${YACCLAB_INPUT_DATASET_PATH}/dataset3D.zip")
		message(STATUS "${YACCLAB_INPUT_DATASET_PATH}/dataset3D.zip already exists, download skipped!")
	else()
		message(STATUS "Downloading 3D dataset...")
		file(DOWNLOAD http://imagelab.ing.unimore.it/files/YACCLAB_dataset3D.zip "${YACCLAB_INPUT_DATASET_PATH}/dataset3D.zip" STATUS status SHOW_PROGRESS)
		list( GET status 0 error_code )
		if( error_code )
			message(ERROR " Cmake is not able to download the YACCLAB 3D dataset. \nPlease, retry or download it manually and put all unzipped dataset directories in the 'input' path:\n'" ${YACCLAB_INPUT_DATASET_PATH} "'. \n(The dataset is available at http://imagelab.ing.unimore.it/files/YACCLAB_dataset3D.zip).")
			if(EXISTS "${YACCLAB_INPUT_DATASET_PATH}/dataset3D.zip")
				file(REMOVE "${YACCLAB_INPUT_DATASET_PATH}/dataset3D.zip")
			endif()
		else()
			message(STATUS "Downloading done.")
		endif()
		
		if(EXISTS "${YACCLAB_INPUT_DATASET_PATH}/dataset3D.zip")
			message(STATUS "Unpacking (it may take a while)...")
			execute_process(COMMAND ${CMAKE_COMMAND} -E tar xvf "${YACCLAB_INPUT_DATASET_PATH}/dataset3D.zip" WORKING_DIRECTORY "${YACCLAB_INPUT_DATASET_PATH}" OUTPUT_QUIET)
			message(STATUS "Unpacking done.")
		endif()
	endif()
endif()
