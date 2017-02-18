################################################################################
# Project:  Lib PyQt
# Purpose:  CMake build scripts
# Author:   Dmitry Baryshnikov, dmitry.baryshnikov@nexgis.com
################################################################################
# Copyright (C) 2017, NextGIS <info@nextgis.com>
#
# Permission is hereby granted, free of charge, to any person obtaining a
# copy of this software and associated documentation files (the "Software"),
# to deal in the Software without restriction, including without limitation
# the rights to use, copy, modify, merge, publish, distribute, sublicense,
# and/or sell copies of the Software, and to permit persons to whom the
# Software is furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included
# in all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
# OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
# THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
# FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
# DEALINGS IN THE SOFTWARE.
################################################################################
cmake_minimum_required(VERSION 2.8.12 FATAL_ERROR)

project(sip)

# some init settings
set(CMAKE_COLOR_MAKEFILE ON)
# set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

include(util)
check_version(SIP_MAJOR_VERSION SIP_MINOR_VERSION SIP_PATCH_VERSION)
set(VERSION "${SIP_MAJOR_VERSION}.${SIP_MINOR_VERSION}.${SIP_PATCH_VERSION}")
report_version(${PROJECT_NAME} ${VERSION})

set(CONFIGURE_ARGS --destdir ${CMAKE_INSTALL_PREFIX})

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} --debug)
endif()

if(APPLE)
    # may be --arch x86 --arch x86_64 --arch ppc --arch ppc64
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} --arch x86_64)
endif()

set(CONFIGURE_ARGS ${CONFIGURE_ARGS} --bindir ${CMAKE_INSTALL_PREFIX}/bin)
set(CONFIGURE_ARGS ${CONFIGURE_ARGS} --incdir ${CMAKE_INSTALL_PREFIX}/include)
set(CONFIGURE_ARGS ${CONFIGURE_ARGS} --sipdir ${CMAKE_INSTALL_PREFIX}/share/${PROJECT_NAME})

if(SIP_PATCH_VERSION EQUAL 0)
    set(SIP_INPUT_NAME "sip-${SIP_MAJOR_VERSION}.${SIP_MINOR_VERSION}")
else()
    set(SIP_INPUT_NAME "sip-${SIP_MAJOR_VERSION}.${SIP_MINOR_VERSION}.${SIP_PATCH_VERSION}")
endif()
set(DOWNLOAD_URL "https://sourceforge.net/projects/pyqt/files/sip/${SIP_INPUT_NAME}")
if(WIN32)
    # https://sourceforge.net/projects/pyqt/files/sip/sip-4.19/sip-4.19.zip
    set(DOWNLOAD_URL "${DOWNLOAD_URL}/${SIP_INPUT_NAME}.zip")
else()
    # https://sourceforge.net/projects/pyqt/files/sip/sip-4.19/sip-4.19.tar.gz
    set(DOWNLOAD_URL "${DOWNLOAD_URL}/${SIP_INPUT_NAME}.tar.gz")
endif()
set(OUT_NAME "sip-${VERSION}.tar.gz")

if(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}")
    status_message("Download ${DOWNLOAD_URL}")
    file(DOWNLOAD "${DOWNLOAD_URL}" "${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}"
         SHOW_PROGRESS
     )
endif()

include(ProcessorCount)
ProcessorCount(CPU_COUNT)
if(NOT CPU_COUNT OR CPU_COUNT EQUAL 0)
    set(CPU_COUNT 1)
endif()

find_package(PythonInterp REQUIRED)

message(STATUS "Configure args: ${CONFIGURE_ARGS}")

if(NOT EXISTS ${CMAKE_CURRENT_BINARY_DIR}/${SIP_INPUT_NAME})
    status_message("Extract ${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}")
    execute_process(COMMAND ${CMAKE_COMMAND} -E tar xzf ${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME})
endif()

status_message("Configure")
execute_process(COMMAND ${PYTHON_EXECUTABLE} configure.py ${CONFIGURE_ARGS}
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${SIP_INPUT_NAME}
)

#TODO: Fix on WINDOWS
add_custom_target(build_qt ALL make -j ${CPU_COUNT}
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${SIP_INPUT_NAME}
    VERBATIM
)

install(CODE "execute_process(
              COMMAND make install
              WORKING_DIRECTORY  ${CMAKE_CURRENT_BINARY_DIR}/${SIP_INPUT_NAME}
             )"
)

if (REGISTER_PACKAGE)
    set(INSTALL_INC_DIR ${CMAKE_INSTALL_PREFIX}/include)
    set(INSTALL_LIB_DIR ${CMAKE_INSTALL_PREFIX}/lib)
    set(INSTALL_BIN_DIR ${CMAKE_INSTALL_PREFIX}/bin)
    set(INSTALL_BIN_PATH ${INSTALL_BIN_DIR}/sip)
    set(INSTALL_SHARE_DIR ${CMAKE_INSTALL_PREFIX}/share/${PROJECT_NAME})

    set(EXPORT_PACKAGE_NAME SIP)
    string(TOUPPER ${EXPORT_PACKAGE_NAME} EXPORT_PACKAGE_NAME_UPPER)
    set(EXPORT_PACKAGE_LIB_NAME sip)

    export(PACKAGE ${EXPORT_PACKAGE_NAME})

    include(CMakePackageConfigHelpers)
    configure_package_config_file(
        ${CMAKE_SOURCE_DIR}/cmake/PackageConfig.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}/${EXPORT_PACKAGE_NAME}Config.cmake
        INSTALL_DESTINATION ${CMAKE_CURRENT_BINARY_DIR} )
    write_basic_package_version_file(
        ${CMAKE_CURRENT_BINARY_DIR}/${EXPORT_PACKAGE_NAME}ConfigVersion.cmake
        VERSION ${VERSION}
        COMPATIBILITY AnyNewerVersion )
endif()
