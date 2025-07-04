cmake_minimum_required(VERSION 3.18.0)

project(ni_grpc_device_server
  LANGUAGES C CXX
  VERSION 2.13.0)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# Workaround for: https://bugs.chromium.org/p/boringssl/issues/detail?id=423
if (CMAKE_SYSTEM_PROCESSOR STREQUAL "AMD64")
  set(CMAKE_SYSTEM_PROCESSOR "amd64")
endif()

if(MSVC)
  add_definitions(-D_WIN32_WINNT=0x600)
  add_definitions(-DNOMINMAX) # Disable ancient min and max macros in Windows.h
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4267 /wd4244") # Disable possible loss of precision warnings
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /std:c17") # Needed for boringssl
endif()

#----------------------------------------------------------------------
# Use C++17 (needed for shared_mutex support on Linux)
#----------------------------------------------------------------------
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Enforce consistent runtime library settings in case of MSVC
if (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
  if (POLICY CMP0091)
    cmake_policy(SET CMP0091 NEW)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>" CACHE STRING "Use static runtime" FORCE)
    set(gRPC_MSVC_STATIC_RUNTIME ON CACHE BOOL "Use static runtime for gRPC" FORCE)
    set(libprotoc_MSVC_STATIC_RUNTIME ON CACHE BOOL "Use static runtime for libprotoc" FORCE)
    set(protobuf_MSVC_STATIC_RUNTIME ON CACHE BOOL "Use static runtime for protobuf" FORCE)
    set(ABSL_MSVC_STATIC_RUNTIME ON CACHE BOOL "Use static runtime for Abseil" FORCE)
    message("Setting policy CMP0091 and runtime library")
  else()
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MD")
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MDd")
    message("Using dynamic runtime library")
  endif()
endif()

#---
# Configuration options
#---
option(USE_NILRT_LEGACY_TOOLCHAIN "Enable to use tools and libraries from a NILRT compile toolchain." OFF)
option(USE_PYTHON_VIRTUALENV "Enable to use the automatically-generated python venv, local to this project source." ON)
option(USE_SUBMODULE_LIBS "Enable to link against the submodule libraries, rather than their native-OS equivalents." ON)
option(INCLUDE_SIDEBAND_RDMA "Include support for RDMA sideband transfers." OFF)
option(SIDEBAND_STATIC "Link grpc-sideband statically to current application." ON)

#----------------------------------------------------------------------
# Setup build dependencies, according to the toolchain options.
#----------------------------------------------------------------------
if(USE_SUBMODULE_LIBS)
  # The archetypical WIN32 build case
  # protobuf_INSTALL must be turned OFF whenever building it as a cmake subdir.
  set(protobuf_INSTALL OFF)

  add_subdirectory(third_party/grpc ${CMAKE_CURRENT_BINARY_DIR}/grpc)
  set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
  add_subdirectory(third_party/gtest ${CMAKE_CURRENT_BINARY_DIR}/gtest EXCLUDE_FROM_ALL)
  add_subdirectory(third_party/json ${CMAKE_CURRENT_BINARY_DIR}/json)
  add_subdirectory(third_party/utfcpp ${CMAKE_CURRENT_BINARY_DIR}/utfcpp)
  add_subdirectory(third_party/grpc-sideband ${CMAKE_CURRENT_BINARY_DIR}/grpc-sideband)

  set(_PROTOBUF_PROTOC $<TARGET_FILE:protobuf::protoc>)
  set(_REFLECTION grpc++_reflection)
  set(_GRPC grpc)
  set(_GRPC_CPP_PLUGIN_EXECUTABLE $<TARGET_FILE:grpc_cpp_plugin>)
  set(_GRPC_GRPCPP grpc++)
  set(_PROTOBUF_LIBPROTOBUF libprotobuf)
  set(_UTF8CPP utf8cpp)
  set(_GRPC_SIDEBAND ni_grpc_sideband)
else()
  find_program(_PROTOBUF_PROTOC protoc)
  find_program(_GRPC_CPP_PLUGIN_EXECUTABLE grpc_cpp_plugin)
  find_library(_GRPC_GPR gpr)
  find_library(_ABSEIL_SYNC absl_synchronization REQUIRED)

  if(USE_NILRT_LEGACY_TOOLCHAIN)
    # The archetypical NILRT SDK toolchain build case
    add_subdirectory(third_party/grpc ${CMAKE_CURRENT_BINARY_DIR}/grpc EXCLUDE_FROM_ALL)
    set(_REFLECTION grpc++_reflection)
    set(_GRPC_GRPCPP grpc++)
    set(_PROTOBUF_LIBPROTOBUF libprotobuf)
  else()
    # The archetypical linux build case (including OpenEmbedded)
    find_library(_GRPC_GRPCPP grpc++)
    find_library(_PROTOBUF_LIBPROTOBUF protobuf)
    find_library(_REFLECTION grpc++_reflection)
    find_library(_GRPC_SIDEBAND ni_grpc_sideband REQUIRED)

    find_package(gRPC REQUIRED)
    find_package(GTest REQUIRED)
    find_package(nlohmann_json REQUIRED)
    find_package(utf8cpp REQUIRED)

    set(_GRPC gRPC::grpc)
    set(_UTF8CPP utf8cpp::utf8cpp)
  endif()
endif()

# Python3 Virtual Environment
if(USE_PYTHON_VIRTUALENV)
  include(CreateVirtualEnvironment)
  CreateVirtualEnvironment(virtual_environment
    REQUIREMENTS_TXT
      ${CMAKE_SOURCE_DIR}/python_build_requirements.txt
    ENV_NAME
      venv
    OUT_PYTHON_EXE
      PYTHON_EXE
  )
else()
  find_package(Python3 REQUIRED)
  set(PYTHON_EXE ${Python3_EXECUTABLE})
endif()

enable_testing()

#----------------------------------------------------------------------
# Include generated *.pb.h files
#----------------------------------------------------------------------
set(proto_srcs_dir "${CMAKE_CURRENT_BINARY_DIR}/proto")
file(MAKE_DIRECTORY ${proto_srcs_dir})
include_directories(
  "${proto_srcs_dir}"
  "./generated"
  "./imports/include"
  "./source"
  "./third_party/grpc-sideband/src"
  "./third_party/grpc-sideband/moniker_src"
)
if(WIN32)
  link_directories("./imports/lib/win64")
endif()

#----------------------------------------------------------------------
# Get list of NI Driver APIs and directories to generate from
#----------------------------------------------------------------------
set(metadata_dir ${CMAKE_SOURCE_DIR}/source/codegen/metadata)
set(service_output_dir ${CMAKE_SOURCE_DIR}/generated)
set(codegen_dir ${CMAKE_SOURCE_DIR}/source/codegen)
set(custom_dir ${CMAKE_SOURCE_DIR}/source/custom)
set(nidrivers "nidaqmx" "nidcpower" "nidigitalpattern" "nidmm" "nifake" "nifake_extension" "nifake_non_ivi" "nifgen" "nifpga" "nimxlcterminaladaptor_restricted" "nirfmxbluetooth" "nirfmxcdma2k" "nirfmxgsm" "nirfmxinstr" "nirfmxinstr_restricted" "nirfmxlte" "nirfmxnr" "nirfmxpulse" "nirfmxspecan" "nirfmxspecan_restricted" "nirfmxtdscdma" "nirfmxwcdma" "nirfmxdemod" "nirfmxwlan" "nirfmxvna" "nirfmxwlan_restricted" "nirfsa" "nirfsg" "nirfsg_restricted" "niscope" "niscope_restricted" "niswitch" "nisync" "nitclk" "nixnet" "nixnetsocket" "visa")

#----------------------------------------------------------------------
# Create NI Driver APIs proto and server files
#----------------------------------------------------------------------
set(codegen_scripts
  "${codegen_dir}/client_helpers.py"
  "${codegen_dir}/common_helpers.py"
  "${codegen_dir}/generate_service.py"
  "${codegen_dir}/generate_shared_service_files.py"
  "${codegen_dir}/service_helpers.py"
  "${codegen_dir}/proto_helpers.py"
  "${codegen_dir}/metadata_mutation.py"
  "${codegen_dir}/metadata_validation.py"
  "${codegen_dir}/template_helpers.py"
  "${codegen_dir}/templates/client.cpp.mako"
  "${codegen_dir}/templates/client.h.mako"
  "${codegen_dir}/templates/client_helpers.mako"
  "${codegen_dir}/templates/library_interface.h.mako"
  "${codegen_dir}/templates/library.cpp.mako"
  "${codegen_dir}/templates/library.h.mako"
  "${codegen_dir}/templates/mock_library.h.mako"
  "${codegen_dir}/templates/proto.mako"
  "${codegen_dir}/templates/proto_helpers.mako"
  "${codegen_dir}/templates/register_all_services.h.mako"
  "${codegen_dir}/templates/register_all_services.cpp.mako"
  "${codegen_dir}/templates/service.cpp.mako"
  "${codegen_dir}/templates/service_helpers.mako"
  "${codegen_dir}/templates/service.h.mako"
  "${codegen_dir}/templates/service_registrar.h.mako"
  "${codegen_dir}/templates/service_registrar.cpp.mako"
  )

# Populated in the api loop below.
set(nidriver_service_library_hdrs "")
set(nidriver_service_srcs "")
set(nidriver_client_srcs "")

# We'll codegen all drivers, but some are not included in the server build.
set(nidrivers_to_build ${nidrivers})
# Fake drivers are not in the server and are added manually to the UnitTestRunner build.
list(FILTER nidrivers_to_build EXCLUDE REGEX "^nifake.*")
if (NOT WIN32)
  # None of the RFmx drivers support Linux and building against the headers fails.
  # Exclude them on Linux.
  list(FILTER nidrivers_to_build EXCLUDE REGEX "^nirfmx.*")
endif()

set(all_codegen_dependencies "")

if(USE_PYTHON_VIRTUALENV)
  list(APPEND all_codegen_dependencies virtual_environment)
endif()

foreach(api ${nidrivers})
  set(codegen_dependencies
    "${metadata_dir}/${api}/attributes.py"
    "${metadata_dir}/${api}/attributes_addon.py"
    "${metadata_dir}/${api}/config.py"
    "${metadata_dir}/${api}/config_addon.py"
    "${metadata_dir}/${api}/enums.py"
    "${metadata_dir}/${api}/enums_addon.py"
    "${metadata_dir}/${api}/functions.py"
    "${metadata_dir}/${api}/functions_addon.py"
    "${metadata_dir}/${api}/__init__.py")
  if (EXISTS "${metadata_dir}/${api}/custom_proto.mako")
    set(codegen_dependencies ${codegen_dependencies} "${metadata_dir}/${api}/custom_proto.mako")
  endif()
  set(all_codegen_dependencies
    ${all_codegen_dependencies}
    ${codegen_dependencies}
  )

  set(output_files
    ${service_output_dir}/${api}/${api}_client.cpp
    ${service_output_dir}/${api}/${api}_client.h
    ${service_output_dir}/${api}/${api}_compilation_test.cpp
    ${service_output_dir}/${api}/${api}_library_interface.h
    ${service_output_dir}/${api}/${api}_library.cpp
    ${service_output_dir}/${api}/${api}_library.h
    ${service_output_dir}/${api}/${api}_mock_library.h
    ${service_output_dir}/${api}/${api}.proto
    ${service_output_dir}/${api}/${api}_service.cpp
    ${service_output_dir}/${api}/${api}_service.h
    ${service_output_dir}/${api}/${api}_service_registrar.cpp
    ${service_output_dir}/${api}/${api}_service_registrar.h
  )
  set(nidriver_service_library_hdrs
    ${nidriver_service_library_hdrs}
    "${service_output_dir}/${api}/${api}_library.h"
    "${service_output_dir}/${api}/${api}_library_interface.h"
    "${service_output_dir}/${api}/${api}_mock_library.h")
  set(gen_command COMMAND ${PYTHON_EXE} ${codegen_dir}/generate_service.py ${metadata_dir}/${api}/ -o ${service_output_dir}/)
  if (api IN_LIST nidrivers_to_build)
    set(nidriver_service_srcs
      ${nidriver_service_srcs}
      "${service_output_dir}/${api}/${api}_service.cpp"
      "${service_output_dir}/${api}/${api}_service_registrar.cpp"
      "${service_output_dir}/${api}/${api}_library.cpp"
      "${custom_dir}/${api}_service.custom.cpp")
    set(nidriver_client_srcs
      ${nidriver_client_srcs}
      "${service_output_dir}/${api}/${api}_client.cpp")
  endif()
  set(proto_dependencies ${codegen_dependencies} ${codegen_scripts})
  if (USE_PYTHON_VIRTUALENV)
    list(APPEND proto_dependencies virtual_environment)
  endif()
  add_custom_command(OUTPUT ${output_files}
    ${gen_command}
    COMMENT "Generating proto file and service for ${api}"
    DEPENDS ${proto_dependencies})
  add_custom_target(generate_proto_and_service_target_for_${api}
    DEPENDS ${output_files})
endforeach()

add_custom_command(
  OUTPUT
    ${service_output_dir}/register_all_services.cpp
    ${service_output_dir}/register_all_services.h
  COMMAND
    ${PYTHON_EXE} ${codegen_dir}/generate_shared_service_files.py ${metadata_dir}/ -o ${service_output_dir}/
  COMMENT
    "Generating shared service files"
  DEPENDS
    ${all_codegen_dependencies}
    ${codegen_scripts}
)

set(nidriver_service_srcs
    ${nidriver_service_srcs}
    ${service_output_dir}/register_all_services.cpp
)

#----------------------------------------------------------------------
# Proto file
#----------------------------------------------------------------------
get_filename_component(session_proto "imports/protobuf/session.proto" ABSOLUTE)
get_filename_component(session_utilities_proto "source/protobuf/session_utilities.proto" ABSOLUTE)
get_filename_component(nidevice_proto "imports/protobuf/nidevice.proto" ABSOLUTE)
get_filename_component(deviceid_restricted_proto "source/protobuf_restricted/deviceid_restricted.proto" ABSOLUTE)
get_filename_component(debugsessionproperties_restricted_proto "source/protobuf_restricted/debugsessionproperties_restricted.proto" ABSOLUTE)
get_filename_component(calibrationoperations_restricted_proto "source/protobuf_restricted/calibrationoperations_restricted.proto" ABSOLUTE)
get_filename_component(data_moniker_proto "imports/protobuf/data_moniker.proto" ABSOLUTE)
get_filename_component(session_proto_path "${session_proto}" PATH)

#----------------------------------------------------------------------
# Generate sources from proto files
# Usage: GenerateGrpcSources(PROTO <path> OUTPUT <path>...)
#----------------------------------------------------------------------
function(GenerateGrpcSources)
  set(oneValueArgs PROTO)
  set(multiValueArgs OUTPUT)
  cmake_parse_arguments(GENERATE_ARGS "" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})
  set(output_files "${GENERATE_ARGS_OUTPUT}")
  set(proto_file "${GENERATE_ARGS_PROTO}")
  if(USE_SUBMODULE_LIBS)
    set(protobuf_includes_arg -I ${CMAKE_SOURCE_DIR}/third_party/grpc/third_party/protobuf/src/)
  endif()
  get_filename_component(proto_name "${proto_file}" NAME)
  get_filename_component(proto_path "${proto_file}" PATH)
  # Asssumption: all outputs are in the same directory: use the zero-th.
  list(GET output_files 0 proto_srcs)
  get_filename_component(proto_out_path "${proto_srcs}" PATH)
  string(FIND "${proto_name}" "nixnetsocket" generating_nixnetsocket)
  # If generating nixnetsocket on Windows apply the winsock2 compatibility fix after codegen.
  # Else do normal codegen.
  if(MSVC AND (NOT generating_nixnetsocket EQUAL -1))
    add_custom_command(
      OUTPUT ${output_files}
      COMMAND ${_PROTOBUF_PROTOC}
      ARGS --grpc_out ${proto_out_path}
        --cpp_out ${proto_out_path}
        -I ${proto_path}
        ${protobuf_includes_arg}
        -I ${CMAKE_SOURCE_DIR}/imports/protobuf
        --plugin=protoc-gen-grpc=${_GRPC_CPP_PLUGIN_EXECUTABLE}
        ${proto_file}
      COMMAND ${CMAKE_COMMAND} -E echo \#include \"custom/xnetsocket_winsock2_compatibility.h\" > ${proto_srcs_dir}/nixnetsocket.pb.h.tmp
      COMMAND ${CMAKE_COMMAND} -E cat ${proto_srcs_dir}/nixnetsocket.pb.h >> ${proto_srcs_dir}/nixnetsocket.pb.h.tmp
      COMMAND ${CMAKE_COMMAND} -E cat ${proto_srcs_dir}/nixnetsocket.pb.h.tmp > ${proto_srcs_dir}/nixnetsocket.pb.h
      DEPENDS "${proto_file}" "${session_proto}"
      VERBATIM)
  else()
    add_custom_command(
      OUTPUT ${output_files}
      COMMAND ${_PROTOBUF_PROTOC}
      ARGS --grpc_out ${proto_out_path}
        --cpp_out ${proto_out_path}
        -I ${proto_path}
        ${protobuf_includes_arg}
        -I ${CMAKE_SOURCE_DIR}/imports/protobuf
        --plugin=protoc-gen-grpc=${_GRPC_CPP_PLUGIN_EXECUTABLE}
        ${proto_file}
      DEPENDS "${proto_file}" "${session_proto}"
      VERBATIM)
  endif()
