cmake_minimum_required(VERSION 3.16)

# KDE Application Version, managed by release script
set (RELEASE_SERVICE_VERSION_MAJOR "21")
set (RELEASE_SERVICE_VERSION_MINOR "11")
set (RELEASE_SERVICE_VERSION_MICRO "70")
set (RELEASE_SERVICE_VERSION "${RELEASE_SERVICE_VERSION_MAJOR}.${RELEASE_SERVICE_VERSION_MINOR}.${RELEASE_SERVICE_VERSION_MICRO}")

project(okular VERSION ${RELEASE_SERVICE_VERSION})

set(QT_REQUIRED_VERSION "5.12.0") # Remember to update the QT_DEPRECATED_WARNINGS_SINCE below
set(KF5_REQUIRED_VERSION "5.68.0") # Remember to update the KF_DEPRECATED_WARNINGS_SINCE below

set(OKULAR_UI "desktop" CACHE STRING "Which Okular user interface to build. Possible values: desktop, mobile, both. Default: desktop")

if(OKULAR_UI STREQUAL "desktop" OR OKULAR_UI STREQUAL "both")
    set(BUILD_DESKTOP ON)
else()
    set(BUILD_DESKTOP OFF)
endif()
if(OKULAR_UI STREQUAL "mobile" OR OKULAR_UI STREQUAL "both")
    set(BUILD_MOBILE ON)
else()
    set(BUILD_MOBILE OFF)
endif()

if (ANDROID)
    set(QT_REQUIRED_VERSION "5.13.0")
endif()

find_package(ECM ${KF5_REQUIRED_VERSION} CONFIG REQUIRED)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${ECM_MODULE_PATH})

include(ECMInstallIcons)
include(ECMSetupVersion)
include(ECMOptionalAddSubdirectory)
include(GenerateExportHeader)
include(FeatureSummary)
include(ECMAddAppIcon)
include(KDECompilerSettings NO_POLICY_SCOPE)
include(KDEInstallDirs)
include(KDECMakeSettings)
include(ECMAddTests)
include(ECMAddAppIcon)
include(CMakePackageConfigHelpers)
include(ECMSetupQtPluginMacroNames)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

ecm_setup_version(${PROJECT_VERSION}
                  VARIABLE_PREFIX OKULAR
                  VERSION_HEADER "${CMAKE_CURRENT_BINARY_DIR}/core/version.h"
                  PACKAGE_VERSION_FILE "${CMAKE_CURRENT_BINARY_DIR}/Okular5ConfigVersion.cmake")

find_package(Qt5 ${QT_REQUIRED_VERSION} CONFIG REQUIRED COMPONENTS Core Test Widgets PrintSupport Svg Qml Quick)
if (BUILD_DESKTOP)
    find_package(Qt5 ${QT_REQUIRED_VERSION} CONFIG REQUIRED COMPONENTS DBus)
endif()
find_package(Qt5 ${QT_REQUIRED_VERSION} OPTIONAL_COMPONENTS TextToSpeech)
if (NOT Qt5TextToSpeech_FOUND)
    message(STATUS "Qt5TextToSpeech not found, speech features will be disabled")
else()
    add_definitions(-DHAVE_SPEECH)
endif()

if(ANDROID)
    find_package(Qt5 ${QT_REQUIRED_VERSION} CONFIG REQUIRED COMPONENTS AndroidExtras)
endif()

ecm_setup_qtplugin_macro_names(
    JSON_ARG2
        "OKULAR_EXPORT_PLUGIN"
    CONFIG_CODE_VARIABLE
        PACKAGE_SETUP_AUTOMOC_VARIABLES
)

set(optionalComponents)
if (ANDROID)
#   we want to make sure that generally all components are found

    set(optionalComponents "OPTIONAL_COMPONENTS")
endif()

find_package(KF5 ${KF5_REQUIRED_VERSION} REQUIRED COMPONENTS
    Archive
    Bookmarks
    Completion
    Config
    ConfigWidgets
    CoreAddons
    Crash
    I18n
    IconThemes
    KIO
    Parts
    TextWidgets
    ThreadWeaver
    WindowSystem
    ${optionalComponents}
    DocTools
    JS
    Wallet
)

if(KF5Wallet_FOUND)
    add_definitions(-DWITH_KWALLET=1)
endif()
if(KF5JS_FOUND)
    add_definitions(-DWITH_KJS=1)
endif()

if(NOT WIN32 AND NOT ANDROID)
    find_package(KF5 ${KF5_REQUIRED_VERSION} REQUIRED COMPONENTS
        Activities
    )
    set_package_properties("KF5Activities" PROPERTIES
        DESCRIPTION  "Activities interface library"
        URL "https://api.kde.org/frameworks/kactivities/html/"
        TYPE RECOMMENDED
        PURPOSE "Required for Activities integration.")
