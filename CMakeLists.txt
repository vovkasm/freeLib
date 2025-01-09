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

set(CMAKE_OSX_DEPLOYMENT_TARGET 10.15)
set(CMAKE_XCODE_ATTRIBUTE_DEVELOPMENT_TEAM "GE3M8BMVUQ")
set(CMAKE_XCODE_ATTRIBUTE_CODE_SIGN_IDENTITY "Developer ID Application")
set(CMAKE_XCODE_ATTRIBUTE_CODE_SIGN_STYLE "Manual")

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

add_subdirectory(src/)
add_subdirectory(data/)

install(TARGETS freelib
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    BUNDLE DESTINATION .
)

if(APPLE)
    set(ICNS_PATH ${CMAKE_SOURCE_DIR}/data/macos/freeLib.icns)
    set_source_files_properties(${ICNS_PATH} TARGET_DIRECTORY freelib PROPERTIES MACOSX_PACKAGE_LOCATION "Resources")

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

    find_package(QT NAMES Qt6 Qt5 COMPONENTS Core)
    set(QT "Qt${QT_VERSION_MAJOR}")
    find_package(${QT} COMPONENTS Core)
    get_target_property(QMAKE_EXECUTABLE Qt5::qmake IMPORTED_LOCATION)
    execute_process(
        COMMAND ${QMAKE_EXECUTABLE} -query QT_INSTALL_PLUGINS
        OUTPUT_VARIABLE QT_PLUGIN_DIR
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )

    install(CODE "set(TARGET_BUNDLE_NAME \"freelib.app\")")
    install(CODE "set(PLUGIN_DIR \"${QT_PLUGIN_DIR}\")")
    install(CODE [[
        set(CMAKE_INSTALL_PREFIX "" CACHE FILEPATH "Install prefix (hack to force cmake to make path absolute)")
        message("CMAKE_INSTALL_PREFIX: ${CMAKE_INSTALL_PREFIX}")

        set(BUNDLE_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/${TARGET_BUNDLE_NAME}")


        foreach(PLUGIN "platforms" "sqldrivers" "imageformats" "tls")
            if(EXISTS ${PLUGIN_DIR}/${PLUGIN})
                file(GLOB files "${PLUGIN_DIR}/${PLUGIN}/*")
                foreach(file ${files})
                        file(GET_RUNTIME_DEPENDENCIES
                            EXECUTABLES ${file}
                            RESOLVED_DEPENDENCIES_VAR PLUGINS
                            UNRESOLVED_DEPENDENCIES_VAR unresolved_deps
                        )

                        foreach(DEPENDENCY ${PLUGINS})
                                file(INSTALL
                                    DESTINATION "${BUNDLE_INSTALL_DIR}/Contents/lib"
                                    TYPE SHARED_LIBRARY
                                    FILES ${DEPENDENCY}
                                    FOLLOW_SYMLINK_CHAIN
                                )
                        endforeach()

                        get_filename_component(singleQtLib ${file} NAME)
                        list(APPEND QT_PLUGINS "${BUNDLE_INSTALL_DIR}/Contents/plugins/${PLUGIN}/${singleQtLib}")
                        file(INSTALL
                            FILES ${file}
                            DESTINATION "${BUNDLE_INSTALL_DIR}/Contents/plugins/${PLUGIN}"
                            TYPE SHARED_LIBRARY
                        )

                endforeach()
            endif()
        endforeach()

        include(BundleUtilities)
        set(BU_CHMOD_BUNDLE_ITEMS 1)
        fixup_bundle("${BUNDLE_INSTALL_DIR}" "${QT_PLUGINS}" "${BUNDLE_INSTALL_DIR}/Contents/lib")
        file(REMOVE_RECURSE "${BUNDLE_INSTALL_DIR}/Contents/lib")

        file(GLOB_RECURSE LIBS FOLLOW_SYMLINKS "${BUNDLE_INSTALL_DIR}/*.dylib")
        file(GLOB FRAMEWORKS FOLLOW_SYMLINKS LIST_DIRECTORIES ON "${BUNDLE_INSTALL_DIR}/Contents/Frameworks/*")
        foreach(item ${LIBS} ${FRAMEWORKS} ${BUNDLE_INSTALL_DIR})
            set(cmd codesign --deep --force --sign "Developer ID Application: VLADIMIR TIMOFEEV" "${item}")
            execute_process(
                COMMAND ${cmd}
                RESULT_VARIABLE codesign_result
            )

            if(NOT codesign_result EQUAL 0)
                message(WARNING "macOS signing failed; ${cmd} returned ${codesign_result}")
            endif()
        endforeach()
    ]])
endif()


