################################################################################
# Project:  Sip
# Purpose:  CMake build scripts
# Author:   Dmitry Baryshnikov, dmitry.baryshnikov@nexgis.com
################################################################################
# Copyright (C) 2017-2018, NextGIS <info@nextgis.com>
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

if(NOT DEFINED PACKAGE_VENDOR)
    set(PACKAGE_VENDOR NextGIS)
endif()

if(NOT DEFINED PACKAGE_BUGREPORT)
    set(PACKAGE_BUGREPORT info@nextgis.com)
endif()


# some init settings
set(CMAKE_COLOR_MAKEFILE ON)
# set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

if(OSX_FRAMEWORK AND (BUILD_SHARED_LIBS OR BUILD_STATIC_LIBS))
  message(FATAL_ERROR "Only OSX_FRAMEWORK key or any or both BUILD_SHARED_LIBS
                       and BUILD_STATIC_LIBS keys are permitted")
endif()

if(OSX_FRAMEWORK)
    set(CMAKE_MACOSX_RPATH ON)
endif()

set(INSTALL_CMAKECONF_DIR CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)

if(NOT WITH_PYTHON2 AND NOT WITH_PYTHON3)
    set(WITH_PYTHON2 ON)
endif()

if(WITH_PYTHON2)
    find_package(PythonInterp 2)
    if(PYTHONINTERP_FOUND)
        set(PYTHON2_EXECUTABLE ${PYTHON_EXECUTABLE})
        set(PYTHON2_VERSION_MAJOR ${PYTHON_VERSION_MAJOR})
        set(PYTHON2_VERSION_MINOR ${PYTHON_VERSION_MINOR})
    else()
        message(WARNING "Cannot find python 2")
    endif()
endif()

unset(PYTHONINTERP_FOUND)
unset(PYTHON_EXECUTABLE)
unset(PYTHON_VERSION_MAJOR)
unset(PYTHON_VERSION_MINOR)

if(WITH_PYTHON3)
    # Hack, not find python3 withouut this
    find_program(PYTHON3_EXECUTABLE python3)
    set(PYTHON_EXECUTABLE ${PYTHON3_EXECUTABLE})

    find_package(PythonInterp 3)
    if(PYTHONINTERP_FOUND)
        set(PYTHON3_EXECUTABLE ${PYTHON_EXECUTABLE})
        set(PYTHON3_VERSION_MAJOR ${PYTHON_VERSION_MAJOR})
        set(PYTHON3_VERSION_MINOR ${PYTHON_VERSION_MINOR})
    else()
        message(WARNING "Cannot find python 3")
    endif()
endif()

if(NOT PYTHON2_EXECUTABLE AND NOT PYTHON3_EXECUTABLE)
    message(FATAL_ERROR "Cannot find python")
endif()

if(PYTHON2_EXECUTABLE)
    set(PYTHON_EXECUTABLE ${PYTHON2_EXECUTABLE})
    set(PYTHON_VERSION_MAJOR ${PYTHON2_VERSION_MAJOR})
    set(PYTHON_VERSION_MINOR ${PYTHON2_VERSION_MINOR})
else()
    set(PYTHON_EXECUTABLE ${PYTHON3_EXECUTABLE})
    set(PYTHON_VERSION_MAJOR ${PYTHON3_VERSION_MAJOR})
    set(PYTHON_VERSION_MINOR ${PYTHON3_VERSION_MINOR})
endif()

include(util)
check_version(SIP_MAJOR_VERSION SIP_MINOR_VERSION SIP_PATCH_VERSION)
set(VERSION "${SIP_MAJOR_VERSION}.${SIP_MINOR_VERSION}.${SIP_PATCH_VERSION}")

report_version(${PROJECT_NAME} ${VERSION})

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
get_cpack_filename(${VERSION} PROJECT_CPACK_FILENAME)

if(OSX_FRAMEWORK)
    set(FLAGS "ARCHFLAGS=-arch x86_64")
elseif(WIN32)
    if(MSVC_VERSION EQUAL 1400)
        set(FLAGS ${FLAGS} "VS90COMNTOOLS=$ENV{VS80COMNTOOLS}")
    elseif(MSVC_VERSION EQUAL 1500)
        set(FLAGS ${FLAGS} "VS90COMNTOOLS=$ENV{VS90COMNTOOLS}")
    elseif(MSVC_VERSION EQUAL 1600)
        set(FLAGS ${FLAGS} "VS90COMNTOOLS=$ENV{VS100COMNTOOLS}")
    elseif(MSVC_VERSION EQUAL 1700)
        set(FLAGS ${FLAGS} "VS90COMNTOOLS=$ENV{VS110COMNTOOLS}")
    elseif(MSVC_VERSION EQUAL 1800)
        set(FLAGS ${FLAGS} "VS90COMNTOOLS=$ENV{VS120COMNTOOLS}")
    elseif(MSVC_VERSION EQUAL 1900)
        set(FLAGS ${FLAGS} "VS90COMNTOOLS=$ENV{VS140COMNTOOLS}")
    else() # 1912 - Need to copy vcvarsall.bat from VC\Auxiliary\Build to VC
        get_filename_component(_COMNTOOLS_PREFIX "${CMAKE_C_COMPILER}" PATH)
        get_filename_component(_COMNTOOLS_PREFIX "${_COMNTOOLS_PREFIX}" PATH)
        get_filename_component(_COMNTOOLS_PREFIX "${_COMNTOOLS_PREFIX}" PATH)
        get_filename_component(_COMNTOOLS_PREFIX "${_COMNTOOLS_PREFIX}" PATH)
        get_filename_component(_COMNTOOLS_PREFIX "${_COMNTOOLS_PREFIX}" PATH)
        get_filename_component(_COMNTOOLS_PREFIX "${_COMNTOOLS_PREFIX}" PATH)
        # get_filename_component(_COMNTOOLS_PREFIX "${_COMNTOOLS_PREFIX}" PATH)
        set(FLAGS ${FLAGS} "VS90COMNTOOLS=${_COMNTOOLS_PREFIX}")
    endif()
else()
    set(FLAGS "NextGIS=TRUE")
endif()

# Download tarbal
set(OUT_DIR_NAME ${PROJECT_NAME}-${VERSION})

if(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}")
    status_message("Download ${DOWNLOAD_URL}")
    file(DOWNLOAD "${DOWNLOAD_URL}" "${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}"
        SHOW_PROGRESS
    )
endif()

# Extract archive
if(NOT EXISTS ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME})
    status_message("Extract ${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}")
    execute_process(COMMAND ${CMAKE_COMMAND} -E tar xzf ${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME})
