cmake_policy(VERSION 3.4)

# libbitcoin-client
#==============================================================================
project(libbitcoin-client
  VERSION 3.0.0
  LANGUAGES CXX C)
cmake_minimum_required(VERSION 3.4)

# Check for baseline language coverage in the compiler for the C++11 standard.
#------------------------------------------------------------------------------
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Process options.
#==============================================================================
# Implement --with-tests and declare WITH_TESTS.
#------------------------------------------------------------------------------
option(WITH_TESTS "Compile with unit tests." ON)

# Implement --with-examples and declare WITH_EXAMPLES.
#------------------------------------------------------------------------------
option(WITH_EXAMPLES "Compile with examples." ON)

# Inherit --enable-shared and define BOOST_TEST_DYN_LINK.
#------------------------------------------------------------------------------
option(ENABLE_SHARED "" OFF)

# Check dependencies.
#==============================================================================
# Require Boost of at least version 1.56.0 and output ${boost_CPPFLAGS/LDFLAGS}.
#------------------------------------------------------------------------------
if (NOT ENABLE_SHARED)
  set(Boost_USE_STATIC_LIBS ON)
endif()
find_package(
  Boost 1.56.0 REQUIRED
  COMPONENTS unit_test_framework)

# Require bitcoin of at least version 3.0.0 and output ${bitcoin_CPPFLAGS/LIBS/PKG}.
#------------------------------------------------------------------------------
if (NOT TARGET libbitcoin)
  find_package(libbitcoin 3.0.0 REQUIRED)
endif()

# Require bitcoin-protocol of at least version 3.0.0 and output ${bitcoin_protocol_CPPFLAGS/LIBS/PKG}.
#------------------------------------------------------------------------------
if (NOT TARGET libbitcoin-protocol)
  find_package(libbitcoin-protocol 3.0.0 REQUIRED)
endif()

# Set flags.
#==============================================================================
include(CheckCXXCompilerFlag)

function(_add_c_compile_flag _Flag _Var)
  check_cxx_compiler_flag(${_Flag} ${_Var})
  if (${_Var})
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${_Flag}" )
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${_Flag}" )
  endif()
endfunction()

function(_add_cxx_compile_flag _Flag _Var)
  check_cxx_compiler_flag(${_Flag} ${_Var})
  if (${_Var})
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${_Flag}" )
  endif()
endfunction()

function(_add_link_flag _Flag _Var)
  check_cxx_compiler_flag(${_Flag} ${_Var})
  if (${_Var})
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${_Flag}" )
    set(CMAKE_STATIC_LINKER_FLAGS "${CMAKE_STATIC_LINKER_FLAGS} ${_Flag}" )
  endif()
endfunction()

# Warn on all stuff.
#------------------------------------------------------------------------------
if (NOT MSVC)
  _add_c_compile_flag(-Wall _has_all_warning_flag)
else()
  _add_c_compile_flag(-W4 _has_all_warning_flag)
  add_definitions(-D_SCL_SECURE_NO_WARNINGS)
endif()

# Warn on extra stuff.
#------------------------------------------------------------------------------
if (NOT MSVC)
  _add_c_compile_flag(-Wextra _has_extra_warning_flag)
endif()

# Be really annoying.
#------------------------------------------------------------------------------
_add_c_compile_flag(-Wpedantic _has_pedantic_warning_flag)
if (_has_pedantic_warning_flag)
  _add_c_compile_flag(-pedantic _has_pedantic_flag)
endif()

# Conform to style.
#------------------------------------------------------------------------------
_add_cxx_compile_flag(-Wno-missing-braces _has_no_missing_braces_warning_flag)

# Conflict in stdlib under clang. Enabled in clang only.
#------------------------------------------------------------------------------
_add_cxx_compile_flag(-Wno-mismatched-tags _has_no_mismatched_tags_warning_flag)

# Clean up boost 1.55 headers. Enabled in gcc only.
#------------------------------------------------------------------------------
_add_c_compile_flag(-Wno-deprecated-declarations _has_no_deprecated_declarations_warning_flag)

# Protect stack.
#------------------------------------------------------------------------------
_add_link_flag(-fstack-protector _has_stack_protector_flag)

# Protect stack comprehensively.
#------------------------------------------------------------------------------
_add_link_flag(-fstack-protector-all _has_stack_protector_all_flag)

# Hide internal functions from external libs. Enabled in gcc only.
#------------------------------------------------------------------------------
_add_cxx_compile_flag(-fvisibility-hidden _has_visibility_hidden_flag)

# Hide inlines from external libs. Enabled in gcc only.
#------------------------------------------------------------------------------
_add_cxx_compile_flag(-fvisibility-inlines-hidden _has_visibility_inlines_hidden_flag)

