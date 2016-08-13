# Hunter package manager CMakeLists.txt
# Base example from:
# https://github.com/forexample/package-example/
# With some logic taken from the original glew's CMakeLists.txt in build/cmake

cmake_minimum_required(VERSION 3.0)

include("cmake/HunterGate.cmake")

HunterGate(
    URL "https://github.com/ruslo/hunter/archive/v0.15.6.tar.gz"
    SHA1 "3dc0cd2ccbc52e8be93967dbc583303a1e825aa2"
)


# get version from config/version
file (STRINGS ${CMAKE_CURRENT_SOURCE_DIR}/config/version  _VERSION_MAJOR_STRING REGEX "GLEW_MAJOR[ ]*=[ ]*[0-9]+.*")
string (REGEX REPLACE "GLEW_MAJOR[ ]*=[ ]*([0-9]+)" "\\1" CPACK_PACKAGE_VERSION_MAJOR ${_VERSION_MAJOR_STRING})
file (STRINGS ${CMAKE_CURRENT_SOURCE_DIR}/config/version  _VERSION_MINOR_STRING REGEX "GLEW_MINOR[ ]*=[ ]*[0-9]+.*")
string (REGEX REPLACE "GLEW_MINOR[ ]*=[ ]*([0-9]+)" "\\1" CPACK_PACKAGE_VERSION_MINOR ${_VERSION_MINOR_STRING})
file (STRINGS ${CMAKE_CURRENT_SOURCE_DIR}/config/version  _VERSION_PATCH_STRING REGEX "GLEW_MICRO[ ]*=[ ]*[0-9]+.*")
string (REGEX REPLACE "GLEW_MICRO[ ]*=[ ]*([0-9]+)" "\\1" CPACK_PACKAGE_VERSION_PATCH ${_VERSION_PATCH_STRING})
set (GLEW_VERSION ${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH})

project(glew VERSION ${GLEW_VERSION})

list(APPEND glew_headers
    ${CMAKE_CURRENT_SOURCE_DIR}/include/GL/glew.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/GL/glxew.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/GL/wglew.h
)

list(APPEND glew_sources src/glew.c)
if(WIN32)
  list(APPEND glew_sources build/glew.rc)
endif()

add_library(glew ${glew_sources} ${glew_headers})

find_package(OpenGL REQUIRED)
target_link_libraries(glew PUBLIC ${OPENGL_LIBRARIES})

if(NOT APPLE AND UNIX)
  hunter_add_package(x11)
  hunter_add_package(xau)
  hunter_add_package(xcb)
  find_package(X11 REQUIRED)
  list(APPEND glew_LIBRARIES "${X11_X11_LIB}")
  if(NOT BUILD_SHARED_LIBS)
    find_library(X11_Xcb_LIB xcb)
    list(APPEND glew_LIBRARIES "${X11_Xcb_LIB}")
    # Libs below are found through find_package(X11 REQUIRED) above
    list(APPEND glew_LIBRARIES "${X11_Xau_LIB}")
    list(APPEND glew_LIBRARIES "${CMAKE_DL_LIBS}")
    target_link_libraries(glew PUBLIC "${glew_LIBRARIES}")
  endif()
endif()

target_compile_definitions(glew PUBLIC GLEW_BUILD GLEW_NO_GLU)
target_include_directories(glew PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/include>")

if (NOT BUILD_SHARED_LIBS)
  target_compile_definitions(glew PUBLIC GLEW_STATIC)
endif()

if(MSVC)
  target_compile_definitions(glew PRIVATE VC_EXTRALEAN)
  target_link_libraries (glew PRIVATE -BASE:0x62AA0000)
endif()


#### install

set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(include_install_dir "include")

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(targets_export_name "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

include(CMakePackageConfigHelpers)

write_basic_package_version_file(
    "${version_config}" COMPATIBILITY SameMajorVersion
)

configure_package_config_file(
    "cmake/Config.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)

install(
    TARGETS glew
    EXPORT "${targets_export_name}"
    LIBRARY DESTINATION "lib"
    ARCHIVE DESTINATION "lib"
    RUNTIME DESTINATION "bin"
    INCLUDES DESTINATION "${include_install_dir}"
)

install(
    FILES ${glew_headers}
    DESTINATION "${include_install_dir}/GL"
)

install(
    FILES "${project_config}" "${version_config}"
    DESTINATION "${config_install_dir}"
)

install(
    EXPORT "${targets_export_name}"
    NAMESPACE "${namespace}"
    DESTINATION "${config_install_dir}"
)
