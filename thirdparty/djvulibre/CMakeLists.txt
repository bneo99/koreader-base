project(djvulibre)
cmake_minimum_required(VERSION 3.5.1)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_LIST_DIR}/../cmake_modules")
include("koreader_thirdparty_common")
include("koreader_thirdparty_git")

enable_language(C CXX)

assert_var_defined(CC)
assert_var_defined(CFLAGS)
assert_var_defined(CXX)
assert_var_defined(CXXFLAGS)
assert_var_defined(LDFLAGS)
assert_var_defined(CHOST)

ep_get_source_dir(SOURCE_DIR)

# fix build error due to -Werror under Fedora 26 (and potentially other systems)
set(CFLAGS "${CFLAGS} -Wno-error")

set(CFG_ENV_VAR "CC=\"${CC}\" CXX=\"${CXX}\" CFLAGS=\"${CFLAGS}\" CXXFLAGS=\"${CXXFLAGS}\" LDFLAGS=\"${LDFLAGS}\"")
set(CFG_OPTS "-q --disable-desktopfiles --disable-static --enable-shared --disable-xmltools --disable-largefile --without-jpeg --without-tiff -host=\"${CHOST}\"")
set(CFG_CMD1 sh -c "${CFG_ENV_VAR} ${SOURCE_DIR}/configure ${CFG_OPTS}")

if(DEFINED ENV{ANDROID})
    set(CFG_CMD2 sh -c "${ISED} 's|version_type=none|version_type=linux|' libtool")
    set(CFG_CMD3 sh -c "${ISED} 's|need_lib_prefix=no|need_lib_prefix=yes|' libtool")
    set(CFG_CMD4 sh -c "${ISED} 's|need_version=no|need_version=yes|' libtool")
    set(CFG_CMD5 sh -c "${ISED} 's|library_names_spec=.*|library_names_spec=\"\\\\\$libname\\\\\$release\\\\\$shared_ext\\\\\$versuffix \\\\\$libname\\\\\$release\\\\\$shared_ext\\\\\$major \\\\\$libname\\\\\$shared_ext\"|' libtool")
    set(CFG_CMD6 sh -c "${ISED} 's|soname_spec=.*|soname_spec=\"\\\\\$libname\\\\\$release\\\\\$shared_ext\\\\\$major\"|' libtool")
endif()

# Technically we don't need tools on any platform but they only cause issues on Android
if(DEFINED ENV{ANDROID})
    set(PATCH_CMD1 sh -c "${ISED} \"s|DIST_SUBDIRS = libdjvu.*|DIST_SUBDIRS = libdjvu|g\" Makefile")
    set(PATCH_CMD2 sh -c "${ISED} \"s|SUBDIRS = libdjvu tools share \\$(am__append_1) \\$(am__append_2)|SUBDIRS = libdjvu share \\$(am__append_1) \\$(am__append_2)|g\" Makefile")
    set(PATCH_CMD3 "${KO_PATCH} ${CMAKE_CURRENT_SOURCE_DIR}/android.patch")
endif()

# Never *ever* call setlocale, for any reason.
# Most of our target platforms have custom/broken/incomplete locales, don't screw with it.
# And this runs in our main thread, so this is Very Bad(TM).
set(PATCH_CMD4 "${KO_PATCH} ${CMAKE_CURRENT_SOURCE_DIR}/djvulibre-no-locale-mangling.patch")

ko_write_gitclone_script(
    GIT_CLONE_SCRIPT_FILENAME
    https://gitlab.com/koreader/djvulibre.git
    6a1e5ba1c9ef81c205a4b270c3f121a1e106f4fc
    ${SOURCE_DIR}
)

include(ExternalProject)
ExternalProject_Add(
    ${PROJECT_NAME}
    DOWNLOAD_COMMAND ${CMAKE_COMMAND} -P ${GIT_CLONE_SCRIPT_FILENAME}
    PATCH_COMMAND NOCONFIGURE=1 ./autogen.sh
    BUILD_IN_SOURCE 1
    CONFIGURE_COMMAND
        COMMAND ${CFG_CMD1}
        COMMAND ${CFG_CMD2} COMMAND ${CFG_CMD3} COMMAND ${CFG_CMD4} COMMAND ${CFG_CMD5} COMMAND ${CFG_CMD6}
        COMMAND ${PATCH_CMD1} COMMAND ${PATCH_CMD2} COMMAND ${PATCH_CMD3} COMMAND ${PATCH_CMD4}
    BUILD_COMMAND ${KO_MAKE_RECURSIVE} -j${PARALLEL_JOBS} SUBDIRS_FIRST=libdjvu
    # skip install
    INSTALL_COMMAND ""
)
