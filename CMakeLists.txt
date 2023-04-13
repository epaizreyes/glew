cmake_minimum_required(VERSION 2.8.12)
project("glew")

set(SRC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/src)
set(INCLUDE_DIR "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>$<INSTALL_INTERFACE:include>")
set(RC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/build)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/bin)

include("GeneratePkgConfig.cmake")

option(GLEW_BUILD_SHARED "Build the shared glew library" ON)
option(GLEW_BUILD_STATIC "Build the static glew library" ON)
option(USE_GLU "Use GLU" OFF)
option(PKG_CONFIG_REPRESENTATIVE_TARGET "Generate pc file for specified target as glew. glew|glew_shared" OFF)
option(GLEW_INSTALL "Generate installation target" ON)
option(ONLY_LIBS "Do not build executables" OFF)

file(STRINGS ${CMAKE_CURRENT_SOURCE_DIR}/config/version  _VERSION_MAJOR_STRING REGEX "GLEW_MAJOR[ ]*=[ ]*[0-9]+.*")
string(REGEX REPLACE "GLEW_MAJOR[ ]*=[ ]*([0-9]+)" "\\1" CPACK_PACKAGE_VERSION_MAJOR ${_VERSION_MAJOR_STRING})
file(STRINGS ${CMAKE_CURRENT_SOURCE_DIR}/config/version  _VERSION_MINOR_STRING REGEX "GLEW_MINOR[ ]*=[ ]*[0-9]+.*")
string(REGEX REPLACE "GLEW_MINOR[ ]*=[ ]*([0-9]+)" "\\1" CPACK_PACKAGE_VERSION_MINOR ${_VERSION_MINOR_STRING})
file(STRINGS ${CMAKE_CURRENT_SOURCE_DIR}/config/version  _VERSION_PATCH_STRING REGEX "GLEW_MICRO[ ]*=[ ]*[0-9]+.*")
string(REGEX REPLACE "GLEW_MICRO[ ]*=[ ]*([0-9]+)" "\\1" CPACK_PACKAGE_VERSION_PATCH ${_VERSION_PATCH_STRING})
set(GLEW_VERSION ${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH})

set(LIBGLEW_SRCS ${SRC_DIR}/glew.c)

set(DEFINITIONS)
if(WIN32)
	list(APPEND DEFINITIONS -DWIN32_MEAN_AND_LEAN -DVC_EXTRALEAN -D_CRT_SECURE_NO_WARNINGS)
endif()

if(MSVC)
	list(APPEND LIBGLEW_SRCS ${RC_DIR}/glew.rc)
endif()

# Use namespaced libraries when supported
if(${CMAKE_VERSION} VERSION_GREATER 3.14)
	set(USE_NAMESPACED_LIB YES)
else()
	set(USE_NAMESPACED_LIB NO)
endif()

if(POLICY CMP0072)
	set(OpenGL_GL_PREFERENCE GLVND)
endif()

find_package(OpenGL REQUIRED)

set(pc_requires)

if(NOT USE_GLU)
	list(APPEND DEFINITIONS -DGLEW_NO_GLU)
else()
	if(NOT OPENGL_GLU_FOUND)
		message(FATAL_ERROR "GLU is not found. but GLU option is enabled")
	endif()

	list(APPEND pc_requires glu)

	if(USE_NAMESPACED_LIB)
		list(APPEND LIBRARIES OpenGL::GLU)
	else()
		list(APPEND LIBRARIES OPENGL_glu_LIBRARY)
	endif()
endif()

list(APPEND pc_requires gl)
if(USE_NAMESPACED_LIB)
	list(APPEND LIBRARIES OpenGL::GL)
else()
	list(APPEND LIBRARIES OPENGL_opengl_LIBRARY)
endif()

if(APPLE)
	find_library(AGL_LIBRARY AGL)
	list(APPEND LIBRARIES ${AGL_LIBRARY})
