# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.

cmake_minimum_required(VERSION 2.8.10)

# set CMAKE_C_COMPILER, this must set before project command
if (DEFINED ENV{DORIS_GCC_HOME})
    set(CMAKE_C_COMPILER "$ENV{DORIS_GCC_HOME}/bin/gcc")
    set(CMAKE_CXX_COMPILER "$ENV{DORIS_GCC_HOME}/bin/g++")
    set(GCC_HOME $ENV{DORIS_GCC_HOME})
else()
    message(FATAL_ERROR "DORIS_GCC_HOME environment variable is not set")
endif()

project(doris CXX C)

# set CMAKE_BUILD_TYPE
if (DEFINED ENV{BUILD_TYPE})
    set(CMAKE_BUILD_TYPE $ENV{BUILD_TYPE})
endif()
if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE RELEASE)
endif()

string(TOUPPER ${CMAKE_BUILD_TYPE} CMAKE_BUILD_TYPE)
message(STATUS "Build type is ${CMAKE_BUILD_TYPE}")

# set CMAKE_BUILD_TARGET_ARCH
# use `lscpu | grep 'Architecture' | awk '{print $2}'` only support system which language is en_US.UTF-8
execute_process(COMMAND bash "-c" "uname -m"
                OUTPUT_VARIABLE
                CMAKE_BUILD_TARGET_ARCH
                OUTPUT_STRIP_TRAILING_WHITESPACE)
message(STATUS "Build target arch is ${CMAKE_BUILD_TARGET_ARCH}")

# Set dirs
set(BASE_DIR "${CMAKE_CURRENT_SOURCE_DIR}")
set(ENV{DORIS_HOME} "${BASE_DIR}/../")
set(BUILD_DIR "${CMAKE_CURRENT_BINARY_DIR}")
set(THIRDPARTY_DIR "$ENV{DORIS_THIRDPARTY}/installed/")
set(GENSRC_DIR "${BASE_DIR}/../gensrc/build/")
set(SRC_DIR "${BASE_DIR}/src/")
set(TEST_DIR "${CMAKE_SOURCE_DIR}/test/")
set(OUTPUT_DIR "${BASE_DIR}/output")

if (APPLE)
    set(MAKE_TEST "ON")
else()
    option(MAKE_TEST "ON for make unit test or OFF for not" OFF)
endif()
message(STATUS "make test: ${MAKE_TEST}")
option(WITH_MYSQL "Support access MySQL" ON)

# Check gcc
if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    if (CMAKE_CXX_COMPILER_VERSION VERSION_LESS "4.8.2")
        message(FATAL_ERROR "Need GCC version at least 4.8.2")
    endif()
elseif (NOT APPLE)
    message(FATAL_ERROR "Compiler should be GNU")
endif()

set(PIC_LIB_PATH "${THIRDPARTY_DIR}")
if(PIC_LIB_PATH)
    message(STATUS "defined PIC_LIB_PATH")
    set(CMAKE_SKIP_RPATH TRUE)
    set(Boost_USE_STATIC_LIBS ON)
    set(Boost_USE_STATIC_RUNTIME ON)
    set(LIBBZ2 ${PIC_LIB_PATH}/lib/libbz2.a)
    set(LIBZ ${PIC_LIB_PATH}/lib/libz.a)
    set(LIBEVENT ${PIC_LIB_PATH}/lib/libevent.a)
else()
    message(STATUS "undefined PIC_LIB_PATH")
    set(Boost_USE_STATIC_LIBS ON)
    set(Boost_USE_STATIC_RUNTIME ON)
    set(LIBBZ2 -lbz2)
    set(LIBZ -lz)
    set(LIBEVENT event)
endif()


# Compile generated source if necessary
message(STATUS "build gensrc if necessary")
execute_process(COMMAND make -C ${BASE_DIR}/../gensrc/
                RESULT_VARIABLE MAKE_GENSRC_RESULT)
if(NOT ${MAKE_GENSRC_RESULT} EQUAL 0 AND NOT APPLE)
    message(FATAL_ERROR "Failed to build ${BASE_DIR}/../gensrc/")
