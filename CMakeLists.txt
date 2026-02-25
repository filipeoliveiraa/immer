cmake_minimum_required(VERSION 3.21)
cmake_policy(SET CMP0048 NEW) # enable project VERSION
cmake_policy(SET CMP0056 NEW) # honor link flags in try_compile()
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

project(immer VERSION 0.9.0)

set(CMAKE_EXPORT_COMPILE_COMMANDS on)
set(CMAKE_CXX_EXTENSIONS off)

if(NOT MSVC)
  set(CMAKE_CXX_FLAGS
      "${CMAKE_CXX_FLAGS} -Wall -Wextra -pedantic -Wno-unused-parameter -Wno-type-limits"
  )
endif()

if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
  set(CMAKE_CXX_FLAGS
      "${CMAKE_CXX_FLAGS} -Wall -Wextra -pedantic -Wno-unused-parameter -Wno-type-limits"
  )
endif()
if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
  set(CMAKE_CXX_FLAGS
      "${CMAKE_CXX_FLAGS} -Qunused-arguments -Wno-unknown-warning-option")
endif()

include(GNUInstallDirs)
include(ImmerUtils)

# Options
# =======

option(ENABLE_ASAN "compile with address sanitizer enabled")
option(ENABLE_MSAN "compile with memory sanitizer enabled")
option(ENABLE_LSAN "compile with leak sanitizer enabled")
option(ENABLE_UBSAN "compile with undefined behavior sanitizer enabled")
option(ENABLE_COVERAGE "compile with test coverage support")
option(ENABLE_VALGRIND "use valgrind when running tests")

option(DISABLE_WERROR "disable --werror")
option(DISABLE_FREE_LIST "disables the free list heap")
option(DISABLE_THREAD_SAFETY "disables thread safety by default")

option(CHECK_FUZZERS "Add fuzzers as part of make check" off)
option(CHECK_BENCHMARKS "Run benchmarks on check target" off)

option(ENABLE_PYTHON "enable building python module" off)
option(ENABLE_GUILE "enable building guile module" off)
option(ENABLE_BOOST_COROUTINE "run benchmarks with boost coroutine" off)

option(immer_BUILD_TESTS "Build tests" "${PROJECT_IS_TOP_LEVEL}")
option(immer_BUILD_PERSIST_TESTS "Build experimental persist tests" off)
option(immer_BUILD_EXAMPLES "Build examples" "${PROJECT_IS_TOP_LEVEL}")
option(immer_BUILD_DOCS "Build docs" "${PROJECT_IS_TOP_LEVEL}")
option(immer_BUILD_EXTRAS "Build extras" "${PROJECT_IS_TOP_LEVEL}")
option(immer_INSTALL_FUZZERS "Install fuzzers" off)
option(immer_ENABLE_EXCEPTIONS
       "Always enable exceptions regardless of detected compiler support" OFF)
option(immer_DISABLE_EXCEPTIONS
       "Always disable exceptions regardless of detected compiler support" OFF)

if(immer_ENABLE_EXCEPTIONS AND immer_DISABLE_EXCEPTIONS)
  message(FATAL_ERROR "Cannot both enable and disable exceptions")
endif()

if(immer_BUILD_PERSIST_TESTS)
  set(CXX_STANDARD
      17
      CACHE STRING "c++ standard number")
else()
  set(CXX_STANDARD
      14
      CACHE STRING "c++ standard number")
endif()

set(CMAKE_CXX_STANDARD ${CXX_STANDARD})
set(CMAKE_CXX_STANDARD_REQUIRED on)

if(ENABLE_ASAN)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address")
endif()
if(ENABLE_LSAN)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=leak")
endif()
if(ENABLE_UBSAN)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=undefined")
endif()
if(ENABLE_MSAN)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=memory")
endif()

if(NOT MSVC AND NOT DISABLE_WERROR)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror")
endif()

# Dependencies
# ============

if(ENABLE_BOOST_COROUTINE)
  set(immer_boost_components coroutine)
endif()

find_package(Threads)
find_package(BoehmGC)
find_package(Boost CONFIG 1.56 COMPONENTS ${immer_boost_components})

find_program(CCACHE ccache)
if(CCACHE)
  message(STATUS "Using ccache: ${CCACHE}")
  set(CMAKE_CXX_COMPILER_LAUNCHER ${CCACHE})
  set(CMAKE_CXX_LINKER_LAUNCHER ${CCACHE})
else()
  message(STATUS "Could not find ccache")
endif()

