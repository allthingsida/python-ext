cmake_minimum_required(VERSION 3.27)
project(python_ext)

# Specify the C++ standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# Include IDA SDK bootstrap
include($ENV{IDASDK}/ida-cmake/bootstrap.cmake)
find_package(idasdk REQUIRED)

# Find the Python interpreter and development components
find_package(Python COMPONENTS Interpreter Development REQUIRED) #find_package(Python 3.11 COMPONENTS ...)

# Check if Python_LIBRARIES is set and valid
if(NOT Python_LIBRARIES)
    message(FATAL_ERROR "Python libraries not found.")
endif()

# On Windows, link against python3.lib
# (count on IDAPython/idapyswitch to pick the right Python version)
if (WIN32)
    # Link to Python.lib (it will forward to the proper Python3x.lib/.dll)
    string(REGEX REPLACE "python3([0-9]+)\\.lib" "python3.lib" PYTHON_LIB_DIR "${Python_LIBRARIES}")
else()
    set(PYTHON_LIB_DIR "${Python_LIBRARIES}")
endif()

# Add pybind11 subdirectory
add_subdirectory(3rd_party/pybind11 ${CMAKE_CURRENT_BINARY_DIR}/pybind11_build)

# Add plugin using ida-cmake convenience function
ida_add_plugin(python_ext
    SOURCES
        plugin.cpp
        extension.cpp
        extension.h
        idasdk.h
    LIBRARIES
        pybind11::embed
        ${PYTHON_LIB_DIR}
    INCLUDES
        ${Python_INCLUDE_DIRS}
)