endfunction()

set(session_proto_srcs "${proto_srcs_dir}/session.pb.cc")
set(session_proto_hdrs "${proto_srcs_dir}/session.pb.h")
set(session_grpc_srcs "${proto_srcs_dir}/session.grpc.pb.cc")
set(session_grpc_hdrs "${proto_srcs_dir}/session.grpc.pb.h")
set(session_utilities_proto_srcs "${proto_srcs_dir}/session_utilities.pb.cc")
set(session_utilities_proto_hdrs "${proto_srcs_dir}/session_utilities.pb.h")
set(session_utilities_grpc_srcs "${proto_srcs_dir}/session_utilities.grpc.pb.cc")
set(session_utilities_grpc_hdrs "${proto_srcs_dir}/session_utilities.grpc.pb.h")
set(nidevice_proto_srcs "${proto_srcs_dir}/nidevice.pb.cc")
set(nidevice_proto_hdrs "${proto_srcs_dir}/nidevice.pb.h")
set(deviceid_restricted_proto_srcs "${proto_srcs_dir}/deviceid_restricted.pb.cc")
set(deviceid_restricted_proto_hdrs "${proto_srcs_dir}/deviceid_restricted.pb.h")
set(deviceid_restricted_grpc_srcs "${proto_srcs_dir}/deviceid_restricted.grpc.pb.cc")
set(deviceid_restricted_grpc_hdrs "${proto_srcs_dir}/deviceid_restricted.grpc.pb.h")
set(debugsessionproperties_restricted_proto_srcs "${proto_srcs_dir}/debugsessionproperties_restricted.pb.cc")
set(debugsessionproperties_restricted_proto_hdrs "${proto_srcs_dir}/debugsessionproperties_restricted.pb.h")
set(debugsessionproperties_restricted_grpc_srcs "${proto_srcs_dir}/debugsessionproperties_restricted.grpc.pb.cc")
set(debugsessionproperties_restricted_grpc_hdrs "${proto_srcs_dir}/debugsessionproperties_restricted.grpc.pb.h")
set(calibrationoperations_restricted_proto_srcs "${proto_srcs_dir}/calibrationoperations_restricted.pb.cc")
set(calibrationoperations_restricted_proto_hdrs "${proto_srcs_dir}/calibrationoperations_restricted.pb.h")
set(calibrationoperations_restricted_grpc_srcs "${proto_srcs_dir}/calibrationoperations_restricted.grpc.pb.cc")
set(calibrationoperations_restricted_grpc_hdrs "${proto_srcs_dir}/calibrationoperations_restricted.grpc.pb.h")
set(data_moniker_proto_srcs "${proto_srcs_dir}/data_moniker.pb.cc")
set(data_moniker_proto_hdrs "${proto_srcs_dir}/data_moniker.pb.h")
set(data_moniker_grpc_srcs "${proto_srcs_dir}/data_moniker.grpc.pb.cc")
set(data_moniker_grpc_hdrs "${proto_srcs_dir}/data_moniker.grpc.pb.h")