endif()

# Set Boost
set(Boost_DEBUG FALSE)
set(Boost_USE_MULTITHREADED ON)
set(BOOST_ROOT ${THIRDPARTY_DIR})

if (NOT APPLE)
    find_package(Boost 1.55.0 REQUIRED COMPONENTS thread regex filesystem system date_time program_options)
else()
    find_package(Boost 1.55.0 COMPONENTS thread regex filesystem system date_time program_options)
endif()
include_directories(${Boost_INCLUDE_DIRS})
message(STATUS ${Boost_LIBRARIES})

set(GPERFTOOLS_HOME "${THIRDPARTY_DIR}/gperftools")

# Set all libraries
add_library(gflags STATIC IMPORTED)
set_target_properties(gflags PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libgflags.a)

add_library(glog STATIC IMPORTED)
set_target_properties(glog PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libglog.a)

add_library(re2 STATIC IMPORTED)
set_target_properties(re2 PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libre2.a)

add_library(pprof STATIC IMPORTED)
set_target_properties(pprof PROPERTIES IMPORTED_LOCATION
    ${GPERFTOOLS_HOME}/lib/libprofiler.a)

add_library(tcmalloc STATIC IMPORTED)
set_target_properties(tcmalloc PROPERTIES IMPORTED_LOCATION
    ${GPERFTOOLS_HOME}/lib/libtcmalloc.a)

add_library(protobuf STATIC IMPORTED)
set_target_properties(protobuf PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libprotobuf.a)

add_library(protoc STATIC IMPORTED)
set_target_properties(protoc PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libprotoc.a)

add_library(gtest STATIC IMPORTED)
set_target_properties(gtest PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libgtest.a)

add_library(gmock STATIC IMPORTED)
set_target_properties(gmock PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libgmock.a)

add_library(snappy STATIC IMPORTED)
set_target_properties(snappy PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libsnappy.a)

add_library(curl STATIC IMPORTED)
set_target_properties(curl PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libcurl.a)

add_library(lz4 STATIC IMPORTED)
set_target_properties(lz4 PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/liblz4.a)

add_library(thrift STATIC IMPORTED)
set_target_properties(thrift PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libthrift.a)

add_library(thriftnb STATIC IMPORTED)
set_target_properties(thriftnb PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libthriftnb.a)

if(WITH_LZO)
    add_library(lzo STATIC IMPORTED)
    set_target_properties(lzo PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/liblzo2.a)
endif()

if (WITH_MYSQL)
    add_library(mysql STATIC IMPORTED)
    set_target_properties(mysql PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libmysqlclient.a)
endif()

add_library(libevent STATIC IMPORTED)
set_target_properties(libevent PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libevent.a)

add_library(crypto STATIC IMPORTED)
set_target_properties(crypto PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libcrypto.a)

add_library(openssl STATIC IMPORTED)
set_target_properties(openssl PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libssl.a)

add_library(leveldb STATIC IMPORTED)
set_target_properties(leveldb PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libleveldb.a)

add_library(jemalloc STATIC IMPORTED)
set_target_properties(jemalloc PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib64/libjemalloc.a)

add_library(brotlicommon STATIC IMPORTED)
set_target_properties(brotlicommon PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib64/libbrotlicommon.a)

add_library(brotlidec STATIC IMPORTED)
set_target_properties(brotlidec PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib64/libbrotlidec.a)

add_library(brotlienc STATIC IMPORTED)
set_target_properties(brotlienc PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib64/libbrotlienc.a)

add_library(zstd STATIC IMPORTED)
set_target_properties(zstd PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib64/libzstd.a)

add_library(arrow STATIC IMPORTED)
set_target_properties(arrow PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib64/libarrow.a)

add_library(double-conversion STATIC IMPORTED)
set_target_properties(double-conversion PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib64/libdouble-conversion.a)

add_library(parquet STATIC IMPORTED)
set_target_properties(parquet PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib64/libparquet.a)

add_library(brpc STATIC IMPORTED)
set_target_properties(brpc PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib64/libbrpc.a)

