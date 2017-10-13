cmake_minimum_required(VERSION 3.2)

if(NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Type of build")
endif()
set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "RelWithDebInfo" "MinSizeRel")

# force static link of MSVC runtime by overriding compiler flags
# https://cmake.org/Wiki/CMake_FAQ#How_can_I_build_my_MSVC_application_with_a_static_runtime.3F
set(CMAKE_USER_MAKE_RULES_OVERRIDE ${CMAKE_CURRENT_SOURCE_DIR}/cmake/msvc_c_flags_override.cmake)
set(CMAKE_USER_MAKE_RULES_OVERRIDE_CXX ${CMAKE_CURRENT_SOURCE_DIR}/cmake/msvc_cxx_flags_override.cmake)

project(sfwcl)

# install target is not used here
set(CMAKE_INSTALL_PREFIX "${PROJECT_BINARY_DIR}" CACHE INTERNAL "")

# Crysis SDK headers
include_directories(src/cpp/mod/CryEngine)

file(GLOB SFWCL_MOD_SOURCES src/cpp/mod/*.cpp)
add_library(sfwcl MODULE ${SFWCL_MOD_SOURCES})

option(SFWCL_PAK "Create sfwcl.pak" ON)
if(SFWCL_PAK)
	set(SFWCL_PAK_FILE "${PROJECT_BINARY_DIR}/sfwcl.pak")
	set(SFWCL_PAK_BASE "${PROJECT_SOURCE_DIR}/src/lua")
	add_custom_command(
		OUTPUT ${SFWCL_PAK_FILE}
		COMMAND ${CMAKE_COMMAND} -E tar cvf ${SFWCL_PAK_FILE} --format=zip Scripts
		WORKING_DIRECTORY ${SFWCL_PAK_BASE}
	)
	add_custom_target(mod_pak ALL DEPENDS ${SFWCL_PAK_FILE})
endif()

option(SFWCL_LAUNCHER "Build SafeWritingClient.exe" OFF)
if(SFWCL_LAUNCHER)
	add_executable(SafeWritingClient src/cpp/launcher/launcher.cpp)
endif()

option(SFWCL_MAPDL "Build MapDownloader.exe" OFF)
if(SFWCL_MAPDL)
	add_executable(MapDownloader src/cpp/mapdl/mapdl.cpp)
	# hide not needed options of zlib
	mark_as_advanced(FORCE ASM686 AMD64 INSTALL_BIN_DIR INSTALL_LIB_DIR INSTALL_INC_DIR INSTALL_MAN_DIR INSTALL_PKGCONFIG_DIR)
	add_subdirectory("${PROJECT_SOURCE_DIR}/lib/zlib" EXCLUDE_FROM_ALL)
	add_library(miniunz STATIC
		"${PROJECT_SOURCE_DIR}/lib/zlib/contrib/minizip/unzip.c"
		"${PROJECT_SOURCE_DIR}/lib/zlib/contrib/minizip/ioapi.c"
		"${PROJECT_SOURCE_DIR}/lib/zlib/contrib/minizip/iowin32.c"
	)
	target_include_directories(miniunz PUBLIC
		"${PROJECT_SOURCE_DIR}/lib/zlib/contrib/minizip"
		"${PROJECT_SOURCE_DIR}/lib/zlib"
		"${PROJECT_BINARY_DIR}/lib/zlib"
	)
	target_link_libraries(miniunz zlibstatic)
	target_link_libraries(MapDownloader miniunz)
endif()

if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
	message(STATUS "Using MSVC compiler options")
	add_definitions(-D_CRT_SECURE_NO_WARNINGS)
	add_definitions(-D_SILENCE_STDEXT_HASH_DEPRECATION_WARNINGS)
	if(SFWCL_MAPDL)
		set_target_properties(MapDownloader PROPERTIES LINK_FLAGS "/SUBSYSTEM:WINDOWS")
	endif()
elseif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
	message(STATUS "Using GNU compiler options")
	set(SFWCL_MINGW_STATIC_FLAGS "-static-libstdc++ -static-libgcc")
	option(SFWCL_MINGW_STATIC "Link with ${SFWCL_MINGW_STATIC_FLAGS}" ON)
	if(SFWCL_MINGW_STATIC)
		list(APPEND CMAKE_EXE_LINKER_FLAGS ${SFWCL_MINGW_STATIC_FLAGS})
		list(APPEND CMAKE_SHARED_LINKER_FLAGS ${SFWCL_MINGW_STATIC_FLAGS})
		list(APPEND CMAKE_MODULE_LINKER_FLAGS ${SFWCL_MINGW_STATIC_FLAGS})
	endif()
	if(SFWCL_MAPDL)
		set_target_properties(MapDownloader PROPERTIES LINK_FLAGS "-mwindows")
	endif()
else()
	message("Unknown compiler \'" ${CMAKE_CXX_COMPILER_ID} "\'")
endif()