elseif(NOT WIN32)
	find_package(X11 REQUIRED)

	list(APPEND pc_requires x11 xext)
	if(USE_NAMESPACED_LIB)
		list(APPEND LIBRARIES X11::X11 X11::Xext)
	else()
		list(APPEND LIBRARIES X11_X11_LIB X11_Xext_LIB)
	endif()
endif()

set(GLEW_TARGETS)

if(GLEW_BUILD_STATIC)
	add_library(glew STATIC ${LIBGLEW_SRCS})

	set_target_properties(glew PROPERTIES
		OUTPUT_NAME "glew"
		DEBUG_POSTFIX d)

		target_compile_definitions(glew PUBLIC GLEW_STATIC)
	list(APPEND GLEW_TARGETS glew)
endif()

if(GLEW_BUILD_SHARED)
	add_library(glew_shared SHARED ${LIBGLEW_SRCS})

	if(GLEW_BUILD_STATIC)
		set_target_properties(glew_shared PROPERTIES
			OUTPUT_NAME "glew-shared"
			DEBUG_POSTFIX d)
	else()
		set_target_properties(glew_shared PROPERTIES
			OUTPUT_NAME "glew"
			DEBUG_POSTFIX d)
	endif()

	target_compile_definitions(glew_shared PRIVATE GLEW_BUILD)
	list(APPEND GLEW_TARGETS glew_shared)
endif()

foreach(GLEW_TARGET ${GLEW_TARGETS})
	target_compile_definitions(${GLEW_TARGET} PUBLIC ${DEFINITIONS})
	target_include_directories(${GLEW_TARGET} PUBLIC ${INCLUDE_DIR})
	target_link_libraries(${GLEW_TARGET} ${LIBRARIES})
	set_target_properties(${GLEW_TARGET} PROPERTIES VERSION ${GLEW_VERSION})
endforeach()

if(PKG_CONFIG_REPRESENTATIVE_TARGET)
	GeneratePkgConfigFile(${PKG_CONFIG_REPRESENTATIVE_TARGET} "The OpenGL Extension Wrangler library"
		NAME "glew"
		LIBRARY_DIR lib
		REQUIRES ${pc_requires})
endif()

if(GLEW_INSTALL)
    install(TARGETS ${GLEW_TARGETS}
            EXPORT glew-cmake
            ARCHIVE DESTINATION lib)
    install(EXPORT glew-cmake DESTINATION lib/cmake/glew FILE glewConfig.cmake)

    file(GLOB PUBLIC_HEADERS "include/GL/*.h")
    install(FILES ${PUBLIC_HEADERS} DESTINATION include/GL/)
endif()

if(${CMAKE_CURRENT_SOURCE_DIR} STREQUAL ${CMAKE_SOURCE_DIR} AND NOT ONLY_LIBS)
	set(GLEWINFO_SRCS ${SRC_DIR}/glewinfo.c)
	set(VISUALINFO_SRCS ${SRC_DIR}/visualinfo.c)
	if(MSVS)
		list(APPEND GLEWINFO_SRCS ${RC_DIR}/glewinfo.rc)
		list(APPEND VISUALINFO_SRCS ${RC_DIR}/visualinfo.rc)
	endif()

	add_executable(glewinfo ${GLEWINFO_SRCS})
	add_executable(visualinfo ${VISUALINFO_SRCS})

	if(GLEW_BUILD_STATIC)
		target_link_libraries(glewinfo glew)
		target_link_libraries(visualinfo glew)
	else()
		target_link_libraries(glewinfo glew_shared)
		target_link_libraries(visualinfo glew_shared)
	endif()

    if(GLEW_INSTALL)
	   install(TARGETS glewinfo visualinfo DESTINATION bin)
    endif()
endif()

# Define solution tree organization
source_group(src FILES ${LIBGLEW_SRCS})
source_group(src FILES ${GLEWINFO_SRCS})
source_group(src FILES ${VISUALINFO_SRCS})