GenerateGrpcSources(
  PROTO
    ${session_proto}
  OUTPUT
    "${session_proto_srcs}"
    "${session_proto_hdrs}"
    "${session_grpc_srcs}"
    "${session_grpc_hdrs}"
)

GenerateGrpcSources(
  PROTO
    ${session_utilities_proto}
  OUTPUT
    "${session_utilities_proto_srcs}"
    "${session_utilities_proto_hdrs}"
    "${session_utilities_grpc_srcs}"
    "${session_utilities_grpc_hdrs}"
)

GenerateGrpcSources(
  PROTO
    ${nidevice_proto}
  OUTPUT
    "${nidevice_proto_srcs}"
    "${nidevice_proto_hdrs}"
)

GenerateGrpcSources(
  PROTO
    ${deviceid_restricted_proto}
  OUTPUT
    "${deviceid_restricted_proto_srcs}"
    "${deviceid_restricted_proto_hdrs}"
    "${deviceid_restricted_grpc_srcs}"
    "${deviceid_restricted_grpc_hdrs}"
)

GenerateGrpcSources(
  PROTO
    ${debugsessionproperties_restricted_proto}
  OUTPUT
    "${debugsessionproperties_restricted_proto_srcs}"
    "${debugsessionproperties_restricted_proto_hdrs}"
    "${debugsessionproperties_restricted_grpc_srcs}"
    "${debugsessionproperties_restricted_grpc_hdrs}"
)