if(NOT BoehmGC_FOUND)
  set(BoehmGC_LIBRARIES "")
endif()

# Targets
# =======

# the library
add_library(immer INTERFACE)
add_library(immer::immer ALIAS immer)
target_include_directories(
  immer
  INTERFACE $<BUILD_INTERFACE:${immer_BINARY_DIR}/>
            $<BUILD_INTERFACE:${immer_SOURCE_DIR}/>
            $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)

if(immer_ENABLE_EXCEPTIONS)
  message(STATUS "Explicitly enabling exceptions")
  target_compile_definitions(immer INTERFACE IMMER_USE_EXCEPTIONS)
endif()

if(immer_DISABLE_EXCEPTIONS)
  message(STATUS "Explicitly disabling exceptions")
  target_compile_definitions(immer INTERFACE IMMER_NO_EXCEPTIONS)
endif()

target_sources(immer PUBLIC FILE_SET HEADERS
  FILES
    immer/algorithm.hpp
    immer/array.hpp
    immer/array_transient.hpp
    immer/atom.hpp
    immer/box.hpp
    immer/config.hpp
    immer/detail/arrays/no_capacity.hpp
    immer/detail/arrays/node.hpp
    immer/detail/arrays/with_capacity.hpp
    immer/detail/combine_standard_layout.hpp
    immer/detail/hamts/bits.hpp
    immer/detail/hamts/champ.hpp
    immer/detail/hamts/champ_iterator.hpp
    immer/detail/hamts/node.hpp
    immer/detail/iterator_facade.hpp
    immer/detail/rbts/bits.hpp
    immer/detail/rbts/node.hpp
    immer/detail/rbts/operations.hpp
    immer/detail/rbts/position.hpp
    immer/detail/rbts/rbtree.hpp
    immer/detail/rbts/rbtree_iterator.hpp
    immer/detail/rbts/rrbtree.hpp
    immer/detail/rbts/rrbtree_iterator.hpp
    immer/detail/rbts/visitor.hpp
    immer/detail/ref_count_base.hpp
    immer/detail/type_traits.hpp
    immer/detail/util.hpp
    immer/experimental/detail/dvektor_impl.hpp
    immer/experimental/dvektor.hpp
    immer/extra/cereal/immer_array.hpp
    immer/extra/cereal/immer_box.hpp
    immer/extra/cereal/immer_map.hpp
    immer/extra/cereal/immer_set.hpp
    immer/extra/cereal/immer_table.hpp
    immer/extra/cereal/immer_vector.hpp
    immer/extra/io.hpp
    immer/extra/persist/cereal/archives.hpp
    immer/extra/persist/cereal/load.hpp
    immer/extra/persist/cereal/policy.hpp
    immer/extra/persist/cereal/save.hpp
    immer/extra/persist/detail/alias.hpp
    immer/extra/persist/detail/array/pool.hpp
    immer/extra/persist/detail/box/pool.hpp
    immer/extra/persist/detail/cereal/compact_map.hpp
    immer/extra/persist/detail/cereal/input_archive_util.hpp
    immer/extra/persist/detail/cereal/persistable.hpp
    immer/extra/persist/detail/cereal/pools.hpp
    immer/extra/persist/detail/cereal/wrap.hpp
    immer/extra/persist/detail/champ/champ.hpp
    immer/extra/persist/detail/champ/input.hpp
    immer/extra/persist/detail/champ/output.hpp
    immer/extra/persist/detail/champ/pool.hpp
    immer/extra/persist/detail/champ/traits.hpp
    immer/extra/persist/detail/common/pool.hpp
    immer/extra/persist/detail/names.hpp
    immer/extra/persist/detail/node_ptr.hpp
    immer/extra/persist/detail/rbts/input.hpp
    immer/extra/persist/detail/rbts/output.hpp
    immer/extra/persist/detail/rbts/pool.hpp
    immer/extra/persist/detail/rbts/traits.hpp
    immer/extra/persist/detail/rbts/traverse.hpp
    immer/extra/persist/detail/traits.hpp
    immer/extra/persist/detail/transform.hpp
    immer/extra/persist/detail/type_traverse.hpp
    immer/extra/persist/errors.hpp
    immer/extra/persist/hash_container_conversion.hpp
    immer/extra/persist/transform.hpp
    immer/extra/persist/types.hpp
    immer/extra/persist/xxhash/xxhash.hpp
    immer/extra/persist/xxhash/xxhash_64.cpp
    immer/flex_vector.hpp
    immer/flex_vector_transient.hpp
    immer/heap/cpp_heap.hpp
    immer/heap/debug_size_heap.hpp
    immer/heap/free_list_heap.hpp
    immer/heap/gc_heap.hpp
    immer/heap/heap_policy.hpp
    immer/heap/identity_heap.hpp
    immer/heap/malloc_heap.hpp
    immer/heap/split_heap.hpp
    immer/heap/tags.hpp
    immer/heap/thread_local_free_list_heap.hpp
    immer/heap/unsafe_free_list_heap.hpp
    immer/lock/no_lock_policy.hpp
    immer/lock/spinlock_policy.hpp
    immer/map.hpp
    immer/map_transient.hpp
    immer/memory_policy.hpp
    immer/refcount/enable_intrusive_ptr.hpp
    immer/refcount/no_refcount_policy.hpp
    immer/refcount/refcount_policy.hpp
    immer/refcount/unsafe_refcount_policy.hpp
    immer/set.hpp
    immer/set_transient.hpp
    immer/table.hpp
    immer/table_transient.hpp
    immer/transience/gc_transience_policy.hpp
    immer/transience/no_transience_policy.hpp
    immer/vector.hpp
    immer/vector_transient.hpp
)

