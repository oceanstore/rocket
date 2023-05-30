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

set(BUILD_SUPPORT_DIR "${CMAKE_SOURCE_DIR}/build_support")
set(THIRD_PARTY_DIR "${CMAKE_SOURCE_DIR}/third_party")

set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_PATH}:${CMAKE_MODULE_PATH}")
LIST(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_PREFIX}")
set(CMAKE_MODULE_PATH  ${BUILD_SUPPORT_DIR}/cmake/modules)
include(${CMAKE_MODULE_PATH}/BuildSPDK.cmake)
build_spdk()

# #####################################################################################################################
# ANALYSIS SETUP
# #####################################################################################################################

# clang-dity
find_program(CLANG_TIDY_BIN
        NAMES clang-tidy clang-tidy-14
        HINTS ${BUSTUB_CLANG_SEARCH_PATH})

if("${CLANG_TIDY_BIN}" STREQUAL "CLANG_TIDY_BIN-NOTFOUND")
        message(WARNING "couldn't find clang-tidy.")
else()
        # Output compile_commands.json
        set(CMAKE_EXPORT_COMPILE_COMMANDS 1)
        message(STATUS "found clang-tidy at ${CLANG_TIDY_BIN}")
endif()

find_program(CLANG_APPLY_REPLACEMENTS_BIN
        NAMES clang-apply-replacements clang-apply-replacements-14
        HINTS ${BUSTUB_CLANG_SEARCH_PATH})

if("${CLANG_APPLY_REPLACEMENTS_BIN}" STREQUAL "CLANG_APPLY_REPLACEMENTS_BIN-NOTFOUND")
        message(WARNING "couldn't find clang-apply-replacements.")
else()
        # Output compile_commands.json
        set(CMAKE_EXPORT_COMPILE_COMMANDS 1)
        message(STATUS "found clang-apply-replacements at ${CLANG_APPLY_REPLACEMENTS_BIN}")
endif()

# sanitizer
if(NOT DEFINED SANITIZER)
  set(SANITIZER address)
endif()

message("Build mode: ${CMAKE_BUILD_TYPE}")
message("${SANITIZER} sanitizer will be enabled in debug mode.")

# cpplint
find_program(CPPLINT_BIN
        NAMES cpplint cpplint.py
        HINTS "${BUILD_SUPPORT_DIR}")

# valgrind
find_program(VALGRIND_COMMAND valgrind)

# #########################################
# "make check-lint"
# #########################################

if("${CPPLINT_BIN}" STREQUAL "CPPLINT_BIN-NOTFOUND")
        message(WARNING "couldn't find cpplint.")
else()
        message(STATUS "found cpplint at ${CPPLINT_BIN}")
endif()

# Balancing act: cpplint.py takes a non-trivial time to launch,
# so process 12 files per invocation, while still ensuring parallelism
file(GLOB_RECURSE LINT_FILES
        "${CMAKE_CURRENT_SOURCE_DIR}/src/*.h"
        "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/test/*.h"
        "${CMAKE_CURRENT_SOURCE_DIR}/test/*.cpp"
)
add_custom_target(check-lint echo '${LINT_FILES}' | xargs -n12 -P8
        ${CPPLINT_BIN}
        --verbose=2 --quiet
        --linelength=120
        --filter=-legal/copyright,-build/header_guard,-runtime/references # https://github.com/cpplint/cpplint/issues/148
)

# #########################################
# "make valgrind"
# #########################################
if("${VALGRIND_COMMAND}" STREQUAL "VALGRIND_COMMAND-NOTFOUND")
  message(WARNING "couldn't find valgrind.")
else()
  message(STATUS  "found valgrind at ${VALGRIND_COMMAND}")
  add_custom_target(valgrind
      COMMAND ${VALGRIND_COMMAND} --leak-check=full --track-origins=yes $<TARGET_FILE:rocket>
      DEPENDS rocket
      COMMENT "Running Valgrind..."
  )
endif()


# #####################################################################################################################
# COMPILER SETUP
# #####################################################################################################################
# Compiler flags.
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -Wall -Wextra -Werror")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -Wno-unused-parameter -Wno-attributes") # TODO: remove
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -O0 -ggdb -fsanitize=${SANITIZER} -fno-omit-frame-pointer -fno-optimize-sibling-calls")
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

message(STATUS "CMAKE_CXX_FLAGS: ${CMAKE_CXX_FLAGS}")
message(STATUS "CMAKE_CXX_FLAGS_DEBUG: ${CMAKE_CXX_FLAGS_DEBUG}")
message(STATUS "CMAKE_EXE_LINKER_FLAGS: ${CMAKE_EXE_LINKER_FLAGS}")
message(STATUS "CMAKE_SHARED_LINKER_FLAGS: ${CMAKE_SHARED_LINKER_FLAGS}")

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