endif()
find_package(KF5Kirigami2)
set_package_properties(KF5Kirigami2 PROPERTIES
    DESCRIPTION "A QtQuick based components set"
    PURPOSE "Required at runtime by the mobile app"
    TYPE RUNTIME
)
find_package(Phonon4Qt5 CONFIG REQUIRED)
find_package(KF5Purpose)
set_package_properties(KF5Purpose PROPERTIES
    DESCRIPTION "A framework for services and actions integration"
    PURPOSE "Required for enabling the share menu in Okular"
    TYPE OPTIONAL
)
if (KF5Purpose_FOUND)
    set(PURPOSE_FOUND 1)
else()
    set(PURPOSE_FOUND 0)
endif()

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_SOURCE_DIR}/cmake/modules)

find_package(ZLIB REQUIRED)

# This is here instead of in generators since we use if(Poppler_Qt5_FOUND) in autotests/
find_package(Poppler "0.86.0" COMPONENTS Qt5)
set_package_properties("Poppler" PROPERTIES
        TYPE RECOMMENDED
        PURPOSE "Support for PDF files in okular.")

add_definitions(-DQT_USE_QSTRINGBUILDER)
add_definitions(-DTRANSLATION_DOMAIN="okular")
add_definitions(-DQT_NO_URL_CAST_FROM_STRING)
add_definitions(-DQT_DEPRECATED_WARNINGS_SINCE=0x050C00)
add_definitions(-DKF_DEPRECATED_WARNINGS_SINCE=0x054400)

include_directories(${CMAKE_CURRENT_SOURCE_DIR} ${PHONON_INCLUDES} core/synctex ${CMAKE_BINARY_DIR}/core)

if(BUILD_MOBILE)
    add_subdirectory( mobile )
endif()
option(BUILD_COVERAGE "Build the project with gcov support" OFF)

if(BUILD_COVERAGE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-arcs -ftest-coverage")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -lgcov")
endif()

add_subdirectory( icons ) # an own directory for multi-size icons retrieved by KIconThemeLoader
add_subdirectory( part )
if(BUILD_DESKTOP)
    add_subdirectory( shell )
endif()
add_subdirectory( generators )
if(BUILD_TESTING)
   add_subdirectory( autotests )
endif()

if(KF5DocTools_FOUND)
    add_subdirectory(doc)
endif()

include(OkularConfigureChecks.cmake)

if(NOT WIN32)
 set(MATH_LIB m)
else(NOT WIN32)
 set(MATH_LIB)
endif(NOT WIN32)

# okularcore

set(okularcore_SRCS
   core/action.cpp
   core/annotations.cpp
   core/area.cpp
   core/audioplayer.cpp
   core/bookmarkmanager.cpp
   core/chooseenginedialog.cpp
   core/document.cpp
   core/documentcommands.cpp
   core/fontinfo.cpp
   core/form.cpp
   core/generator.cpp
   core/generator_p.cpp
   core/misc.cpp
   core/movie.cpp
   core/observer.cpp
   core/debug.cpp
   core/page.cpp
   core/pagecontroller.cpp
   core/pagesize.cpp
   core/pagetransition.cpp
   core/rotationjob.cpp
   core/scripter.cpp
   core/sound.cpp
   core/sourcereference.cpp
   core/textdocumentgenerator.cpp
   core/textdocumentsettings.cpp
   core/textpage.cpp
   core/tilesmanager.cpp
   core/utils.cpp
   core/view.cpp
   core/fileprinter.cpp
   core/printoptionswidget.cpp
   core/signatureutils.cpp
   core/script/event.cpp
   core/synctex/synctex_parser.c
   core/synctex/synctex_parser_utils.c
)
qt5_add_resources(okularcore_SRCS
   core/script/builtin.qrc
)

ki18n_wrap_ui(okularcore_SRCS
   part/textdocumentsettings.ui
)

install( FILES
           core/action.h
           core/annotations.h
           core/area.h
           core/document.h
           core/fontinfo.h
           core/form.h
           core/generator.h
           core/global.h
           core/page.h
           core/pagesize.h
           core/pagetransition.h
           core/signatureutils.h
           core/sound.h
           core/sourcereference.h
           core/textdocumentgenerator.h
           core/textdocumentsettings.h
           core/textpage.h
           core/tile.h
           core/utils.h
           core/fileprinter.h
           core/printoptionswidget.h
           core/observer.h
           ${CMAKE_CURRENT_BINARY_DIR}/core/version.h
           ${CMAKE_CURRENT_BINARY_DIR}/core/okularcore_export.h
           ${CMAKE_CURRENT_BINARY_DIR}/settings_core.h
         DESTINATION ${KDE_INSTALL_INCLUDEDIR}/okular/core COMPONENT Devel)