# Build
#==============================================================================
# src/libbitcoin-client.la => ${libdir}
#------------------------------------------------------------------------------
set(MODE STATIC)
if (ENABLE_SHARED)
  set(MODE SHARED)
endif()

add_library(libbitcoin-client ${MODE}
  src/dealer.cpp
  src/obelisk_client.cpp
  src/proxy.cpp
  src/socket_stream.cpp)
target_include_directories(libbitcoin-client PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>)

target_compile_definitions(libbitcoin-client PUBLIC -DBCC_STATIC)

target_link_libraries(libbitcoin-client libbitcoin libbitcoin-protocol)

# Group sources
file(GLOB_RECURSE headers
  RELATIVE ${CMAKE_CURRENT_LIST_DIR}
  include/*.h include/*.hpp)
target_sources(libbitcoin-client PRIVATE "${headers}")

get_target_property(sources libbitcoin-client SOURCES)
foreach (source ${sources})
  get_filename_component(group ${source} DIRECTORY)
  if (MSVC)
    string(REPLACE "/" "\\" group ${group})
  endif()
  source_group("${group}" FILES ${source})
endforeach()

# local: test/libbitcoin_client_test
#------------------------------------------------------------------------------
if (WITH_TESTS)
  add_executable(libbitcoin_client_test
    test/main.cpp
    test/proxy.cpp)
  if (ENABLE_SHARED)
    target_compile_definitions(libbitcoin_client_test PRIVATE -DBOOST_TEST_DYN_LINK)
  endif()
  target_link_libraries(libbitcoin_client_test libbitcoin-client)

  target_include_directories(libbitcoin_client_test SYSTEM PUBLIC ${Boost_INCLUDE_DIR})
  target_link_libraries(libbitcoin_client_test
    ${Boost_UNIT_TEST_FRAMEWORK_LIBRARY})

  enable_testing()
  set(_libbitcoin_tests proxy_tests)
  foreach (_test_name ${_libbitcoin_tests})
    add_test(
      NAME test.${_test_name}
      COMMAND libbitcoin_client_test
        --run_test=${_test_name}
        --show_progress=no
        --detect_memory_leak=0
        --report_level=no
        --build_info=yes)
  endforeach()
endif()

# local: examples/console/console
#------------------------------------------------------------------------------
if (WITH_EXAMPLES)
  add_executable(console
    examples/console/client.cpp
    examples/console/client.hpp
    examples/console/connection.cpp
    examples/console/connection.hpp
    examples/console/main.cpp
    examples/console/read_line.cpp
    examples/console/read_line.hpp)
  target_link_libraries(console PUBLIC libbitcoin-client)
endif()

# local: examples/get_height/get_height
#------------------------------------------------------------------------------
if (WITH_EXAMPLES)
  add_executable(get_height
    examples/get_height/main.cpp)
  target_link_libraries(get_height PUBLIC libbitcoin-client)
endif()

# Install
#==============================================================================
install(TARGETS libbitcoin-client
  EXPORT libbitcoin-client
  ARCHIVE DESTINATION lib)

set(_libbitcoin_headers
  # include_bitcoin_HEADERS =
  bitcoin/client.hpp
  # include_bitcoin_client_HEADERS =
  bitcoin/client/dealer.hpp
  bitcoin/client/define.hpp
  bitcoin/client/obelisk_client.hpp
  bitcoin/client/proxy.hpp
  bitcoin/client/socket_stream.hpp
  bitcoin/client/stream.hpp
  bitcoin/client/version.hpp)
foreach (_header ${_libbitcoin_headers})
  get_filename_component(_directory "${_header}" DIRECTORY)
  install(FILES "include/${_header}" DESTINATION "include/${_directory}")
endforeach()

# Export
#==============================================================================
include(CMakePackageConfigHelpers)

export(EXPORT libbitcoin-client
  FILE "${CMAKE_CURRENT_BINARY_DIR}/libbitcoin-clientTargets.cmake")
export(PACKAGE libbitcoin-client)

if (UNIX)
  set(_config_package_location "lib/cmake")
elseif (WIN32)
  set(_config_package_location "cmake")
endif()

install(EXPORT libbitcoin-client
  FILE libbitcoin-clientTargets.cmake
  DESTINATION ${_config_package_location})

configure_file(libbitcoin-clientConfig.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/libbitcoin-clientConfig.cmake"
  @ONLY)

write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/libbitcoin-clientConfigVersion.cmake"
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY AnyNewerVersion)

install(
  FILES
    "${CMAKE_CURRENT_BINARY_DIR}/libbitcoin-clientConfig.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/libbitcoin-clientConfigVersion.cmake"
  DESTINATION ${_config_package_location})