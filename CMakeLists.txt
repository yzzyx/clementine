cmake_minimum_required(VERSION 2.6)
cmake_policy(SET CMP0011 OLD)

include(CheckCXXCompilerFlag)
include(FindPkgConfig)
include(cmake/Summary.cmake)
include(cmake/Version.cmake)
include(cmake/Deb.cmake)
include(cmake/Rpm.cmake)
include(cmake/SpotifyVersion.cmake)
include(cmake/OptionalSource.cmake)
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

if (CMAKE_CXX_COMPILER MATCHES ".*clang")
  set(CMAKE_COMPILER_IS_CLANGXX 1)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-uninitialized")
endif ()

set(CMAKE_REQUIRED_FLAGS "-std=c++0x")
check_cxx_source_compiles(
    "#include <unordered_map>
     int main() {
       std::unordered_map<int, int> m;
       return 0;
     }
    "
    USE_STD_UNORDERED_MAP)
check_cxx_source_compiles(
    "int main() {
       [](){}();
     }
    "
    HAVE_LAMBDAS)
unset(CMAKE_REQUIRED_FLAGS)


if (UNIX AND NOT APPLE)
  set(LINUX 1)
endif (UNIX AND NOT APPLE)

find_package(Qt4 4.5.0 REQUIRED QtCore QtGui QtOpenGL QtSql QtNetwork QtXml)

if(NOT APPLE)
  find_package(Qt4 COMPONENTS QtWebKit)
endif(NOT APPLE)

# Find Qt's lconvert binary.  Try qt's binary dir first, fall back to looking in PATH
find_program(QT_LCONVERT_EXECUTABLE NAMES lconvert lconvert-qt4 PATHS ${QT_BINARY_DIR} NO_DEFAULT_PATH)
find_program(QT_LCONVERT_EXECUTABLE NAMES lconvert lconvert-qt4)

if(APPLE)
  if(NOT QT_MAC_USE_COCOA)
    message(FATAL_ERROR "Cocoa support is required")
  endif(NOT QT_MAC_USE_COCOA)
endif(APPLE)

find_package(OpenGL REQUIRED)
find_package(Boost REQUIRED)
find_package(Gettext REQUIRED)
find_package(PkgConfig REQUIRED)
find_package(Protobuf REQUIRED)
find_package(FFTW3)

pkg_check_modules(CDIO libcdio)
pkg_check_modules(CHROMAPRINT libchromaprint)
pkg_check_modules(GIO gio-2.0)
pkg_check_modules(GLIB glib-2.0)
pkg_check_modules(GOBJECT gobject-2.0)
pkg_check_modules(GSTREAMER gstreamer-0.10)
pkg_check_modules(GSTREAMER_APP gstreamer-app-0.10)
pkg_check_modules(GSTREAMER_BASE gstreamer-base-0.10)
pkg_check_modules(GSTREAMER_CDDA gstreamer-cdda-0.10)
pkg_check_modules(GSTREAMER_TAG gstreamer-tag-0.10)
pkg_check_modules(INDICATEQT indicate-qt)
pkg_check_modules(LIBGPOD libgpod-1.0>=0.7.92)
pkg_check_modules(LIBMTP libmtp>=1.0)
pkg_check_modules(LIBMYGPO_QT libmygpo-qt>=1.0.7)
pkg_check_modules(LIBXML libxml-2.0)
pkg_check_modules(QCA qca2)
pkg_check_modules(QJSON REQUIRED QJson)
pkg_check_modules(SPOTIFY libspotify>=12.1.45)
pkg_check_modules(TAGLIB REQUIRED taglib>=1.6)

if (WIN32)
  find_package(ZLIB REQUIRED)
  find_library(QTSPARKLE_LIBRARIES qtsparkle)
endif (WIN32)

find_library(LASTFM_LIBRARIES lastfm)
find_path(LASTFM_INCLUDE_DIRS lastfm/ws.h)
find_path(LASTFM1_INCLUDE_DIRS lastfm/Track.h)

