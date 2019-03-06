#
# std::experimental::io
#

add_library(xio-test-config INTERFACE)

target_link_libraries(xio-test-config INTERFACE xio)
target_compile_definitions(xio-test-config INTERFACE $<$<CONFIG:Release>:NDEBUG>)
target_compile_options(xio-test-config INTERFACE
  -pedantic -pedantic-errors
  -march=native -ftemplate-backtrace-limit=0
  $<$<CXX_COMPILER_ID:GNU>:
    -Wall -Wextra
    -Wconversion -Wold-style-cast -Wfloat-equal
    -Wlogical-op -Wundef -Wredundant-decls
    -Wshadow -Wwrite-strings -Wpointer-arith
    -Wformat=2 -Wswitch-default -Wcast-qual
    -Wcast-align -Wswitch-enum -Wnon-virtual-dtor
    -Wctor-dtor-privacy -Wdisabled-optimization
    -Winvalid-pch -Wnoexcept -Wmissing-declarations
    -Woverloaded-virtual -Wdouble-promotion
    -Wtrampolines -Wzero-as-null-pointer-constant
    -Wuseless-cast -Wvector-operation-performance
    -Wshift-overflow=2 -Wnull-dereference -Walloc-zero
    -Wduplicated-cond -Walloca -Wduplicated-branches
    $<$<NOT:$<VERSION_LESS:CXX_COMPILER_VERSION,8.0>>:
      -Wcast-align=strict>
    >
  $<$<CXX_COMPILER_ID:Clang>:
    -Weverything
    -Wno-c++98-compat -Wno-c++98-compat-pedantic
    -Wno-c++98-compat-bind-to-temporary-copy
    -Wno-c++98-compat-local-type-template-args
    -Wno-padded -Wno-undefined-func-template
    -Wno-exit-time-destructors>
  $<$<BOOL:${XIO_WERROR}>:
    -Werror>
  $<$<CONFIG:Debug>:
    -O0 -fno-inline -g3 -fstack-protector-all>
  $<$<CONFIG:Release>:
    -Ofast -g0>)

find_package(Catch2 QUIET)
if(NOT Catch2_FOUND)
    FetchContent_Declare(
        catch2
        GIT_REPOSITORY https://github.com/catchorg/catch2.git
    )
    FetchContent_GetProperties(catch2)
    if(NOT catch2_POPULATED)
        FetchContent_Populate(catch2)
        add_subdirectory(${catch2_SOURCE_DIR} ${catch2_BINARY_DIR})
    endif()
endif()

function(add_xio_test TESTNAME EXENAME FIRSTSOURCE)
  add_executable(${EXENAME} ${FIRSTSOURCE} ${ARGN})
  target_link_libraries(${EXENAME} xio-test-config Catch2::Catch2)
  add_test(${TESTNAME} ${EXENAME})
endfunction()

add_xio_test(empty empty empty-test.cpp)