GenerateGrpcSources(
  PROTO
    ${calibrationoperations_restricted_proto}
  OUTPUT
    "${calibrationoperations_restricted_proto_srcs}"
    "${calibrationoperations_restricted_proto_hdrs}"
    "${calibrationoperations_restricted_grpc_srcs}"
    "${calibrationoperations_restricted_grpc_hdrs}"
)

GenerateGrpcSources(
  PROTO
    ${data_moniker_proto}
  OUTPUT
    "${data_moniker_proto_srcs}"
    "${data_moniker_proto_hdrs}"
    "${data_moniker_grpc_srcs}"
    "${data_moniker_grpc_hdrs}"
)

set(nidriver_service_library_hdrs
  ${nidriver_service_library_hdrs}
  "${session_proto_hdrs}"
  "${session_grpc_hdrs}"
  "${session_utilities_proto_hdrs}"
  "${session_utilities_grpc_hdrs}"
  "${nidevice_proto_hdrs}"
  "${deviceid_restricted_proto_hdrs}"
  "${deviceid_restricted_grpc_hdrs}"
  "${debugsessionproperties_restricted_proto_hdrs}"
  "${debugsessionproperties_restricted_grpc_hdrs}"
  "${calibrationoperations_restricted_proto_hdrs}"
  "${calibrationoperations_restricted_grpc_hdrs}"
)