add_library(rocksdb STATIC IMPORTED)
set_target_properties(rocksdb PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/librocksdb.a)

add_library(librdkafka_cpp STATIC IMPORTED)
set_target_properties(librdkafka_cpp PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/librdkafka++.a)

add_library(librdkafka STATIC IMPORTED)
set_target_properties(librdkafka PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/librdkafka.a)

add_library(libs2 STATIC IMPORTED)
set_target_properties(libs2 PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libs2.a)

add_library(bitshuffle STATIC IMPORTED)
set_target_properties(bitshuffle PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libbitshuffle.a)

add_library(roaring STATIC IMPORTED)
set_target_properties(roaring PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/libroaring.a)

add_library(orc STATIC IMPORTED)
set_target_properties(orc PROPERTIES IMPORTED_LOCATION ${THIRDPARTY_DIR}/lib/liborc.a)

find_program(THRIFT_COMPILER thrift ${CMAKE_SOURCE_DIR}/bin)

# Check if functions are supported in this platform. All flags will generated
# in gensrc/build/common/env_config.h.
# You can check funcion here which depends on platform. Don't forget add this
# to be/src/common/env_config.h.in
include(CheckFunctionExists)
check_function_exists(sched_getcpu HAVE_SCHED_GETCPU)

# compiler flags that are common across debug/release builds
#  -Wall: Enable all warnings.
#  -Wno-sign-compare: suppress warnings for comparison between signed and unsigned
#    integers
#   -fno-strict-aliasing: disable optimizations that assume strict aliasing. This
#       is unsafe to do if the code uses casts (which we obviously do).
#  -Wno-unknown-pragmas: suppress warnings for unknown (compiler specific) pragmas
#  -Wno-deprecated: gutil contains deprecated headers
#  -Wno-vla: we use C99-style variable-length arrays
#  -pthread: enable multithreaded malloc
#  -DBOOST_DATE_TIME_POSIX_TIME_STD_CONFIG: enable nanosecond precision for boost
#  -fno-omit-frame-pointers: Keep frame pointer for functions in register
set(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -Wall -Wno-sign-compare -Wno-unknown-pragmas -pthread")
set(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -fno-strict-aliasing -fno-omit-frame-pointer")
set(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -std=gnu++11 -D__STDC_FORMAT_MACROS")
set(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -Wno-deprecated -Wno-vla")
set(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -DBOOST_DATE_TIME_POSIX_TIME_STD_CONFIG")
set(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -DBOOST_SYSTEM_NO_DEPRECATED")
if ("${CMAKE_BUILD_TARGET_ARCH}" STREQUAL "x86" OR "${CMAKE_BUILD_TARGET_ARCH}" STREQUAL "x86_64")
    set(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -msse4.2")
endif()
set(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS}  -Wno-attributes -DS2_USE_GFLAGS -DS2_USE_GLOG")

if (WITH_MYSQL)
    set(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -DDORIS_WITH_MYSQL")
endif()

if (WITH_LZO)
    set(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -DDORIS_WITH_LZO")
endif()

if (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 7.0)
    set(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -faligned-new")
endif()

# For any gcc builds:
#   -g: Enable symbols for profiler tools
#   -Wno-unused-local-typedefs: Do not warn for local typedefs that are unused.
set(CXX_GCC_FLAGS "-g -Wno-unused-local-typedefs")

# For CMAKE_BUILD_TYPE=Debug
#   -ggdb: Enable gdb debugging
# Debug information is stored as dwarf2 to be as compatible as possible
#   -Werror: compile warnings should be errors when using the toolchain compiler.
# Only enable for debug builds because this is what we test in pre-commit tests.
set(CXX_FLAGS_DEBUG "${CXX_GCC_FLAGS} -Werror -ggdb -O0 -gdwarf-2")

# For CMAKE_BUILD_TYPE=Release
#   -O3: Enable all compiler optimizations
#   -DNDEBUG: Turn off dchecks/asserts/debug only code.
#   -gdwarf-2: Debug information is stored as dwarf2 to be as compatible as possible
set(CXX_FLAGS_RELEASE "${CXX_GCC_FLAGS} -O3 -gdwarf-2 -DNDEBUG")

SET(CXX_FLAGS_ASAN "${CXX_GCC_FLAGS} -O1 -fsanitize=address -DADDRESS_SANITIZER")
SET(CXX_FLAGS_LSAN "${CXX_GCC_FLAGS} -O1 -fsanitize=leak -DLEAK_SANITIZER")

# Set the flags to the undefined behavior sanitizer, also known as "ubsan"
# Turn on sanitizer and debug symbols to get stack traces:
SET(CXX_FLAGS_UBSAN "${CXX_GCC_FLAGS} -ggdb3 -fsanitize=undefined")
# Ignore a number of noisy errors with too many false positives:
# TODO(zc): 
# SET(CXX_FLAGS_UBSAN "${CXX_FLAGS_UBSAN} -fno-sanitize=alignment,function,vptr,float-divide-by-zero,float-cast-overflow")
# Don't enforce wrapped signed integer arithmetic so that the sanitizer actually sees
# undefined wrapping:
SET(CXX_FLAGS_UBSAN "${CXX_FLAGS_UBSAN} -fno-wrapv")
# To ease debugging, turn off all optimizations:
SET(CXX_FLAGS_UBSAN "${CXX_FLAGS_UBSAN} -O0")

# Set the flags to the thread sanitizer, also known as "tsan"
# Turn on sanitizer and debug symbols to get stack traces:
SET(CXX_FLAGS_TSAN "${CXX_GCC_FLAGS} -O1 -ggdb3 -fsanitize=thread -DTHREAD_SANITIZER")

# Set compile flags based on the build type.
if ("${CMAKE_BUILD_TYPE}" STREQUAL "DEBUG")
    SET(CMAKE_CXX_FLAGS ${CXX_FLAGS_DEBUG})
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "RELEASE" OR "${CMAKE_BUILD_TYPE}" STREQUAL "BCC")
    SET(CMAKE_CXX_FLAGS ${CXX_FLAGS_RELEASE})
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "ASAN")
    SET(CMAKE_CXX_FLAGS "${CXX_FLAGS_ASAN}")
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "LSAN")
    SET(CMAKE_CXX_FLAGS "${CXX_FLAGS_LSAN}")
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "UBSAN")
    SET(CMAKE_CXX_FLAGS "${CXX_FLAGS_UBSAN}")
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "TSAN")
    SET(CMAKE_CXX_FLAGS "${CXX_FLAGS_TSAN}")
else()
    message(FATAL_ERROR "Unknown build type: ${CMAKE_BUILD_TYPE}")
endif()

# Add flags that are common across build types
SET(CMAKE_CXX_FLAGS "${CXX_COMMON_FLAGS} ${CMAKE_CXX_FLAGS}")

message(STATUS "Compiler Flags: ${CMAKE_CXX_FLAGS}")

# Thrift requires these two definitions for some types that we use
add_definitions(-DHAVE_INTTYPES_H -DHAVE_NETINET_IN_H)

# Set include dirs
include_directories( 
    ${SRC_DIR}/
    ${TEST_DIR}/
    ${GENSRC_DIR}/
    ${THIRDPARTY_DIR}/include
    ${GPERFTOOLS_HOME}/include
    ${THIRDPARTY_DIR}/include/thrift/
    ${THIRDPARTY_DIR}/include/event/
)

set(WL_START_GROUP "-Wl,--start-group")
set(WL_END_GROUP "-Wl,--end-group")

# Set Palo libraries
set(DORIS_LINK_LIBS
    ${WL_START_GROUP}
    Agent
    Common
    Env
    Exec
    Exprs
    Gutil
    Olap
    Rowset
    OlapFs
    Runtime
    Service
    Udf
    Util
    DorisGen
    Webserver
    TestUtil
    Geo
    Plugin        
    ${WL_END_GROUP}
)

# Set thirdparty libraries
set(DORIS_DEPENDENCIES
    ${DORIS_DEPENDENCIES}
    ${WL_START_GROUP}
    rocksdb
    librdkafka_cpp
    librdkafka
    libs2
    snappy
    ${Boost_LIBRARIES}
    thrift
    thriftnb
    glog
    re2
    pprof
    lz4
    libevent
    curl
    ${LIBZ}
    ${LIBBZ2}
    gflags
    brpc
    protobuf
    openssl
    crypto
    leveldb
    bitshuffle
    roaring
    double-conversion
    jemalloc
    brotlicommon
    brotlidec
    brotlienc
    zstd
    arrow
    parquet
    orc
    ${WL_END_GROUP}
)

if(WITH_LZO)
    set(DORIS_DEPENDENCIES ${DORIS_DEPENDENCIES}
        lzo
    )
endif()

if (WITH_MYSQL)
    set(DORIS_DEPENDENCIES ${DORIS_DEPENDENCIES}
        mysql
        )
endif()

# Add all external dependencies. They should come after the palo libs.
# static link gcc's lib
set(DORIS_LINK_LIBS ${DORIS_LINK_LIBS}
    ${DORIS_DEPENDENCIES}
    -static-libstdc++
    -static-libgcc
)

if ("${CMAKE_BUILD_TYPE}" STREQUAL "BCC")
    set(DORIS_LINK_LIBS ${DORIS_LINK_LIBS}
        -Wl,--dynamic-linker=/lib64/ld-linux-x86-64.so.2
    )
endif()

# Add sanitize static link flags or tcmalloc
if ("${CMAKE_BUILD_TYPE}" STREQUAL "DEBUG" OR "${CMAKE_BUILD_TYPE}" STREQUAL "RELEASE" OR "${CMAKE_BUILD_TYPE}" STREQUAL "BCC")
    set(DORIS_LINK_LIBS ${DORIS_LINK_LIBS} tcmalloc)
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "ASAN")
    set(DORIS_LINK_LIBS ${DORIS_LINK_LIBS} -static-libasan)
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "LSAN")
    set(DORIS_LINK_LIBS ${DORIS_LINK_LIBS} -static-liblsan)
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "UBSAN")
    set(DORIS_LINK_LIBS ${DORIS_LINK_LIBS} -static-libubsan tcmalloc)
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "TSAN")
    set(DORIS_LINK_LIBS ${DORIS_LINK_LIBS} -static-libtsan)
else()
    message(FATAL_ERROR "Unknown build type: ${CMAKE_BUILD_TYPE}")
endif()

set(DORIS_LINK_LIBS ${DORIS_LINK_LIBS}
    -lrt -lbfd -liberty -lc -lm -ldl -pthread
)

# Set libraries for test
set (TEST_LINK_LIBS ${DORIS_LINK_LIBS} 
    ${WL_START_GROUP}
    gmock
    gtest
    ${WL_END_GROUP}
)

# Only build static libs
set(BUILD_SHARED_LIBS OFF)

if (${MAKE_TEST} STREQUAL "ON")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-arcs -ftest-coverage -DGTEST_USE_OWN_TR1_TUPLE=0")
    SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fprofile-arcs -ftest-coverage -lgcov")
    add_definitions(-DBE_TEST)
endif ()

add_subdirectory(${SRC_DIR}/agent)
add_subdirectory(${SRC_DIR}/common)
add_subdirectory(${SRC_DIR}/env)
add_subdirectory(${SRC_DIR}/exec)
add_subdirectory(${SRC_DIR}/exprs)
add_subdirectory(${SRC_DIR}/gen_cpp)
add_subdirectory(${SRC_DIR}/geo)
add_subdirectory(${SRC_DIR}/gutil)
add_subdirectory(${SRC_DIR}/http)
add_subdirectory(${SRC_DIR}/olap)
add_subdirectory(${SRC_DIR}/runtime)
add_subdirectory(${SRC_DIR}/service)
add_subdirectory(${SRC_DIR}/testutil)
#add_subdirectory(${SRC_DIR}/tools)
add_subdirectory(${SRC_DIR}/udf)
add_subdirectory(${SRC_DIR}/tools)
add_subdirectory(${SRC_DIR}/udf_samples)
add_subdirectory(${SRC_DIR}/util)
add_subdirectory(${SRC_DIR}/plugin)

# Utility CMake function to make specifying tests and benchmarks less verbose
FUNCTION(ADD_BE_TEST TEST_NAME)
    set(BUILD_OUTPUT_ROOT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/")
    # This gets the directory where the test is from (e.g. 'exprs' or 'runtime')
    get_filename_component(DIR_NAME ${CMAKE_CURRENT_SOURCE_DIR} NAME)
    get_filename_component(TEST_DIR_NAME ${TEST_NAME} PATH)
    get_filename_component(TEST_FILE_NAME ${TEST_NAME} NAME)

    ADD_EXECUTABLE(${TEST_FILE_NAME} ${TEST_NAME}.cpp)
    TARGET_LINK_LIBRARIES(${TEST_FILE_NAME} ${TEST_LINK_LIBS})
    SET_TARGET_PROPERTIES(${TEST_FILE_NAME} PROPERTIES COMPILE_FLAGS "-fno-access-control")
    if (NOT "${TEST_DIR_NAME}" STREQUAL "")
        SET_TARGET_PROPERTIES(${TEST_FILE_NAME} PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${BUILD_OUTPUT_ROOT_DIRECTORY}/${TEST_DIR_NAME}")
    endif()
    ADD_TEST(${TEST_FILE_NAME} "${BUILD_OUTPUT_ROOT_DIRECTORY}/${TEST_NAME}")
ENDFUNCTION()

FUNCTION(ADD_BE_PLUGIN PLUGIN_NAME)
    set(BUILD_OUTPUT_ROOT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/")
    
    get_filename_component(DIR_NAME ${CMAKE_CURRENT_SOURCE_DIR} NAME)
    get_filename_component(PLUGIN_DIR_NAME ${PLUGIN_NAME} PATH)
    get_filename_component(PLUGIN_FILE_NAME ${PLUGIN_NAME} NAME)
    
    ADD_LIBRARY(${PLUGIN_FILE_NAME} SHARED ${PLUGIN_NAME}.cpp)
    
    TARGET_LINK_LIBRARIES(${PLUGIN_FILE_NAME} ${DORIS_LINK_LIBS})
    SET_TARGET_PROPERTIES(${PLUGIN_FILE_NAME} PROPERTIES COMPILE_FLAGS "-fno-access-control")

    if (NOT "${PLUGIN_DIR_NAME}" STREQUAL "")
        SET_TARGET_PROPERTIES(${PLUGIN_FILE_NAME} PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${BUILD_OUTPUT_ROOT_DIRECTORY}/${PLUGIN_DIR_NAME}")
    endif ()

ENDFUNCTION()

if (${MAKE_TEST} STREQUAL "ON")
    add_subdirectory(${TEST_DIR}/agent)
    add_subdirectory(${TEST_DIR}/common)
    add_subdirectory(${TEST_DIR}/env)
    add_subdirectory(${TEST_DIR}/exec)
    add_subdirectory(${TEST_DIR}/exprs)
    add_subdirectory(${TEST_DIR}/geo)
    add_subdirectory(${TEST_DIR}/http)
    add_subdirectory(${TEST_DIR}/olap)
    add_subdirectory(${TEST_DIR}/runtime)
    add_subdirectory(${TEST_DIR}/udf)
    add_subdirectory(${TEST_DIR}/util)
    add_subdirectory(${TEST_DIR}/plugin)
    add_subdirectory(${TEST_DIR}/plugin/example)
endif ()

# Install be
install(DIRECTORY DESTINATION ${OUTPUT_DIR})
install(DIRECTORY DESTINATION ${OUTPUT_DIR}/bin)
install(DIRECTORY DESTINATION ${OUTPUT_DIR}/conf)

install(FILES
    ${BASE_DIR}/../bin/start_be.sh
    ${BASE_DIR}/../bin/stop_be.sh
    PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE
    GROUP_READ GROUP_WRITE GROUP_EXECUTE
    WORLD_READ WORLD_EXECUTE
    DESTINATION ${OUTPUT_DIR}/bin)

install(FILES
    ${BASE_DIR}/../conf/be.conf
    DESTINATION ${OUTPUT_DIR}/conf)

