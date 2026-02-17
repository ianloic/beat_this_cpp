cmake_minimum_required(VERSION 3.10)
project(BeatThisCpp)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Platform-specific compiler flags
if(WIN32)
    # Windows-specific settings
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
    if(MSVC)
        # Use UTF-8 encoding and disable some warnings
        add_compile_options(/utf-8 /wd4996)
        # Use static runtime
        # set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
    endif()
else()
    # Unix-like systems (macOS, Linux)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++20")
endif()

if(APPLE)
    # OBJCXX is enabled at the root level
    set_source_files_properties(Source/beat_this_api.cpp PROPERTIES LANGUAGE OBJCXX)
endif()



# --- ONNX Runtime --- #
# ONNX Runtime is now provided by the parent project via cmake/FetchONNXRuntime.cmake
if(NOT TARGET onnxruntime)
    message(FATAL_ERROR "onnxruntime target not found. Please ensure it is defined in the parent project.")
endif()

# --- Find Dependencies --- #
if(WIN32)
    # Windows: Use vcpkg find_package
    
else()
    # macOS and Linux: Use pkg-config
    find_package(PkgConfig REQUIRED)
endif()

if(CMAKE_SYSTEM_NAME MATCHES "iOS")
    set(BEAT_THIS_LIB_TYPE STATIC)
else()
    set(BEAT_THIS_LIB_TYPE SHARED)
endif()

add_library(beat_this_api ${BEAT_THIS_LIB_TYPE}
    Source/beat_this_api.cpp 
    Source/MelSpectrogram.cpp 
    Source/InferenceProcessor.cpp 
    Source/Postprocessor.cpp)

# Windows-specific settings for DLL export
if(WIN32)
    set_target_properties(beat_this_api PROPERTIES
        WINDOWS_EXPORT_ALL_SYMBOLS ON
        ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/$<CONFIG>
        LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/$<CONFIG>
        RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/$<CONFIG>
    )
endif()

# Check if all required dependencies were found
# find_package will throw an error if dependencies are not found

target_include_directories(beat_this_api PRIVATE
    ${ONNXRUNTIME_INCLUDE_DIRS}
    ${CMAKE_CURRENT_SOURCE_DIR}/Submodule/pocketfft # Add PocketFFT include directory
    ${CMAKE_CURRENT_SOURCE_DIR}/Submodule/miniaudio # Add miniaudio include directory
    ${CMAKE_CURRENT_SOURCE_DIR}/Source # For header files
)

target_link_libraries(beat_this_api PRIVATE onnxruntime)

if(APPLE)
    find_library(AVFOUNDATION_LIB AVFoundation)
    find_library(COREAUDIO_LIB CoreAudio)
    find_library(AUDIOTOOLBOX_LIB AudioToolbox)
    find_library(COREFOUNDATION_LIB CoreFoundation)
    
    target_link_libraries(beat_this_api PRIVATE
        ${AVFOUNDATION_LIB} 
        ${COREAUDIO_LIB} 
        ${AUDIOTOOLBOX_LIB} 
        ${COREFOUNDATION_LIB}
    )
endif()

if(WIN32)
    add_custom_command(
        TARGET beat_this_api POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${ONNXRUNTIME_LIBRARIES}/onnxruntime.dll"
        "$<TARGET_FILE_DIR:beat_this_api>"
        COMMENT "Copying onnxruntime.dll to API DLL directory"
    )
endif()

# Main executable with integrated beat analysis and audio generation
add_executable(beat_this_cpp Source/main.cpp)

target_include_directories(beat_this_cpp PRIVATE
    ${ONNXRUNTIME_INCLUDE_DIRS} # For main.cpp to include onnxruntime_cxx_api.h
    ${CMAKE_CURRENT_SOURCE_DIR}/Source # For beat_this_api.h
    ${CMAKE_CURRENT_SOURCE_DIR}/Submodule/miniaudio # Add miniaudio include directory
)

target_link_libraries(beat_this_cpp beat_this_api)

if(WIN32)
    add_custom_command(
        TARGET beat_this_cpp POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${ONNXRUNTIME_LIBRARIES}/onnxruntime.dll"
        "$<TARGET_FILE_DIR:beat_this_cpp>"
        COMMENT "Copying onnxruntime.dll to executable directory"
    )
endif()
