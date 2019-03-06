#
# std::experimental::io
#
cmake_minimum_required(VERSION 3.13)

include(FetchContent)

project(experimental-io CXX)

# check if xio is a subproject
if(CMAKE_CURRENT_LIST_DIR STREQUAL CMAKE_SOURCE_DIR)
    set(NOT_SUBPROJECT ON)
endif()

option(XIO_WERROR "Use -Werror compiler flag" ON)

# required cmcstl2 dependency
find_package(cmcstl2 CONFIG QUIET)
if(NOT cmcstl2_FOUND)
    FetchContent_Declare(
        cmcstl2
        GIT_REPOSITORY https://github.com/CaseyCarter/cmcstl2.git
    )
    FetchContent_GetProperties(cmcstl2)
    if(NOT cmcstl2_POPULATED)
        FetchContent_Populate(cmcstl2)
        add_subdirectory(${cmcstl2_SOURCE_DIR} ${cmcstl2_BINARY_DIR})
    endif()
endif()

add_library(xio INTERFACE)
target_link_libraries(xio INTERFACE stl2)
target_include_directories(xio INTERFACE
    $<BUILD_INTERFACE:/include>
    $<INSTALL_INTERFACE:/include>)
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

# only enable the test suite if xio is being built as a standalone project
# or a test build is requested
if(NOT_SUBPROJECT OR BUILD_TESTING)
    include(CTest)
    add_subdirectory(test)
endif()
