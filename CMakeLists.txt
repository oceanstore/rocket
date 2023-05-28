cmake_minimum_required(VERSION 3.12)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON) # For clang-tidy.
set(BUILD_SHARED_LIBS OFF) # We expect external libraries to be linked statically.
set(CMAKE_CXX_STANDARD 20) # Compile as C++17.
set(CMAKE_CXX_STANDARD_REQUIRED ON) # Require C++17 support.

project(rocket
        VERSION 2023.05
        DESCRIPTION "The oceanstore fast storage engine"
        LANGUAGES C CXX
)

message("${CMAKE_SOURCE_DIR}")



set(THIRD_PARTY_DIR "${CMAKE_SOURCE_DIR}/third_party")

set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_PATH}:${CMAKE_MODULE_PATH}")

LIST(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_PREFIX}")

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_SOURCE_DIR}/cmake/modules)

include(${CMAKE_SOURCE_DIR}/cmake/modules/BuildSPDK.cmake)

build_spdk()

# #####################################################################################################################
# COMPILER SETUP
# #####################################################################################################################

# Includes.
set(ROCKET_SRC_INCLUDE_DIR ${PROJECT_SOURCE_DIR}/src/include)
set(ROCKET_TEST_INCLUDE_DIR ${PROJECT_SOURCE_DIR}/test/include)
set(ROCKET_THIRD_PARTY_INCLUDE_DIR
        ${PROJECT_SOURCE_DIR}/third_party
        ${PROJECT_SOURCE_DIR}/third_party/spdk/include
)
include_directories(${ROCKET_SRC_INCLUDE_DIR} ${ROCKET_TEST_INCLUDE_DIR} ${ROCKET_THIRD_PARTY_INCLUDE_DIR})
include_directories(BEFORE src) # This is needed for gtest.

add_executable(rocket src/main/rocket.cpp)
target_link_libraries(rocket ${spdk_libs})
target_include_directories(
        rocket PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)
# message(${ROCKET_THIRD_PARTY_INCLUDE_DIR})
# message($<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)