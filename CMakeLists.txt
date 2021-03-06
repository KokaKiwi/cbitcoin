cmake_minimum_required(VERSION 2.8)
project(cbitcoin)

# Paths
set(PROJECT_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}")
set(PROJECT_BUILD_DIR "${CMAKE_CURRENT_BUILD_DIR}")

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${PROJECT_SOURCE_DIR}/cmake")

set(DEBUG ON CACHE BOOL "Set debug.")
set(STATIC_LIB ON CACHE BOOL "Compile into static/shared libraries.")

if (DEBUG)
    set(OFLAGS "-ggdb3")
else (DEBUG)
    # On gcc, -O4 is used as -O3, but clang will use it as -O4
    # Should detect which compiler is used
    set(OFLAGS "-O4 -march=native")
endif (DEBUG)

set(CMAKE_C_FLAGS "-Wall ${OFLAGS}")

find_package(OpenSSL REQUIRED)
include_directories(${OPENSSL_INCLUDE_DIR})

find_package(Event REQUIRED)
include_directories(${EVENT_INCLUDE_DIR})

find_package(Threads REQUIRED)

if (STATIC_LIB)
    set(LIB_TYPE STATIC)
else (STATIC_LIB)
    set(LIB_TYPE SHARED)
endif (STATIC_LIB)

# Core libraries
set(LIBS
    cbitcoin
    cbitcoin-crypto
    cbitcoin-random
    cbitcoin-network
    cbitcoin-storage
)

file(GLOB crypto_SOURCE dependencies/crypto/*.c)
include_directories(dependencies/crypto)
add_library(cbitcoin-crypto ${LIB_TYPE} ${crypto_SOURCE})
target_link_libraries(cbitcoin-crypto ${OPENSSL_LIBRARIES})

file(GLOB random_SOURCE dependencies/random/*.c)
include_directories(dependencies/random)
add_library(cbitcoin-random ${LIB_TYPE} ${random_SOURCE})

file(GLOB network_SOURCE dependencies/sockets/*.c)
include_directories(dependencies/sockets)
add_library(cbitcoin-network ${LIB_TYPE} ${network_SOURCE})
target_link_libraries(cbitcoin-network ${EVENT_LIBRARIES})

file(GLOB storage_SOURCE dependencies/storage/*.c)
include_directories(dependencies/storage)
add_library(cbitcoin-storage ${LIB_TYPE} ${storage_SOURCE})

file(GLOB core_SOURCES src/*.c)
include_directories(include)
add_library(cbitcoin ${LIB_TYPE} ${core_SOURCES})
target_link_libraries(cbitcoin
    cbitcoin-crypto
    cbitcoin-random
    cbitcoin-network
    cbitcoin-storage
)

install(
    TARGETS
        ${LIBS}
    DESTINATION
        lib
)

# Tests
enable_testing()

file(
    COPY
        ${PROJECT_SOURCE_DIR}/test/scriptCases.txt
    DESTINATION
        ${CMAKE_BINARY_DIR}/test
)

file(GLOB tests_SOURCE test/*.c)
foreach (test ${tests_SOURCE})
    get_filename_component(test_name ${test} NAME_WE)

    add_executable(${test_name} ${test})
    target_link_libraries(${test_name}
        ${LIBS}
        ${CMAKE_THREAD_LIBS_INIT}
        ${OPENSSL_LIBRARIES}
        ${EVENT_LIBRARIES}
    )

    add_test(test_${test_name} ${test_name})
endforeach (test)

# Doc
find_package(Doxygen)
if (DOXYGEN_FOUND)
    add_custom_target(doc
        COMMAND
            doxygen ${PROJECT_SOURCE_DIR}/Doxyfile
        WORKING_DIRECTORY
            ${PROJECT_SOURCE_DIR}
    )
endif (DOXYGEN_FOUND)