install( FILES
           interfaces/configinterface.h
           interfaces/guiinterface.h
           interfaces/printinterface.h
           interfaces/saveinterface.h
           interfaces/viewerinterface.h
         DESTINATION ${KDE_INSTALL_INCLUDEDIR}/okular/interfaces COMPONENT Devel)

ki18n_wrap_ui(okularcore_SRCS
   core/chooseenginewidget.ui
)

kconfig_add_kcfg_files(okularcore_SRCS GENERATE_MOC conf/settings_core.kcfgc)

add_library(okularcore SHARED ${okularcore_SRCS})
generate_export_header(okularcore BASE_NAME okularcore EXPORT_FILE_NAME "${CMAKE_CURRENT_BINARY_DIR}/core/okularcore_export.h")

if (ANDROID)
    set(fileName ${CMAKE_BINARY_DIR}/Okular5Core-android-dependencies.xml)
    file(WRITE "${fileName}" "<rules><dependencies><lib name='Okular5Core'><depends>\n"
                                "<bundled file='${KDE_INSTALL_PLUGINDIR}/okular/generators' />\n"
                             "</depends></lib></dependencies></rules>\n")
    install(FILES ${fileName} DESTINATION ${KDE_INSTALL_LIBDIR})
endif()


# Special handling for linking okularcore on OSX/Apple
IF(APPLE)
    SET(OKULAR_IOKIT "-framework IOKit" CACHE STRING "Apple IOKit framework")
ENDIF(APPLE)

# Extra library needed by imported synctex code on Windows
if(WIN32)
    set(SHLWAPI shlwapi)
endif(WIN32)

target_link_libraries(okularcore
PRIVATE
    ${OKULAR_IOKIT}
    ${SHLWAPI}
    KF5::Archive
    KF5::KIOCore
    KF5::KIOWidgets
    KF5::I18n
    KF5::ThreadWeaver
    KF5::Bookmarks
    Phonon::phonon4qt5
    ${MATH_LIB}
    ZLIB::ZLIB
PUBLIC  # these are included from the installed headers
    KF5::CoreAddons
    KF5::XmlGui
    KF5::ConfigGui
    Qt5::PrintSupport
    Qt5::Widgets
)


if (KF5Wallet_FOUND)
    target_link_libraries(okularcore PRIVATE KF5::Wallet)
endif()
if (KF5JS_FOUND)
    target_sources(okularcore PRIVATE
            core/script/executor_kjs.cpp
            core/script/kjs_app.cpp
            core/script/kjs_console.cpp
            core/script/kjs_data.cpp
            core/script/kjs_display.cpp
            core/script/kjs_document.cpp
            core/script/kjs_field.cpp
            core/script/kjs_fullscreen.cpp
            core/script/kjs_field.cpp
            core/script/kjs_spell.cpp
            core/script/kjs_util.cpp
            core/script/kjs_event.cpp
            core/script/kjs_ocg.cpp
    )
    target_link_libraries(okularcore PRIVATE KF5::JS KF5::JSApi)
endif()

set_target_properties(okularcore PROPERTIES VERSION 9.0.0 SOVERSION 9 OUTPUT_NAME Okular5Core EXPORT_NAME Core)

install(TARGETS okularcore EXPORT Okular5Targets ${KDE_INSTALL_TARGETS_DEFAULT_ARGS})

install(FILES conf/okular.kcfg DESTINATION ${KDE_INSTALL_KCFGDIR})
install(FILES conf/okular_core.kcfg DESTINATION ${KDE_INSTALL_KCFGDIR})
install(FILES core/okularGenerator.desktop DESTINATION ${KDE_INSTALL_KSERVICETYPES5DIR})

