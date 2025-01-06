cmake_minimum_required(VERSION 3.16)

set(FREELIB_VERSION_MAJOR "6")
set(FREELIB_VERSION_MINOR "1")
set(FREELIB_VERSION_PATCH "51")
set(FREELIB_VERSION "${FREELIB_VERSION_MAJOR}.${FREELIB_VERSION_MINOR}.${FREELIB_VERSION_PATCH}")
project(freeLib VERSION "${FREELIB_VERSION}" LANGUAGES CXX)

set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)

set(CMAKE_INCLUDE_CURRENT_DIR ON)

# Retrieve git HEAD revision hash
set(GIT_HEAD_OVERRIDE "" CACHE STRING "Manually set the Git HEAD hash when missing (eg, when no .git folder exists)")
execute_process(COMMAND git rev-parse --short=7 HEAD
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE GIT_HEAD
        ERROR_QUIET)
string(STRIP "${GIT_HEAD}" GIT_HEAD)
if(GIT_HEAD STREQUAL "" AND NOT GIT_HEAD_OVERRIDE STREQUAL "")
    string(SUBSTRING "${GIT_HEAD_OVERRIDE}" 0 7 GIT_HEAD)
elseif(EXISTS ${CMAKE_SOURCE_DIR}/.gitrev)
    file(READ ${CMAKE_SOURCE_DIR}/.gitrev GIT_HEAD)
endif()
message(STATUS "Found Git HEAD Revision: ${GIT_HEAD}\n")

if(UNIX AND NOT APPLE)
    file(COPY src/xsl/fonts DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/xsl)

    include(GNUInstallDirs)
    set(DATA_INSTALL_DIR "${CMAKE_INSTALL_DATADIR}/freelib")

    install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/xsl/fonts DESTINATION ${DATA_INSTALL_DIR})
endif()

if(APPLE)
    set(DATA_INSTALL_DIR "freelib")
endif()

add_subdirectory(src/)
add_subdirectory(data/)

install(TARGETS freelib
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    BUNDLE DESTINATION .
)

if(APPLE)
    set(ICNS_PATH ${CMAKE_SOURCE_DIR}/data/macos/freeLib.icns)
    set_source_files_properties(${ICNS_PATH} PROPERTIES
        MACOSX_PACKAGE_LOCATION "Resources"
    )
    target_sources(freelib PRIVATE
        ${ICNS_PATH}
    )
    set_target_properties(freelib PROPERTIES
        MACOSX_BUNDLE_BUNDLE_NAME "freeLib"
        MACOSX_BUNDLE_BUNDLE_VERSION ${PROJECT_VERSION}
        MACOSX_BUNDLE_GUI_IDENTIFIER org.vovkasm.freeLib
        MACOSX_BUNDLE_ICON_FILE "freeLib.icns"
        MACOSX_BUNDLE_SHORT_VERSION_STRING ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}
    )

    install(CODE [[
        set(CMAKE_INSTALL_PREFIX "" CACHE FILEPATH "Install prefix (hack to force cmake to make path absolute)")
        message("CMAKE_INSTALL_PREFIX: ${CMAKE_INSTALL_PREFIX}")

        include(BundleUtilities)
        set(BU_CHMOD_BUNDLE_ITEMS 1)
        fixup_bundle("${CMAKE_INSTALL_PREFIX}/freelib.app" "" "")
    ]])
endif()