install(TARGETS immer EXPORT ImmerConfig FILE_SET HEADERS)
install(EXPORT ImmerConfig DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/Immer")
install(DIRECTORY immer DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/ImmerConfigVersion.cmake"
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY SameMajorVersion ARCH_INDEPENDENT)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/ImmerConfigVersion.cmake"
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/Immer")

# development target to be used in tests, examples, benchmarks...
immer_canonicalize_cmake_booleans(DISABLE_FREE_LIST DISABLE_THREAD_SAFETY
                                  CHECK_SLOW_TESTS)
add_library(immer-dev INTERFACE)
target_include_directories(
  immer-dev SYSTEM INTERFACE ${Boost_INCLUDE_DIR} ${BoehmGC_INCLUDE_DIR}
                             ${CMAKE_CURRENT_SOURCE_DIR}/tools/include)
target_link_libraries(immer-dev INTERFACE immer ${BoehmGC_LIBRARIES}
                                          ${CMAKE_THREAD_LIBS_INIT})
target_compile_definitions(
  immer-dev
  INTERFACE -DIMMER_CXX_STANDARD=${CXX_STANDARD}
            -DIMMER_HAS_LIBGC=1
            -DIMMER_NO_FREE_LIST=${DISABLE_FREE_LIST}
            -DIMMER_NO_THREAD_SAFETY=${DISABLE_THREAD_SAFETY}
            -DIMMER_SLOW_TESTS=${CHECK_SLOW_TESTS}
            -DFMT_HEADER_ONLY=1)
if(ENABLE_COVERAGE)
  target_compile_options(immer-dev INTERFACE "--coverage")
  target_link_libraries(immer-dev INTERFACE "--coverage")
endif()

# Testing
# =======

if(immer_BUILD_TESTS
   OR immer_BUILD_EXAMPLES
   OR immer_BUILD_EXTRAS)
  add_custom_target(
    check
    COMMAND ${CMAKE_CTEST_COMMAND} --output-on-failure
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "Build and run all the tests and examples.")
endif()

if(ENABLE_VALGRIND)
  # these variables need to before including CTest, because including it
  # triggers the generation of the config DartConfiguration.tcl
  find_program(MEMORYCHECK_COMMAND valgrind)
  set(MEMORYCHECK_COMMAND_OPTIONS
      "${MEMORYCHECK_COMMAND_OPTIONS} --leak-check=full --error-exitcode=1 --suppressions=${PROJECT_SOURCE_DIR}/test/extra/persist/valgrind.supp"
  )
  set(MEMORYCHECK_SUPPRESSIONS_FILE
      "${PROJECT_SOURCE_DIR}/test/valgrind_suppress.txt"
      CACHE FILEPATH "")
  message(STATUS "Valgrind enabled for tests")
endif()

if(immer_BUILD_TESTS)
  enable_testing()

  add_subdirectory(test)
  add_subdirectory(benchmark)
endif()

if(immer_BUILD_EXAMPLES)
  add_subdirectory(example)
endif()

if(immer_BUILD_DOCS)
  add_subdirectory(doc)
endif()

if(immer_BUILD_EXTRAS)
  add_subdirectory(extra/fuzzer)
  add_subdirectory(extra/python)
  add_subdirectory(extra/guile)
endif()