if(BUILD_DESKTOP)
  # okularpart
  set(okularpart_SRCS
    part/preferencesdialog.cpp
    part/dlgaccessibility.cpp
    part/dlgdebug.cpp
    part/dlgeditor.cpp
    part/dlggeneral.cpp
    part/dlgannotations.cpp
    part/dlgperformance.cpp
    part/dlgpresentation.cpp
    part/editannottooldialog.cpp
    part/editdrawingtooldialog.cpp
    part/widgetannottools.cpp
    part/widgetconfigurationtoolsbase.cpp
    part/widgetdrawingtools.cpp
    part/part.cpp
    part/xmlgui_helper.cpp
    part/extensions.cpp
    part/embeddedfilesdialog.cpp
    part/actionbar.cpp
    part/annotationactionhandler.cpp
    part/annotwindow.cpp
    part/annotationmodel.cpp
    part/annotationpopup.cpp
    part/annotationpropertiesdialog.cpp
    part/annotationproxymodels.cpp
    part/annotationtools.cpp
    part/annotationwidgets.cpp
    part/bookmarklist.cpp
    part/certificateviewer.cpp
    part/colormodemenu.cpp
    part/cursorwraphelper.cpp
    part/debug_ui.cpp
    part/drawingtoolactions.cpp
    part/fileprinterpreview.cpp
    part/findbar.cpp
    part/formwidgets.cpp
    part/guiutils.cpp
    part/ktreeviewsearchline.cpp
    part/latexrenderer.cpp
    part/minibar.cpp
    part/okmenutitle.cpp
    part/pageitemdelegate.cpp
    part/pagepainter.cpp
    part/pagesizelabel.cpp
    part/pageviewannotator.cpp
    part/pageviewmouseannotation.cpp
    part/pageview.cpp
    part/magnifierview.cpp
    part/pageviewutils.cpp
    part/presentationsearchbar.cpp
    part/presentationwidget.cpp
    part/propertiesdialog.cpp
    part/revisionviewer.cpp
    part/searchlineedit.cpp
    part/searchwidget.cpp
    part/sidebar.cpp
    part/side_reviews.cpp
    part/snapshottaker.cpp
    part/thumbnaillist.cpp
    part/toc.cpp
    part/tocmodel.cpp
    part/toggleactionmenu.cpp
    part/videowidget.cpp
    part/layers.cpp
    part/signatureguiutils.cpp
    part/signaturepropertiesdialog.cpp
    part/signaturemodel.cpp
    part/signaturepanel.cpp
  )

if (Qt5TextToSpeech_FOUND)
   set(okularpart_SRCS ${okularpart_SRCS}
      part/tts.cpp)
endif()

kconfig_add_kcfg_files(okularpart_SRCS GENERATE_MOC conf/settings.kcfgc)

add_library(okularpart SHARED ${okularpart_SRCS})
generate_export_header(okularpart BASE_NAME okularpart)

target_link_libraries(okularpart okularcore
    ${MATH_LIB}
    Qt5::Svg
    Qt5::DBus
    Phonon::phonon4qt5
    KF5::Archive
    KF5::Bookmarks
    KF5::I18n
    KF5::IconThemes
    KF5::ItemViews
    KF5::KIOCore
    KF5::KIOFileWidgets
    KF5::KIOWidgets
    KF5::Parts
    KF5::Solid
    KF5::WindowSystem
    KF5::TextWidgets
)

if(KF5Wallet_FOUND)
    target_link_libraries(okularpart KF5::Wallet)
endif()

if (KF5Purpose_FOUND)
    target_link_libraries(okularpart KF5::PurposeWidgets)
endif()

set_target_properties(okularpart PROPERTIES PREFIX "")

if (Qt5TextToSpeech_FOUND)
   target_link_libraries(okularpart Qt5::TextToSpeech)
endif()
install(TARGETS okularpart DESTINATION ${KDE_INSTALL_PLUGINDIR})
endif()


########### install files ###############

install(FILES okular.upd DESTINATION ${KDE_INSTALL_KCONFUPDATEDIR})

install( FILES okular_part.desktop  DESTINATION  ${KDE_INSTALL_KSERVICES5DIR} )
install( FILES part/part.rc part/part-viewermode.rc DESTINATION ${KDE_INSTALL_KXMLGUI5DIR}/okular )

if (${ECM_VERSION} STRGREATER "5.58.0")
    install(FILES okular.categories  DESTINATION  ${KDE_INSTALL_LOGGINGCATEGORIESDIR})
else()
    install(FILES okular.categories  DESTINATION ${KDE_INSTALL_CONFDIR})
endif()

########### cmake files #################

set(CMAKECONFIG_INSTALL_DIR "${KDE_INSTALL_CMAKEPACKAGEDIR}/Okular5")
configure_package_config_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/Okular5Config.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/Okular5Config.cmake"
  INSTALL_DESTINATION  ${CMAKECONFIG_INSTALL_DIR}
  PATH_VARS  INCLUDE_INSTALL_DIR CMAKE_INSTALL_PREFIX
)

install(FILES
  "${CMAKE_CURRENT_BINARY_DIR}/Okular5Config.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/Okular5ConfigVersion.cmake"
  DESTINATION "${CMAKECONFIG_INSTALL_DIR}"
  COMPONENT Devel
)

install(EXPORT Okular5Targets DESTINATION "${CMAKECONFIG_INSTALL_DIR}" FILE Okular5Targets.cmake NAMESPACE Okular::)

########### summary #################

feature_summary(WHAT ALL FATAL_ON_MISSING_REQUIRED_PACKAGES)

message("-- Building Desktop User Interface: ${BUILD_DESKTOP}")
message("-- Building Mobile Interface: ${BUILD_MOBILE}")
message("")
