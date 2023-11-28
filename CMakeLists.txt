cmake_minimum_required(VERSION 2.8...3.14)
project(libffi C)

# Check headers
include ( CheckIncludeFiles )
check_include_files ( alloca.h HAVE_ALLOCA_H )
check_include_files ( dlfcn.h HAVE_DLFCN_H )
check_include_files ( inttypes.h HAVE_INTTYPES_H )
check_include_files ( memory.h HAVE_MEMORY_H )
check_include_files ( stdint.h HAVE_STDINT_H )
check_include_files ( stdlib.h HAVE_STDLIB_H )
check_include_files ( strings.h HAVE_STRINGS_H )
check_include_files ( string.h HAVE_STRING_H )
check_include_files ( sys/mman.h HAVE_SYS_MMAN_H )
check_include_files ( sys/stat.h HAVE_SYS_STAT_H )
check_include_files ( sys/types.h HAVE_SYS_TYPES_H )
check_include_files ( unistd.h HAVE_UNISTD_H )
set ( STDC_HEADERS 1 )
# Check functions
include ( CheckFunctionExists )
check_function_exists ( memcpy HAVE_MEMCPY )
check_function_exists ( mmap HAVE_MMAP )
if ( HAVE_MMAP )
  set ( HAVE_MMAP_ANON 1 )
  set ( HAVE_MMAP_FILE 1 )
endif ( )
check_function_exists ( alloca HAVE_ALLOCA )
# Check types
include ( CheckTypeSize )
check_type_size ( "long double" HAVE_LONG_DOUBLE )
check_type_size ( "double" SIZEOF_DOUBLE )
check_type_size ( "long double" SIZEOF_LONG_DOUBLE )
check_type_size ( "void*" SIZEOF_VOID_P )

set(SOURCES_LIST
    src/closures.c
    src/java_raw_api
    src/prep_cif.c
    src/raw_api.c
    src/tramp.c
    src/types.c)

if(CMAKE_BUILD_TYPE MATCHES DEBUG)
    list(APPEND SOURCES_LIST src/debug.c)
    add_definitions(-DFFI_DEBUG)
endif()

include(configure_platform.cmake)
include(configure_options.cmake)

file(COPY ${FFI_CONFIG_FILE} DESTINATION ${CMAKE_CURRENT_BINARY_DIR})
file(COPY src/${TARGETDIR}/ffitarget.h DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/include)

include_directories(${CMAKE_CURRENT_BINARY_DIR}/include)
include_directories(${CMAKE_CURRENT_BINARY_DIR})
include_directories(include)

add_definitions(-DFFI_BUILDING)

add_library(objlib OBJECT ${SOURCES_LIST})
set_property(TARGET objlib PROPERTY POSITION_INDEPENDENT_CODE 1)

add_library(ffi_static STATIC $<TARGET_OBJECTS:objlib>)

target_include_directories(
  ffi_static PUBLIC
  "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>")

if(MSVC)
    add_definitions(-DFFI_BUILDING_DLL)
    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
elseif(CMAKE_SYSTEM_NAME MATCHES Darwin)
    set(CMAKE_MACOSX_RPATH 1)
endif()
add_library(ffi_shared SHARED $<TARGET_OBJECTS:objlib> ${OBJECTS_LIST})

target_include_directories(
  ffi_shared PUBLIC
  "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>")

set_target_properties(ffi_static PROPERTIES OUTPUT_NAME ffi)
set_target_properties(ffi_shared PROPERTIES OUTPUT_NAME ffi)

install(TARGETS ffi_static ffi_shared
    EXPORT ${PROJECT_NAME}Targets
    RUNTIME DESTINATION bin
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib)

install(EXPORT ${PROJECT_NAME}Targets
    DESTINATION share/${PROJECT_NAME})

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/include/ffi.h
    ${CMAKE_CURRENT_BINARY_DIR}/include/ffitarget.h
    DESTINATION include)