foreach(api ${nidrivers})
  GenerateGrpcSources(
    PROTO
      ${service_output_dir}/${api}/${api}.proto
    OUTPUT
      "${proto_srcs_dir}/${api}.pb.cc"
      "${proto_srcs_dir}/${api}.pb.h"
      "${proto_srcs_dir}/${api}.grpc.pb.cc"
      "${proto_srcs_dir}/${api}.grpc.pb.h"
  )
  set(nidriver_service_library_hdrs
    ${nidriver_service_library_hdrs}
    "${proto_srcs_dir}/${api}.pb.h"
    "${proto_srcs_dir}/${api}.grpc.pb.h")
  if(api IN_LIST nidrivers_to_build)
    set(nidriver_service_srcs
      ${nidriver_service_srcs}
      "${proto_srcs_dir}/${api}.pb.cc"
      "${proto_srcs_dir}/${api}.grpc.pb.cc")
  endif()
endforeach()

add_executable(ni_grpc_device_server
   "imports/include/nierr_Status.cpp"
   "source/server/calibration_operations_restricted_service_registrar.cpp"
   "source/server/calibration_operations_restricted_service.cpp"
   "source/server/core_server.cpp"
   "source/server/core_services_registrar.cpp"
   "source/server/data_moniker_service.cpp"
   "source/server/debug_session_properties_restricted_service_registrar.cpp"
   "source/server/debug_session_properties_restricted_service.cpp"
   "source/server/device_enumerator.cpp"
   "source/server/feature_toggles.cpp"
   "source/server/logging.cpp"
   "source/server/semaphore.cpp"
   "source/server/server_configuration_parser.cpp"
   "source/server/server_security_configuration.cpp"
   "source/server/session_repository.cpp"
   "source/server/session_utilities_service.cpp"
   "source/server/session_utilities_service_registrar.cpp"
   "source/server/shared_library.cpp"
   "source/server/software_enumerator.cpp"
   "source/server/syscfg_library.cpp"
   "source/server/syscfg_resource_accessor.cpp"
   "source/server/syscfg_session_handler.cpp"
   ${nidevice_proto_srcs}
   ${session_proto_srcs}
   ${session_grpc_srcs}
   ${session_utilities_proto_srcs}
   ${session_utilities_grpc_srcs}
   ${deviceid_restricted_proto_srcs}
   ${deviceid_restricted_grpc_srcs}
   ${debugsessionproperties_restricted_proto_srcs}
   ${debugsessionproperties_restricted_grpc_srcs}
   ${calibrationoperations_restricted_proto_srcs}
   ${calibrationoperations_restricted_grpc_srcs}
   ${data_moniker_proto_srcs}
   ${data_moniker_grpc_srcs}
   ${nidriver_service_srcs})

# Enable warnings only on source that we own, not generated code or dependencies
file(GLOB_RECURSE SOURCE_TO_ENABLE_WARNINGS "source/*.cpp")
if(WIN32)
  set_source_files_properties(
    ${SOURCE_TO_ENABLE_WARNINGS}
    APPEND
    PROPERTIES
    COMPILE_OPTIONS "/WX"
  )
else()
  set_source_files_properties(
    ${SOURCE_TO_ENABLE_WARNINGS}
    APPEND
    PROPERTIES
    COMPILE_OPTIONS "-Werror"
  )
endif()

if(WIN32)
  set(version_generated_dir "${CMAKE_CURRENT_BINARY_DIR}/version")
  file(MAKE_DIRECTORY ${version_generated_dir})

  # Generate version resource file for Windows
  configure_file(
    source/server/version.rc.in
    ${version_generated_dir}/version.rc
    @ONLY)

  target_sources(ni_grpc_device_server
    PRIVATE "${version_generated_dir}/version.rc"
  )
endif()

