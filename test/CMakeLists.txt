#
# test/CMakeLists.txt
#
# Author       : Guido Masella (guido.masella@gmail.com)
# Date created : 2022/02/22
#

# common library for all the tests it already links to qg8 so the test
# executables don't have to do it by themselves
add_library(qg8tests INTERFACE common_test.h)
target_include_directories(qg8tests INTERFACE
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/test>)
target_link_libraries(qg8tests INTERFACE qg8)
# hacky way to get the datafile working with ctest. Surely there are more
# elegant solutions, however for now the tests are bound to the absolute
# path of the data in source.
target_compile_definitions(qg8tests INTERFACE _TESTDATA_DIR="${PROJECT_SOURCE_DIR}/test/data")

# wrapper around add_executable for the tests here
function(add_qg8_executable name)
    add_executable(${name} ${ARGN})
    target_link_libraries(${name} PRIVATE qg8tests)
endfunction()

# wrapper around add_test for the tests here
function(add_qg8_test name)
    add_qg8_executable(${name} ${ARGN})
    target_compile_options(${name}
        PRIVATE
        -W -Wall -Wpedantic -Wextra)
    add_test(NAME qg8.${name} COMMAND $<TARGET_FILE:${name}>)
endfunction()

# Chunk TESTS
add_qg8_test(TestChunk chunk/chunk_test.c)

# File TESTS
add_qg8_test(TestFileRead file/file_read.c)
add_qg8_test(TestFileWrite file/file_write.c)

# Graph TESTS
add_qg8_test(TestGraphCreate graph/graph_create.c)
add_qg8_test(TestGraphLoad graph/graph_load.c)

# Tensor TESTS
add_qg8_test(TestTensor tensor/tensor_test.c)
# these other tests are defined to pass only if the execution fails
add_qg8_test(TestTensorBad1 tensor/bad_tensor1.c)
set_property(TEST qg8.TestTensorBad1 PROPERTY WILL_FAIL On)
add_qg8_test(TestTensorBad2 tensor/bad_tensor2.c)
set_property(TEST qg8.TestTensorBad2 PROPERTY WILL_FAIL On)
add_qg8_test(TestTensorBad3 tensor/bad_tensor3.c)
set_property(TEST qg8.TestTensorBad3 PROPERTY WILL_FAIL On)

# vim: set ft=cmake ts=2 sts=2 et sw=2 tw=80 : #