find_path(SPARSEHASH_INCLUDE_DIRS google/sparsetable)

# Google Drive support needs Taglib 1.8, but this version isn't in old Ubuntu
# distros.  If the user seems to want Drive support (ie. they have sparsehash
# installed and haven't disabled drive), and has an old taglib, compile our
# internal one and use that instead.
option(USE_BUILTIN_TAGLIB "If the system's version of Taglib is too old for Google Drive support, compile our builtin version instead" ON)
if (USE_BUILTIN_TAGLIB AND
    (NOT "${ENABLE_GOOGLE_DRIVE}" STREQUAL "OFF") AND
    SPARSEHASH_INCLUDE_DIRS AND
    TAGLIB_VERSION VERSION_LESS 1.8)
  set(TAGLIB_VERSION 1.8)
  set(TAGLIB_INCLUDE_DIRS "${CMAKE_BINARY_DIR}/3rdparty/taglib/headers/taglib/;${CMAKE_BINARY_DIR}/3rdparty/taglib/headers/")
  set(TAGLIB_LIBRARY_DIRS "")
  set(TAGLIB_LIBRARIES tag)
  set(TAGLIB_HAS_OPUS ON)
  add_subdirectory(3rdparty/taglib)
else()
  set(CMAKE_REQUIRED_INCLUDES "${TAGLIB_INCLUDE_DIRS}")
  set(CMAKE_REQUIRED_LIBRARIES "${TAGLIB_LIBRARIES}")
  check_cxx_source_compiles("#include <opusfile.h>
      int main() { char *s; TagLib::Ogg::Opus::File opusfile(s); return 0;}" TAGLIB_HAS_OPUS)
  set(CMAKE_REQUIRED_INCLUDES)
  set(CMAKE_REQUIRED_LIBRARIES)
endif()

if(LASTFM_INCLUDE_DIRS AND LASTFM1_INCLUDE_DIRS)
  set(HAVE_LIBLASTFM1 ON)
endif()

if (APPLE)
  find_library(GROWL Growl)
  find_library(SPARKLE Sparkle)

  find_library(SPOTIFY libspotify)
  if (SPOTIFY)
    set (SPOTIFY_FOUND ON)
    set (SPOTIFY_INCLUDE_DIRS ${SPOTIFY})
    set (SPOTIFY_LIBRARIES ${SPOTIFY})
  endif (SPOTIFY)

  add_subdirectory(3rdparty/SPMediaKeyTap)
  set(SPMEDIAKEYTAP_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/SPMediaKeyTap)
  set(SPMEDIAKEYTAP_LIBRARIES SPMediaKeyTap)
endif (APPLE)

find_package(Threads)

if(${CMAKE_BUILD_TYPE} MATCHES "Release")
  add_definitions(-DNDEBUG)
  add_definitions(-DQT_NO_DEBUG_OUTPUT)
endif(${CMAKE_BUILD_TYPE} MATCHES "Release")

# Set up definitions and paths
add_definitions(${QT_DEFINITIONS})
link_directories(${TAGLIB_LIBRARY_DIRS})
link_directories(${QJSON_LIBRARY_DIRS})
link_directories(${GSTREAMER_LIBRARY_DIRS})

# Don't try to use webkit if their include directories couldn't be found.
if (NOT QT_QTWEBKIT_INCLUDE_DIR)
  set (QT_USE_QTWEBKIT 0)
endif (NOT QT_QTWEBKIT_INCLUDE_DIR)

include_directories(${Boost_INCLUDE_DIRS})
include_directories(${TAGLIB_INCLUDE_DIRS})
include_directories(${QJSON_INCLUDE_DIRS})
include_directories(${GSTREAMER_INCLUDE_DIRS})
include_directories(${GLIB_INCLUDE_DIRS})
include_directories(${GLIBCONFIG_INCLUDE_DIRS})
include_directories(${LIBXML_INCLUDE_DIRS})

if (WIN32)
  # RC compiler
  string(REPLACE "gcc" "windres" CMAKE_RC_COMPILER_INIT ${CMAKE_C_COMPILER})
  enable_language(RC)
  SET(CMAKE_RC_COMPILE_OBJECT "<CMAKE_RC_COMPILER> -O coff -o <OBJECT> <SOURCE> -I ${CMAKE_SOURCE_DIR}/dist/windows")
endif(WIN32)

add_definitions(-DQT_NO_CAST_TO_ASCII -DQT_STRICT_ITERATORS)

# Translations stuff
find_program(GETTEXT_XGETTEXT_EXECUTABLE xgettext PATHS /target/bin)
if(NOT GETTEXT_XGETTEXT_EXECUTABLE)
  message(FATAL_ERROR "Could not find xgettext executable")
endif(NOT GETTEXT_XGETTEXT_EXECUTABLE)
find_program(GETTEXT_MSGMERGE_EXECUTABLE msgmerge PATHS /target/bin)
if(NOT GETTEXT_MSGMERGE_EXECUTABLE)
  message(FATAL_ERROR "Could not find msgmerge executable")
endif(NOT GETTEXT_MSGMERGE_EXECUTABLE)
find_program(GETTEXT_MSGFMT_EXECUTABLE msgfmt PATHS /target/bin)
if(NOT GETTEXT_MSGFMT_EXECUTABLE)
  message(FATAL_ERROR "Could not find msgfmt executable")
endif(NOT GETTEXT_MSGFMT_EXECUTABLE)

# Optional bits
if(WIN32)
  option(ENABLE_WIN32_CONSOLE "Show the windows console even outside Debug mode" OFF)
endif(WIN32)

optional_component(BREAKPAD OFF "Crash reporting")

optional_component(GOOGLE_DRIVE ON "Google Drive support"
  DEPENDS "Google sparsehash" SPARSEHASH_INCLUDE_DIRS
  DEPENDS "Taglib 1.8" "TAGLIB_VERSION VERSION_GREATER 1.7.999"
)

optional_component(UBUNTU_ONE ON "Ubuntu One file support"
  DEPENDS "Google sparsehash" SPARSEHASH_INCLUDE_DIRS
  DEPENDS "Taglib 1.8" "TAGLIB_VERSION VERSION_GREATER 1.7.999"
)

optional_component(DROPBOX ON "Dropbox support"
  DEPENDS "Google sparsehash" SPARSEHASH_INCLUDE_DIRS
  DEPENDS "Taglib 1.8" "TAGLIB_VERSION VERSION_GREATER 1.7.999"
)

optional_component(SKYDRIVE ON "Skydrive support"
  DEPENDS "Google sparsehash" SPARSEHASH_INCLUDE_DIRS
  DEPENDS "Taglib 1.8" "TAGLIB_VERSION VERSION_GREATER 1.7.999"
)

optional_component(BOX ON "Box support"
  DEPENDS "Google sparsehash" SPARSEHASH_INCLUDE_DIRS
  DEPENDS "Taglib 1.8" "TAGLIB_VERSION VERSION_GREATER 1.7.999"
)

optional_component(AUDIOCD ON "Devices: Audio CD support"
  DEPENDS "libcdio" CDIO_FOUND
)

optional_component(LIBGPOD ON "Devices: iPod classic support"
  DEPENDS "libgpod" LIBGPOD_FOUND
)

optional_component(GIO ON "Devices: GIO device backend"
  DEPENDS "libgio" GIO_FOUND
  DEPENDS "Linux or Windows" "NOT APPLE"
)

optional_component(LIBMTP ON "Devices: MTP support"
  DEPENDS "libmtp" LIBMTP_FOUND
)

optional_component(SOUNDMENU ON "Gnome sound menu integration"
  DEPENDS "indicate-qt" INDICATEQT_FOUND
)

optional_component(LIBLASTFM ON "Last.fm support"
  DEPENDS "liblastfm" LASTFM_LIBRARIES LASTFM_INCLUDE_DIRS
)

optional_component(DBUS ON "D-Bus support"
  DEPENDS "Linux" LINUX
)

optional_component(WIIMOTEDEV ON "Wiimote support"
  DEPENDS "D-Bus support" HAVE_DBUS
)

optional_component(DEVICEKIT ON "Devices: DeviceKit backend"
  DEPENDS "D-Bus support" HAVE_DBUS
)

optional_component(SPOTIFY_BLOB ON "Spotify support: non-GPL binary helper"
  DEPENDS "protobuf" PROTOBUF_FOUND PROTOBUF_PROTOC_EXECUTABLE
  DEPENDS "libspotify" SPOTIFY_FOUND
)

optional_component(MOODBAR ON "Moodbar support"
  DEPENDS "fftw3" FFTW3_FOUND
)

optional_component(SPARKLE ON "Sparkle integration"
  DEPENDS "Mac OS X" APPLE
  DEPENDS "Sparkle" SPARKLE
)

optional_component(VISUALISATIONS ON "Visualisations")

if(NOT HAVE_SPOTIFY_BLOB AND NOT QCA_FOUND)
  message(FATAL_ERROR "Either QCA must be available or the non-GPL Spotify "
          "code must be compiled in")
elif(QCA_FOUND)
  set(HAVE_SPOTIFY_DOWNLOADER ON)
endif()

# Find DBus if it's enabled
if (HAVE_DBUS)
  find_package(Qt4 REQUIRED QtDbus)
endif ()

# We can include the Qt definitions now
include(${QT_USE_FILE})

# Remove GLU and GL from the link line - they're not really required
# and don't exist on my mingw toolchain
list(REMOVE_ITEM QT_LIBRARIES "-lGLU -lGL")

option(BUNDLE_PROJECTM_PRESETS "Install Clementine's own copies of libprojectm presets - disable this if you want to use a system package instead" ON)

if(ENABLE_VISUALISATIONS)
  # When/if upstream accepts our patches then these options can be used to link
  # to system installed projectM instead.
  option(USE_SYSTEM_PROJECTM "Don't set this option unless your system projectM library has been compiled with the Clementine patches in 3rdparty" OFF)
  if(USE_SYSTEM_PROJECTM)
    pkg_check_modules(LIBPROJECTM libprojectM)
  else(USE_SYSTEM_PROJECTM)
    add_subdirectory(3rdparty/libprojectm)
    set(LIBPROJECTM_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/libprojectm)
    set(LIBPROJECTM_LIBRARIES projectM)
  endif(USE_SYSTEM_PROJECTM)
endif(ENABLE_VISUALISATIONS)

# We compile our own qsqlite3 by default now, because in Qt 4.7 the symbols we
# need from it are private.
option(STATIC_SQLITE "Compile and use a static sqlite3 library" ON)

set(HAVE_STATIC_SQLITE ${STATIC_SQLITE})
if(STATIC_SQLITE)
  message(STATUS "Building static qsqlite plugin")
  add_subdirectory(3rdparty/qsqlite)
  include_directories("3rdparty/qsqlite")
else()
  if (NOT I_HATE_MY_USERS)
    message(FATAL_ERROR "No, really, Clementine needs sqlite to be linked "
        "statically.  If it's not, Clementine won't be able to resolve the "
        "functions it needs to register full text search support.  Search will "
        "be really slow and users will be unhappy.  If you're packaging "
        "Clementine for a distribution and you really hate your users and want "
        "them to have a bad time please rerun cmake with -DI_HATE_MY_USERS=ON")
  else()
    if (NOT MY_USERS_WILL_SUFFER_BECAUSE_OF_ME)
      message(FATAL_ERROR "So you really hate your users and you want to "
          "create a crippled package that doesn't work properly?  Please send "
          "an email to the devs and help us understand why.")
    endif()
  endif()
endif(STATIC_SQLITE)

# When/if upstream accepts our patches then these options can be used to link
# to system installed qtsingleapplication instead.
option(USE_SYSTEM_QTSINGLEAPPLICATION "Don't set this option unless your system QtSingleApplication library has been compiled with the Clementine patches in 3rdparty" OFF)
if(USE_SYSTEM_QTSINGLEAPPLICATION)
  find_path(QTSINGLEAPPLICATION_INCLUDE_DIRS qtsingleapplication.h PATH_SUFFIXES QtSolutions)
  find_library(QTSINGLEAPPLICATION_LIBRARIES QtSolutions_SingleApplication-2.6)
  find_library(QTSINGLECOREAPPLICATION_LIBRARIES QtSolutions_SingleCoreApplication-2.6)
else(USE_SYSTEM_QTSINGLEAPPLICATION)
  add_subdirectory(3rdparty/qtsingleapplication)
  set(QTSINGLEAPPLICATION_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/qtsingleapplication)
  set(QTSINGLEAPPLICATION_LIBRARIES qtsingleapplication)
endif(USE_SYSTEM_QTSINGLEAPPLICATION)

# QtIoCompressor isn't patched, so we can use a system version if it's
# available
find_path(QTIOCOMPRESSOR_INCLUDE_DIRS qtiocompressor.h PATH_SUFFIXES QtSolutions)
find_library(QTIOCOMPRESSOR_LIBRARIES QtSolutions_IOCompressor-2.3)
if(NOT QTIOCOMPRESSOR_INCLUDE_DIRS OR NOT QTIOCOMPRESSOR_LIBRARIES)
  add_subdirectory(3rdparty/qtiocompressor)
  set(QTIOCOMPRESSOR_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/qtiocompressor)
  set(QTIOCOMPRESSOR_LIBRARIES qtiocompressor)
endif(NOT QTIOCOMPRESSOR_INCLUDE_DIRS OR NOT QTIOCOMPRESSOR_LIBRARIES)

# When/if upstream accepts our or reimplement our patches then these options can be
# used to link to system installed qxt instead.
option(USE_SYSTEM_QXT "Don't set this option unless your system Qxt library has been compiled with the Clementine patches in 3rdparty" OFF)
if (USE_SYSTEM_QXT)
  find_path(QXTCORE_INCLUDE_DIRS qxtglobal.h PATH_SUFFIXES QxtCore)
  find_path(QXTGUI_INCLUDE_DIRS qxtglobalshortcut.h PATH_SUFFIXES QxtGui)
  set(QXT_INCLUDE_DIRS ${QXTCORE_INCLUDE_DIRS} ${QXTGUI_INCLUDE_DIRS})
  # We only need its header. We don't need to link to QxtCore.
  find_library(QXT_LIBRARIES QxtGui)
else (USE_SYSTEM_QXT)
  add_definitions(-DQXT_STATIC -DBUILD_QXT_GUI -DBUILD_QXT_CORE)
  set(QXT_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/qxt)
  set(QXT_LIBRARIES qxt)
  if (NOT APPLE)
    add_subdirectory(3rdparty/qxt)
  endif (NOT APPLE)
endif (USE_SYSTEM_QXT)

# Use system libechonest if it's available
find_path(ECHONEST_INCLUDE_DIRS echonest/echonest_export.h)
find_library(ECHONEST_LIBRARIES echonest)
if(NOT ECHONEST_INCLUDE_DIRS OR NOT ECHONEST_LIBRARIES)
  add_subdirectory(3rdparty/libechonest)
  set(ECHONEST_INCLUDE_DIRS ${CMAKE_BINARY_DIR}/3rdparty/libechonest)
  set(ECHONEST_LIBRARIES echonest)
endif(NOT ECHONEST_INCLUDE_DIRS OR NOT ECHONEST_LIBRARIES)

# Use system gmock if it's available
# We need to look for both gmock and gtest
find_path(GMOCK_INCLUDE_DIRS gmock/gmock.h)
find_library(GMOCK_LIBRARIES gmock)
if(GMOCK_INCLUDE_DIRS)
  find_path(GTEST_INCLUDE_DIRS gtest/gtest.h)
  find_library(GTEST_LIBRARIES gtest)
  if(GTEST_INCLUDE_DIRS)
    set(USE_SYSTEM_GMOCK 1)
    set(GMOCK_LIBRARIES ${GMOCK_LIBRARIES} ${GTEST_LIBRARIES})
  endif(GTEST_INCLUDE_DIRS)
endif(GMOCK_INCLUDE_DIRS)

# Use system sha2 if it's available
find_path(SHA2_INCLUDE_DIRS sha2.h)
find_library(SHA2_LIBRARIES sha2)
if(NOT SHA2_INCLUDE_DIRS OR NOT SHA2_LIBRARIES)
  add_subdirectory(3rdparty/sha2)
  set(SHA2_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/sha2)
  set(SHA2_LIBRARIES sha2)
endif(NOT SHA2_INCLUDE_DIRS OR NOT SHA2_LIBRARIES)

# Use our 3rdparty chromaprint if a system one wasn't found
if(NOT CHROMAPRINT_FOUND)
  add_subdirectory(3rdparty/chromaprint)
  set(CHROMAPRINT_LIBRARIES chromaprint_p)
  set(CHROMAPRINT_INCLUDE_DIRS ${CMAKE_SOURCE_DIR}/3rdparty/chromaprint/src)

  if(WIN32)
    add_definitions(-DCHROMAPRINT_NODLL)
  endif(WIN32)
endif(NOT CHROMAPRINT_FOUND)

# Use the system libmygpo-qt if a recent enough version was found
if(LIBMYGPO_QT_FOUND)
  set(MYGPOQT_LIBRARIES ${LIBMYGPO_QT_LIBRARIES})
  set(MYGPOQT_INCLUDE_DIRS ${LIBMYGPO_QT_INCLUDE_DIRS})
else()
  add_definitions(-DMYGPO_EXPORT=)
  add_subdirectory(3rdparty/libmygpo-qt)
  set(MYGPOQT_LIBRARIES mygpo-qt)
  set(MYGPOQT_INCLUDE_DIRS ${CMAKE_SOURCE_DIR}/3rdparty/libmygpo-qt/)
endif()

# Qocoa
add_subdirectory(3rdparty/qocoa)

# Subdirectories
add_subdirectory(src)
if (WIN32)
  add_subdirectory(3rdparty/qtwin)
  add_subdirectory(3rdparty/tinysvcmdns)
endif (WIN32)
add_subdirectory(tests)
add_subdirectory(dist)
add_subdirectory(tools/ultimate_lyrics_parser)
add_subdirectory(ext/libclementine-common)
add_subdirectory(ext/libclementine-tagreader)
add_subdirectory(ext/clementine-tagreader)
add_subdirectory(ext/libclementine-remote)
add_subdirectory(ext/libclementine-spotifyblob)

option(WITH_DEBIAN OFF)
if(WITH_DEBIAN)
  add_subdirectory(debian)
endif(WITH_DEBIAN)

if(HAVE_BREAKPAD)
  add_subdirectory(3rdparty/google-breakpad)
endif(HAVE_BREAKPAD)

if(HAVE_SPOTIFY_BLOB)
  add_subdirectory(ext/clementine-spotifyblob)
endif(HAVE_SPOTIFY_BLOB)

if(HAVE_MOODBAR)
  add_subdirectory(gst/moodbar)
endif()

# Uninstall support
configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
  IMMEDIATE @ONLY)

add_custom_target(uninstall
  "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")

# Show a summary of what we have enabled
summary_show()