endif()

set(TMP_INSTALL_PREFIX "${CMAKE_CURRENT_BINARY_DIR}/inst_tmp")

if(OSX_FRAMEWORK)
    set(INSTALL_PYTHON_DIR Library/Python/${PYTHON_VERSION_MAJOR}.${PYTHON_VERSION_MINOR}/site-packages)
else()
    set(INSTALL_PYTHON_DIR lib/python${PYTHON_VERSION_MAJOR}${PYTHON_VERSION_MINOR}/site-packages)
endif()
set(CONFIGURE_ARGS --destdir ${TMP_INSTALL_PREFIX}/${INSTALL_PYTHON_DIR})

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} --debug)
endif()

if(APPLE)
    # may be --arch x86 --arch x86_64 --arch ppc --arch ppc64
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} --arch=x86_64)
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} --deployment-target=${CMAKE_OSX_DEPLOYMENT_TARGET})
endif()

set(CONFIGURE_ARGS ${CONFIGURE_ARGS} --bindir ${TMP_INSTALL_PREFIX}/bin)
set(CONFIGURE_ARGS ${CONFIGURE_ARGS} --incdir ${TMP_INSTALL_PREFIX}/include)
set(CONFIGURE_ARGS ${CONFIGURE_ARGS} --sipdir ${TMP_INSTALL_PREFIX}/share/${PROJECT_NAME})

message(STATUS "Configure args: ${CONFIGURE_ARGS}")

macro(build_py ver)

    add_custom_target(configure_py${ver} ALL
        COMMAND ${CMAKE_COMMAND} -E env "${FLAGS}"
        ${PYTHON${ver}_EXECUTABLE} configure.py ${CONFIGURE_ARGS}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}
        VERBATIM
    )

    if(WIN32)
        add_custom_target(build_py${ver} ALL
            COMMAND ${CMAKE_COMMAND} -E env "${FLAGS}" nmake
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}
            DEPENDS configure_py${ver}
            VERBATIM
        )
        add_custom_target(install_py${ver} ALL
            COMMAND ${CMAKE_COMMAND} -E env "${FLAGS}" nmake install
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}
            DEPENDS build_py${ver}
            VERBATIM
        )
    else()
        add_custom_target(build_py${ver} ALL
            COMMAND ${CMAKE_COMMAND} -E env "${FLAGS}" make
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}
            DEPENDS configure_py${ver}
            VERBATIM
        )
        add_custom_target(install_py${ver} ALL
            COMMAND ${CMAKE_COMMAND} -E env "${FLAGS}" make install
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}
            DEPENDS build_py${ver}
            VERBATIM
        )
    endif()

endmacro()

if(PYTHON2_EXECUTABLE)
    build_py(2)
endif()

if(PYTHON3_EXECUTABLE)
    build_py(3)
endif()

set(PACKAGE_NAME ${PROJECT_NAME})
string(TOUPPER ${PACKAGE_NAME} PACKAGE_UPPER_NAME)

configure_file(cmake/PackageConfig.cmake.in
    ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake @ONLY)

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL )
    install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/inst_tmp/ DESTINATION ".")
    install(FILES
      ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake
      DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)
endif()

# Archiving ====================================================================
set(PACKAGE_NAME ${PROJECT_NAME})

set(CPACK_PACKAGE_NAME "${PACKAGE_NAME}")
set(CPACK_PACKAGE_VENDOR "${PACKAGE_VENDOR}")
set(CPACK_PACKAGE_VERSION "${VERSION}")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${PACKAGE_NAME} Installation")
set(CPACK_PACKAGE_RELOCATABLE TRUE)
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
set(CPACK_GENERATOR "ZIP")
set(CPACK_MONOLITHIC_INSTALL ON)
set(CPACK_STRIP_FILES TRUE)

# Get cpack zip archive name
set(CPACK_PACKAGE_FILE_NAME ${PROJECT_CPACK_FILENAME})

include(CPack)