execute_process(
  COMMAND git rev-parse --abbrev-ref HEAD
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
  OUTPUT_VARIABLE GIT_BRANCH
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

# Generate version header file
# This file requires product name and version info that mako does not know, but CMake does
configure_file(
  source/server/version.h.in
  ${service_output_dir}/version.h
  @ONLY)

target_sources(ni_grpc_device_server
  PUBLIC "${service_output_dir}/version.h"
)


if(CMAKE_SYSTEM_NAME STREQUAL Linux)
  target_sources(ni_grpc_device_server
    PRIVATE "source/server/linux/syslog_logging.cpp"
    PRIVATE "source/server/linux/daemonize.cpp")
endif()
if(CMAKE_SYSTEM_NAME STREQUAL Windows)
  target_sources(ni_grpc_device_server
    PRIVATE "source/server/windows/console_ctrl_handler.cpp")
endif()

set(server_lib_deps
  ${_ABSEIL_SYNC}
  ${_GRPC_GPR}
  ${_GRPC_GRPCPP}
  ${_GRPC}
  ${_PROTOBUF_LIBPROTOBUF}
  ${_REFLECTION}
  ${_UTF8CPP}
  ${CMAKE_DL_LIBS}
  ${_GRPC_SIDEBAND}
  nlohmann_json::nlohmann_json
)

target_link_libraries(ni_grpc_device_server
  ${server_lib_deps}
)

set_target_properties(ni_grpc_device_server PROPERTIES BUILD_WITH_INSTALL_RPATH TRUE)

#----------------------------------------------------------------------
# Copy server_config.json to binary output directory
#----------------------------------------------------------------------
add_custom_command(
   TARGET ni_grpc_device_server POST_BUILD
   COMMAND  ${CMAKE_COMMAND} -E copy_if_different
            ${CMAKE_SOURCE_DIR}/source/config/localhost_config.json
            $<TARGET_FILE_DIR:ni_grpc_device_server>/server_config.json)

#----------------------------------------------------------------------
# Generate server_capabilities.json to binary output directory
#----------------------------------------------------------------------
add_custom_command(
   TARGET ni_grpc_device_server POST_BUILD
   COMMAND  ${PYTHON_EXE} ${codegen_dir}/generate_server_capabilities.py ${metadata_dir}/
            -o $<TARGET_FILE_DIR:ni_grpc_device_server>/)

# Link test executable against gtest
add_executable(IntegrationTestsRunner
    "imports/include/nierr_Status.cpp"
    "source/tests/utilities/run_all_tests.cpp"
    "source/tests/integration/ni_fake_service_tests_endtoend.cpp"
    "source/tests/integration/ni_fake_non_ivi_service_tests_endtoend.cpp"
    "source/tests/integration/session_utilities_service_tests.cpp"
    "source/tests/integration/session_utilities_service_tests_endtoend.cpp"
    "source/tests/integration/visa_resource_manager_tests.cpp"
    "source/tests/integration/ni_fake_daqmx_streaming_tests.cpp"
    "source/tests/integration/ni_fake_fpga_streaming_tests.cpp"
    "source/server/calibration_operations_restricted_service_registrar.cpp"
    "source/server/calibration_operations_restricted_service.cpp"
    "source/server/core_services_registrar.cpp"
    "source/server/data_moniker_service.cpp"
    "source/server/debug_session_properties_restricted_service_registrar.cpp"
    "source/server/debug_session_properties_restricted_service.cpp"
    "source/server/device_enumerator.cpp"
    "source/server/feature_toggles.cpp"
    "source/server/semaphore.cpp"
    "source/server/session_repository.cpp"
    "source/server/session_utilities_service.cpp"
    "source/server/session_utilities_service_registrar.cpp"
    "source/server/shared_library.cpp"
    "source/server/software_enumerator.cpp"
    "source/server/syscfg_library.cpp"
    "source/server/syscfg_resource_accessor.cpp"
    "source/server/syscfg_session_handler.cpp"
    ${nidevice_proto_srcs}
    ${session_proto_srcs}
    ${session_grpc_srcs}
    ${session_utilities_proto_srcs}
    ${session_utilities_grpc_srcs}
    ${deviceid_restricted_proto_srcs}
    ${deviceid_restricted_grpc_srcs}
    ${debugsessionproperties_restricted_proto_srcs}
    ${debugsessionproperties_restricted_grpc_srcs}
    ${calibrationoperations_restricted_proto_srcs}
    ${calibrationoperations_restricted_grpc_srcs}
    ${data_moniker_proto_srcs}
    ${data_moniker_grpc_srcs}
    ${nidriver_service_srcs}
    "${proto_srcs_dir}/nifake.pb.cc"
    "${proto_srcs_dir}/nifake.grpc.pb.cc"
    "${proto_srcs_dir}/nifake_non_ivi.pb.cc"
    "${proto_srcs_dir}/nifake_non_ivi.grpc.pb.cc"
    "${service_output_dir}/nidaqmx/nidaqmx_client.cpp"
    "${service_output_dir}/nifake/nifake_client.cpp"
    "${service_output_dir}/nifake/nifake_service.cpp"
    "${service_output_dir}/nifpga/nifpga_client.cpp"
    "${service_output_dir}/nifake_non_ivi/nifake_non_ivi_client.cpp"
    "${service_output_dir}/nifake_non_ivi/nifake_non_ivi_service.cpp"
    "${custom_dir}/nifake_non_ivi_service.custom.cpp"
    "${custom_dir}/nifake_service.custom.cpp"
)

    set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
    set(THREADS_PREFER_PTHREAD_FLAG TRUE)
    find_package(Threads REQUIRED)

target_link_libraries(IntegrationTestsRunner
    ${server_lib_deps}
    gmock
    grpc
    gtest
    Threads::Threads
)

# Ignore the use of deprecated functions in test code
target_compile_definitions(IntegrationTestsRunner
    PRIVATE _CRT_SECURE_NO_WARNINGS _WINSOCK_DEPRECATED_NO_WARNINGS)

add_library(TestApi SHARED
    "source/tests/utilities/test_api.cpp")
add_compile_definitions(TestApi TEST_API_BUILDING)

# Ignore the use of deprecated functions in test code
target_compile_definitions(TestApi
    PRIVATE _CRT_SECURE_NO_WARNINGS)

add_executable(UnitTestsRunner
    "imports/include/nierr_Status.cpp"
    "source/tests/utilities/run_all_tests.cpp"
    "source/tests/unit/callback_router_tests.cpp"
    "source/tests/unit/converters_tests.cpp"
    "source/tests/unit/library_set_runtime_environment_tests.cpp"
    "source/tests/unit/session_resource_repository_tests.cpp"
    "source/tests/unit/session_repository_tests.cpp"
    "source/tests/unit/calibration_operations_tests.cpp"
    "source/tests/unit/debug_session_properties_tests.cpp"
    "source/tests/unit/device_enumerator_tests.cpp"
    "source/tests/unit/software_enumerator_tests.cpp"
    "source/tests/unit/server_configuration_parser_tests.cpp"
    "source/tests/unit/server_security_configuration_tests.cpp"
    "source/tests/unit/ni_fake_non_ivi_service_tests.cpp"
    "source/tests/unit/ni_fake_service_tests.cpp"
    "source/tests/unit/nimxlc_terminal_adaptor_converters_tests.cpp"
    "source/tests/unit/shared_library_tests.cpp"
    "source/tests/unit/syscfg_library_tests.cpp"
    "source/tests/unit/syscfg_resource_accessor_tests.cpp"
    "source/tests/unit/xnet_converters_tests.cpp"
    "source/tests/unit/xnet_socket_converters_tests.cpp"
    "source/server/calibration_operations_restricted_service.cpp"
    "source/server/debug_session_properties_restricted_service.cpp"
    "source/server/device_enumerator.cpp"
    "source/server/feature_toggles.cpp"
    "source/server/semaphore.cpp"
    "source/server/server_configuration_parser.cpp"
    "source/server/server_security_configuration.cpp"
    "source/server/session_repository.cpp"
    "source/server/shared_library.cpp"
    "source/server/software_enumerator.cpp"
    "source/server/syscfg_library.cpp"
    "source/server/syscfg_resource_accessor.cpp"
    "source/server/syscfg_session_handler.cpp"
    ${nidevice_proto_srcs}
    ${session_proto_srcs}
    ${session_grpc_srcs}
    ${session_utilities_proto_srcs}
    ${session_utilities_grpc_srcs}
    ${deviceid_restricted_proto_srcs}
    ${deviceid_restricted_grpc_srcs}
    ${debugsessionproperties_restricted_proto_srcs}
    ${debugsessionproperties_restricted_grpc_srcs}
    ${calibrationoperations_restricted_proto_srcs}
    ${calibrationoperations_restricted_grpc_srcs}
    ${data_moniker_proto_srcs}
    ${data_moniker_grpc_srcs}
    "${proto_srcs_dir}/nifake.pb.cc"
    "${proto_srcs_dir}/nifake.grpc.pb.cc"
    "${proto_srcs_dir}/nifake_extension.pb.cc"
    "${proto_srcs_dir}/nifake_extension.grpc.pb.cc"
    "${proto_srcs_dir}/nifake_non_ivi.pb.cc"
    "${proto_srcs_dir}/nifake_non_ivi.grpc.pb.cc"
    "${proto_srcs_dir}/nimxlcterminaladaptor_restricted.pb.cc"
    "${proto_srcs_dir}/nixnet.grpc.pb.cc"
    "${proto_srcs_dir}/nixnet.pb.cc"
    "${proto_srcs_dir}/nixnetsocket.pb.cc"
    "${service_output_dir}/nifake/nifake_service.cpp"
    "${service_output_dir}/nifake_extension/nifake_extension_service.cpp"
    "${service_output_dir}/nifake_non_ivi/nifake_non_ivi_service.cpp"
    "${service_output_dir}/niswitch/niswitch_library.cpp"
    "${service_output_dir}/nixnet/nixnet_service.cpp"
    "${custom_dir}/nifake_service.custom.cpp"
    "${custom_dir}/nifake_extension_service.custom.cpp"
    "${custom_dir}/nifake_non_ivi_service.custom.cpp"
    "${custom_dir}/nimxlcterminaladaptor_restricted_service.custom.cpp"
    "${custom_dir}/nixnet_service.custom.cpp"
)

# ni_fake_service_tests.cpp and several DAQ cpp files exceed the MSVC limit for the number of sections
# in an obj file defined by PE-COFF. This line disables the limit.
if(MSVC)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /bigobj /D_SILENCE_CXX17_ITERATOR_BASE_CLASS_DEPRECATION_WARNING")
endif(MSVC)

# Ignore the use of deprecated functions in test code
target_compile_definitions(UnitTestsRunner
    PRIVATE _CRT_SECURE_NO_WARNINGS _WINSOCK_DEPRECATED_NO_WARNINGS)

target_include_directories(UnitTestsRunner
    PRIVATE "${service_output_dir}"
    PRIVATE "${service_output_dir}/nifake"
    PRIVATE "${service_output_dir}/nifake_extension"
    PRIVATE "${service_output_dir}/nifake_non_ivi"
    PRIVATE "source/server")

target_link_libraries(UnitTestsRunner
    ${CMAKE_DL_LIBS}
    ${server_lib_deps}
    gmock
    grpc
    gtest
    Threads::Threads
)

#----------------------------------------------------------------------
# Copy test asset certificates to binary output certs sub-directory
#----------------------------------------------------------------------
add_custom_command(
  TARGET UnitTestsRunner POST_BUILD
  COMMAND  ${CMAKE_COMMAND} -E copy_directory
           ${CMAKE_SOURCE_DIR}/source/tests/assets/
           $<TARGET_FILE_DIR:UnitTestsRunner>/)

foreach(api ${nidrivers_to_build})
  set(compilation_test_sources
    ${compilation_test_sources}
    "${service_output_dir}/${api}/${api}_compilation_test.cpp")
endforeach()
add_library(CompilationTests STATIC ${compilation_test_sources})

target_link_libraries(CompilationTests
    ${_GRPC_GRPCPP}
    ${_UTF8CPP}
)
add_custom_target(generated_nidriver_service_library_hdrs DEPENDS ${nidriver_service_library_hdrs})
add_dependencies(CompilationTests generated_nidriver_service_library_hdrs)

set(system_test_runner_sources
  "imports/include/nierr_Status.cpp"
  "source/tests/utilities/run_all_tests.cpp"
  "source/server/core_services_registrar.cpp"
  "source/tests/system/device_server.cpp"
  "source/tests/system/enumerate_devices.cpp"
  "source/tests/system/enumerate_software.cpp"
  "source/tests/system/session_utilities_service_tests.cpp"
  "source/tests/system/nidaqmx_driver_api_tests.cpp"
  "source/tests/system/nidaqmx_session_tests.cpp"
  "source/tests/system/nidcpower_driver_api_tests.cpp"
  "source/tests/system/nidcpower_session_tests.cpp"
  "source/tests/system/nidigital_driver_api_tests.cpp"
  "source/tests/system/nidigital_session_tests.cpp"
  "source/tests/system/nidmm_driver_api_tests.cpp"
  "source/tests/system/nidmm_session_tests.cpp"
  "source/tests/system/nifgen_driver_api_tests.cpp"
  "source/tests/system/nifgen_session_tests.cpp"
  "source/tests/system/nifpga_session_tests.cpp"
  "source/tests/system/nirfsa_driver_api_tests.cpp"
  "source/tests/system/nirfsg_driver_api_tests.cpp"
  "source/tests/system/nirfsg_restricted_driver_api_tests.cpp"
  "source/tests/system/nirfsg_session_tests.cpp"
  "source/tests/system/niscope_driver_api_tests.cpp"
  "source/tests/system/niscope_restricted_driver_api_tests.cpp"
  "source/tests/system/niscope_session_tests.cpp"
  "source/tests/system/niswitch_driver_api_tests.cpp"
  "source/tests/system/niswitch_session_tests.cpp"
  "source/tests/system/nisync_driver_api_tests.cpp"
  "source/tests/system/nisync_session_tests.cpp"
  "source/tests/system/nitclk_driver_api_tests.cpp"
  "source/tests/system/nixnet_lin_driver_api_tests.cpp"
  "source/tests/system/nixnet_can_driver_api_tests.cpp"
  "source/tests/system/nixnet_ethernet_driver_api_tests.cpp"
  "source/tests/system/nixnetsocket_driver_api_tests.cpp"
  "source/tests/system/visa_driver_api_tests.cpp"
  "source/tests/system/visa_session_tests.cpp"
  "source/server/calibration_operations_restricted_service_registrar.cpp"
  "source/server/calibration_operations_restricted_service.cpp"
  "source/server/data_moniker_service.cpp"
  "source/server/debug_session_properties_restricted_service_registrar.cpp"
  "source/server/debug_session_properties_restricted_service.cpp"
  "source/server/device_enumerator.cpp"
  "source/server/feature_toggles.cpp"
  "source/server/semaphore.cpp"
  "source/server/session_repository.cpp"
  "source/server/session_utilities_service.cpp"
  "source/server/session_utilities_service_registrar.cpp"
  "source/server/shared_library.cpp"
  "source/server/software_enumerator.cpp"
  "source/server/syscfg_library.cpp"
  "source/server/syscfg_resource_accessor.cpp"
  "source/server/syscfg_session_handler.cpp"
  ${nidevice_proto_srcs}
  ${session_proto_srcs}
  ${session_grpc_srcs}
  ${session_utilities_proto_srcs}
  ${session_utilities_grpc_srcs}
  ${deviceid_restricted_proto_srcs}
  ${deviceid_restricted_grpc_srcs}
  ${debugsessionproperties_restricted_proto_srcs}
  ${debugsessionproperties_restricted_grpc_srcs}
  ${calibrationoperations_restricted_proto_srcs}
  ${calibrationoperations_restricted_grpc_srcs}
  ${data_moniker_proto_srcs}
  ${data_moniker_grpc_srcs}
  ${nidriver_service_srcs}
  ${nidriver_client_srcs}
)

if(WIN32)
  list(
    APPEND
      system_test_runner_sources
        "source/tests/system/nimxlcterminaladaptor_restricted_driver_api_tests.cpp"
        "source/tests/system/nirfmxbluetooth_driver_api_tests.cpp"
        "source/tests/system/nirfmxbluetooth_session_tests.cpp"
        "source/tests/system/nirfmxcdma2k_driver_api_tests.cpp"
        "source/tests/system/nirfmxcdma2k_session_tests.cpp"
        "source/tests/system/nirfmxgsm_driver_api_tests.cpp"
        "source/tests/system/nirfmxgsm_session_tests.cpp"
        "source/tests/system/nirfmxinstr_driver_api_tests.cpp"
        "source/tests/system/nirfmxinstr_restricted_driver_api_tests.cpp"
        "source/tests/system/nirfmxlte_driver_api_tests.cpp"
        "source/tests/system/nirfmxlte_session_tests.cpp"
        "source/tests/system/nirfmxnr_driver_api_tests.cpp"
        "source/tests/system/nirfmxnr_session_tests.cpp"
        "source/tests/system/nirfmxpulse_driver_api_tests.cpp"
        "source/tests/system/nirfmxpulse_session_tests.cpp"
        "source/tests/system/nirfmxspecan_driver_api_tests.cpp"
        "source/tests/system/nirfmxspecan_restricted_driver_api_tests.cpp"
        "source/tests/system/nirfmxspecan_session_tests.cpp"
        "source/tests/system/nirfmxtdscdma_driver_api_tests.cpp"
        "source/tests/system/nirfmxtdscdma_session_tests.cpp"
        "source/tests/system/nirfmxwcdma_driver_api_tests.cpp"
        "source/tests/system/nirfmxwcdma_session_tests.cpp"
        "source/tests/system/nirfmxdemod_driver_api_tests.cpp"
        "source/tests/system/nirfmxdemod_session_tests.cpp"
        "source/tests/system/nirfmxvna_driver_api_tests.cpp"
        "source/tests/system/nirfmxvna_session_tests.cpp"
        "source/tests/system/nirfmxwlan_driver_api_tests.cpp"
        "source/tests/system/nirfmxwlan_restricted_driver_api_tests.cpp"
        "source/tests/system/nirfmxwlan_session_tests.cpp"
  )
endif()

add_executable(SystemTestsRunner ${system_test_runner_sources})

target_link_libraries(SystemTestsRunner
    ${CMAKE_DL_LIBS}
    ${server_lib_deps}
    gmock
    gtest
)

# Ignore the use of deprecated functions in test code
target_compile_definitions(SystemTestsRunner
    PRIVATE _CRT_SECURE_NO_WARNINGS _WINSOCK_DEPRECATED_NO_WARNINGS)

add_custom_command(
  TARGET SystemTestsRunner POST_BUILD
  COMMAND  ${CMAKE_COMMAND} -E copy_directory
            ${CMAKE_SOURCE_DIR}/source/tests/assets/data/
            $<TARGET_FILE_DIR:SystemTestsRunner>/)

if(USE_NILRT_LEGACY_TOOLCHAIN)
    target_link_libraries(SystemTestsRunner stdc++fs)
    target_compile_definitions(SystemTestsRunner PRIVATE FS_EXPERIMENTAL)
endif()


# Hook up different google test runners to CTest
# add_test( NAME UnitTests COMMAND UnitTestsRunner )
add_test( NAME UnitTests COMMAND UnitTestsRunner )
add_test( NAME IntegrationTests COMMAND IntegrationTestsRunner )
add_test( NAME SystemTests COMMAND SystemTestsRunner )
