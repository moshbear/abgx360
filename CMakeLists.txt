cmake_minimum_required(VERSION 3.0)

project(abgx360 LANGUAGES C)

include(CheckFunctionExists)
include(CheckIncludeFiles)
include(CheckLibraryExists)
include(CheckSymbolExists)
include(CheckTypeSize)

find_package(CURL COMPONENTS libcurl REQUIRED)


CHECK_LIBRARY_EXISTS(z crc32 "" HAVE_LIBZ)
if (NOT HAVE_LIBZ)
	message(FATAL_ERROR "zlib not found")
endif ()

CHECK_SYMBOL_EXISTS(__FreeBSD__ stddef.h IS_FREEBSD)

if (IS_FREEBSD)
	CHECK_INCLUDE_FILES(camlib.h HAVE_CAMLIB_H)
	CHECK_LIBRARY_EXISTS(cam cam_open_device HAVE_CAM_LIB)
	if (NOT HAVE_CAM_LIB)
		message(FATAL_ERROR "CAM library not found")
	endif ()
endif ()

foreach (hdr zlib.h curl/curl.h fnmatch.h pwd.h limits.h malloc.h stddef.h
		stdlib.h string.h strings.h sys/time.h termios.h unistd.h
		fcntl.h stdbool.h time.h
		dirent.h limits.h)
	string(REGEX REPLACE "[/.]" "_" var ${hdr})
	string(TOUPPER "${var}" var_u)
	CHECK_INCLUDE_FILES(${hdr} HAVE_${var_u})
	if (NOT HAVE_${var_u})
		message(FATAL_ERROR "Header <${hdr}> not found")
	endif ()
	unset(var)
	unset(var_u)
endforeach ()

foreach (type size_t off_t int32_t ptrdiff_t)
	string(TOUPPER "${type}" type_u)
	CHECK_TYPE_SIZE(${type} HAVE_${type_u})
	if ("${type_u}" STREQUAL "")
		message(FATAL_ERROR "Type ${type} not found")
	endif ()
endforeach ()


set (CMAKE_REQUIRED_LIBRARIES "m")
foreach (func atexit gettimeofday memset strcasecmp strerror strncasecmp
		strtol strtoul strstr mkdir pow)
	string(TOUPPER "${func}" func_u)
	CHECK_FUNCTION_EXISTS(${func} HAVE_${func_u})
	if (NOT HAVE_${func_u})
		message(FATAL_ERROR "Function ${func} not found")
	endif ()
endforeach ()
unset (CMAKE_REQUIRED_LIBRARIES)

CHECK_FUNCTION_EXISTS(fseeko HAVE_FSEEKO)

configure_file (${CMAKE_CURRENT_SOURCE_DIR}/config.h.in
	${CMAKE_CURRENT_BINARY_DIR}/config.h)

add_executable(abgx360 src/abgx360.c src/rijndael-alg-fst.c src/sha1.c
	src/mspack/lzxd.c src/mspack/system.c)
target_compile_options(abgx360 PRIVATE -Wall -W)
target_compile_features(abgx360 PRIVATE c_std_90)
target_link_libraries(abgx360 PRIVATE ${CURL_LIBRARIES} m z)
target_include_directories(abgx360 PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
target_compile_definitions(abgx360 PRIVATE HAVE_CONFIG_H)

install(TARGETS abgx360 RUNTIME DESTINATION bin)
