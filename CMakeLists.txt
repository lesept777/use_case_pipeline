cmake_minimum_required (VERSION 3.13)

project (pipeline_ecvl_eddl CXX)

include(CheckCXXCompilerFlag)

set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set_property(GLOBAL PROPERTY PREDEFINED_TARGETS_FOLDER "")
set(DOWNLOAD_MNIST ON CACHE BOOL "If flagged the mnist dataset will be automatically downloaded")

#--------------------------------------------------------------------------------------------------------
# BEGIN: Check for c++17 support
#--------------------------------------------------------------------------------------------------------
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang|AppleClang")
    check_cxx_compiler_flag("-std=c++17" COMPILER_SUPPORTS_CXX17)
elseif(MSVC)
    check_cxx_compiler_flag("/std:c++17" COMPILER_SUPPORTS_CXX17)
else()
    # Unrecognized compiler
    set(COMPILER_SUPPORTS_CXX17 ON)
endif()

if(NOT COMPILER_SUPPORTS_CXX17)
    message(FATAL_ERROR "The compiler ${CMAKE_CXX_COMPILER} has no C++17 support. Please use a different C++ compiler.")
endif()

set(CMAKE_CXX_EXTENSIONS OFF)
#--------------------------------------------------------------------------------------------------------
# END: Check for c++17 support
#--------------------------------------------------------------------------------------------------------

# SET BUILD TYPE (RELEASE/DEBUG)
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

if(MSVC)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
endif()

#--------------------------------------------------------------------------------------------------------
# BEGIN: static/dynamic runtime
#--------------------------------------------------------------------------------------------------------
set(pipeline_ecvl_eddl_STATIC ON CACHE BOOL "Whether to build static or dynamic libraries")

if(MSVC)
    if(pipeline_ecvl_eddl_STATIC)
        message(STATUS "Setting static runtime for MSVC")
        foreach(flag_var CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
            string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
        endforeach(flag_var)
    else()
        message(STATUS "Setting dynamic runtime for MSVC")
        foreach(flag_var CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
            string(REGEX REPLACE "/MT" "/MD" ${flag_var} "${${flag_var}}")
        endforeach(flag_var)
    endif()
endif()
#--------------------------------------------------------------------------------------------------------
# END: static/dynamic runtime
#--------------------------------------------------------------------------------------------------------

add_library(METRICS STATIC "")
add_library(MODELS STATIC "")
add_library(DATA_GENERATOR STATIC "")
add_executable(MNIST_BATCH)
add_executable(SKIN_LESION_CLASSIFICATION_TRAINING)
add_executable(SKIN_LESION_SEGMENTATION_TRAINING)
add_executable(SKIN_LESION_CLASSIFICATION_INFERENCE)
add_executable(SKIN_LESION_SEGMENTATION_INFERENCE)
add_executable(PNEUMOTHORAX_SEGMENTATION_TRAINING)
add_executable(PNEUMOTHORAX_SEGMENTATION_INFERENCE)
add_executable(COPY_GROUND_TRUTH_PNEUMOTHORAX)
add_subdirectory(cpp)
add_subdirectory("cpp/metrics")
add_subdirectory("cpp/models")
add_subdirectory("cpp/data_generator")

# Set SKIN_LESION_CLASSIFICATION_TRAINING as default
set_property(DIRECTORY PROPERTY VS_STARTUP_PROJECT SKIN_LESION_CLASSIFICATION_TRAINING)

set(ECVL_BUILD ON CACHE BOOL "Whether to build ECVL or not (requires ECVL)")

find_package(ecvl)
if(ecvl_FOUND)
    # Enable c++17 features
    target_compile_features(METRICS PUBLIC cxx_std_17)
    target_compile_features(COPY_GROUND_TRUTH_PNEUMOTHORAX PUBLIC cxx_std_17)

    target_link_libraries(METRICS ECVL_CORE ECVL_EDDL DATASET)
    target_link_libraries(DATA_GENERATOR METRICS)
    target_link_libraries(MODELS DATA_GENERATOR)
    target_link_libraries(MNIST_BATCH MODELS)
    target_link_libraries(SKIN_LESION_CLASSIFICATION_TRAINING MODELS)
    target_link_libraries(SKIN_LESION_SEGMENTATION_TRAINING MODELS)
    target_link_libraries(SKIN_LESION_CLASSIFICATION_INFERENCE MODELS)
    target_link_libraries(SKIN_LESION_SEGMENTATION_INFERENCE MODELS)
    target_link_libraries(PNEUMOTHORAX_SEGMENTATION_TRAINING MODELS)
    target_link_libraries(PNEUMOTHORAX_SEGMENTATION_INFERENCE MODELS)
    if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang|AppleClang")
        target_link_libraries(COPY_GROUND_TRUTH_PNEUMOTHORAX stdc++fs)
    endif()
    set_target_properties(METRICS PROPERTIES FOLDER "Utils")
    set_target_properties(MODELS PROPERTIES FOLDER "Utils")
    set_target_properties(DATA_GENERATOR PROPERTIES FOLDER "Utils")
    set_target_properties(COPY_GROUND_TRUTH_PNEUMOTHORAX PROPERTIES FOLDER "Utils")

    include_directories("${ecvl_DIR}/include")
    message(STATUS "ECVL found")
endif()

# Download and unpack MNIST dataset
if(DOWNLOAD_MNIST)
    if (IS_DIRECTORY "${CMAKE_BINARY_DIR}/mnist")
        message(STATUS "${CMAKE_BINARY_DIR}/mnist dataset folder already exists, download skipped!")
    else()
        message(STATUS "Downloading MNIST...")
        file(DOWNLOAD https://www.dropbox.com/s/8rslhjpects95k0/mnist.zip "${CMAKE_BINARY_DIR}/mnist.zip"
            STATUS status
            SHOW_PROGRESS
        )
        list(GET status 0 error_code)
        if(error_code)
            message(WARNING "CMake is not able to download the mnist dataset.\n
            Please, retry or download it manually (The dataset is also available at http://imagelab.ing.unimore.it/files/DeepHealth/mnist.zip).")
        else()
            message(STATUS "Downloading done.")
            if(EXISTS "${CMAKE_BINARY_DIR}/mnist.zip")
                message(STATUS "Unpacking (it may take a while)...")
                execute_process(COMMAND ${CMAKE_COMMAND} -E tar xvf "${CMAKE_BINARY_DIR}/mnist.zip" WORKING_DIRECTORY "${CMAKE_BINARY_DIR}" OUTPUT_QUIET)
                message(STATUS "Unpacking done.")
            endif()
        endif()
    endif()
endif()

#--------------------------------------------------------------------------------------------------------
# BEGIN: Summary
#--------------------------------------------------------------------------------------------------------
message(STATUS "--------------------------------------")
message(STATUS "Compiler: ${CMAKE_CXX_COMPILER_ID} ${CMAKE_CXX_COMPILER_VERSION}")
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")
message(STATUS "C++ flags: ${CMAKE_CXX_FLAGS}")
message(STATUS "C++ Release flags: ${CMAKE_CXX_FLAGS_RELEASE}")
message(STATUS "C++ Debug flags: ${CMAKE_CXX_FLAGS_DEBUG}")
#--------------------------------------------------------------------------------------------------------
# END: Summary
#--------------------------------------------------------------------------------------------------------