#
# std::experimental::io
#
#   Copyright Dalton M. Woodard 2019
#
#   Use, modification and distribution is subject to the Boost Software
#   License, Version 1.0. (See accompanying file LICENSE_1_0.txt or copy at
#   http://www.boost.org/LICENSE_1_0.txt)
#
# https://github.com/experimental-io/experimental-io
#
cmake_minimum_required(VERSION 3.13)

include(FetchContent)

project(experimental-io CXX)

# check if xio is being built standalone
if(CMAKE_CURRENT_LIST_DIR STREQUAL CMAKE_SOURCE_DIR)
    option(XIO_BUILD_TESTING "Build xio tests" ON)
else()
    option(XIO_BUILD_TESTING "Build xio tests" OFF)
endif()

option(XIO_WERROR "Use -Werror compiler flag" ON)

# required cmcstl2 dependency
find_package(cmcstl2 CONFIG QUIET)
if(NOT cmcstl2_FOUND)
    FetchContent_Declare(
        cmcstl2
        GIT_REPOSITORY https://github.com/daltonwoodard/cmcstl2.git
        GIT_TAG cmake-adjustment-for-subproject
    )
    FetchContent_GetProperties(cmcstl2)
    if(NOT cmcstl2_POPULATED)
        FetchContent_Populate(cmcstl2)
        set(STL2_BUILD_EXAMPLES OFF CACHE BOOL "")
        set(STL2_BUILD_TESTING OFF CACHE BOOL "")
        add_subdirectory(${cmcstl2_SOURCE_DIR} ${cmcstl2_BINARY_DIR})
    endif()
endif()

add_library(xio INTERFACE)
target_include_directories(xio SYSTEM INTERFACE $<TARGET_PROPERTY:stl2,INTERFACE_INCLUDE_DIRECTORIES>)
target_link_libraries(xio INTERFACE stl2)
target_include_directories(xio INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:$<INSTALL_PREFIX>/include>)
target_compile_features(xio INTERFACE cxx_std_17)
target_compile_options(xio INTERFACE
    $<$<CXX_COMPILER_ID:GNU>:-fconcepts>
    $<$<CXX_COMPILER_ID:Clang>:-Xclang -fconcepts-ts>)

install(DIRECTORY include/ DESTINATION include)
install(TARGETS xio EXPORT experimental-io-targets)
install(EXPORT experimental-io-targets DESTINATION lib/cmake/experimental-io)
file(WRITE ${PROJECT_BINARY_DIR}/experimental-io-config.cmake
    "include(\${CMAKE_CURRENT_LIST_DIR}/experimental-io-targets.cmake)")
install(FILES ${PROJECT_BINARY_DIR}/experimental-io-config.cmake
    DESTINATION lib/cmake/experimental-io)

if(XIO_BUILD_TESTING)
    include(CTest)
    add_custom_target(xio-check ${CMAKE_CTEST_COMMAND} -V)
    add_subdirectory(test)
endif()
