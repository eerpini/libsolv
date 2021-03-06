PROJECT (libsolv)

CMAKE_MINIMUM_REQUIRED (VERSION 2.4)

OPTION (ENABLE_STATIC "Build a static version of the libraries?" OFF)
OPTION (DISABLE_SHARED "Do not build a shared version of the libraries?" OFF)

OPTION (ENABLE_PERL "Build the perl bindings?" OFF)
OPTION (ENABLE_PYTHON "Build the python bindings?" OFF)
OPTION (ENABLE_RUBY "Build the ruby bindings?" OFF)

OPTION (USE_VENDORDIRS "Install the bindings in vendor directories?" OFF)

OPTION (ENABLE_RPMDB "Build with rpm database support?" OFF)
OPTION (ENABLE_RPMDB_PUBKEY "Build with rpmdb pubkey support?" OFF)
OPTION (ENABLE_RPMDB_BYRPMHEADER "Build with rpmdb Header support?" OFF)
OPTION (ENABLE_RPMMD "Build with rpmmd repository support?" OFF)
OPTION (ENABLE_SUSEREPO "Build with suse repository support?" OFF)
OPTION (ENABLE_COMPS "Build with fedora comps support?" OFF)
OPTION (ENABLE_HELIXREPO "Build with helix repository support?" OFF)
OPTION (ENABLE_DEBIAN "Build with debian database/repository support?" OFF)
OPTION (ENABLE_MDKREPO "Build with mandriva/mageia repository support?" OFF)
OPTION (ENABLE_ARCHREPO "Build with archlinux repository support?" OFF)
OPTION (ENABLE_CUDFREPO "Build with cudf repository support?" OFF)

OPTION (ENABLE_LZMA_COMPRESSION "Build with lzma/xz compression support?" OFF)
OPTION (ENABLE_BZIP2_COMPRESSION "Build with bzip2 compression support?" OFF)

#IF(${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERISION} GREATER 2.4)
#ENDIF(${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERISION} GREATER 2.4)

IF (COMMAND cmake_policy)
  # escape preprocessor, see -DVERSION below
  CMAKE_POLICY (SET CMP0005 OLD)
ENDIF (COMMAND cmake_policy)

# Library
IF (DEFINED LIB)
  SET (LIB_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/${LIB}")
ELSE (DEFINED  LIB)
  IF (CMAKE_SIZEOF_VOID_P MATCHES "8")
    SET (LIB_SUFFIX "64")
  ENDIF (CMAKE_SIZEOF_VOID_P MATCHES "8")
  SET (LIB_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/lib${LIB_SUFFIX}")
ENDIF (DEFINED  LIB)
MESSAGE (STATUS "Libraries will be installed in ${LIB_INSTALL_DIR}")
SET (BIN_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/bin")

####################################################################
# CONFIGURATION                                                    #
####################################################################

# where to look first for cmake modules, before ${CMAKE_ROOT}/Modules/ is checked
SET (CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake/modules)
INSTALL( FILES ${CMAKE_MODULE_PATH}/FindLibSolv.cmake DESTINATION ${CMAKE_INSTALL_PREFIX}/share/cmake/Modules )

INCLUDE (${CMAKE_SOURCE_DIR}/VERSION.cmake)

SET (have_system x)

IF (FEDORA)
MESSAGE(STATUS "Building for Fedora")
ADD_DEFINITIONS (-DFEDORA)
SET (ENABLE_RPMDB ON)
SET (ENABLE_RPMMD ON)
SET (have_system ${have_system}x)
ENDIF (FEDORA)

IF (DEBIAN)
MESSAGE (STATUS "Building for Debian")
ADD_DEFINITIONS (-DDEBIAN)
SET (ENABLE_DEBIAN ON)
SET (have_system ${have_system}x)
ENDIF (DEBIAN)

IF (SUSE)
MESSAGE (STATUS "Building for SUSE")
ADD_DEFINITIONS (-DSUSE)
SET (ENABLE_RPMDB ON)
SET (ENABLE_RPMDB_PUBKEY ON)
SET (ENABLE_RPMDB_BYRPMHEADER ON)
SET (ENABLE_RPMMD ON)
SET (ENABLE_SUSEREPO ON)
SET (ENABLE_HELIXREPO ON)
SET (have_system ${have_system}x)
ENDIF (SUSE)

IF (ARCHLINUX)
MESSAGE (STATUS "Building for Archlinux")
ADD_DEFINITIONS (-DARCHLINUX)
SET (ENABLE_ARCHREPO ON)
SET (have_system ${have_system}x)
ENDIF (ARCHLINUX)

IF (MANDRIVA)
MESSAGE (STATUS "Building for Mandriva/Mageia")
ADD_DEFINITIONS (-DMANDRIVA)
SET (ENABLE_MDKREPO ON)
SET (have_system ${have_system}x)
ENDIF (MANDRIVA)

IF (${have_system} STREQUAL x)
    MESSAGE (STATUS "Building for no system")
    ADD_DEFINITIONS (-DNOSYSTEM)
ENDIF (${have_system} STREQUAL x)
IF (${have_system} STRGREATER xx)
    MESSAGE (FATAL_ERROR "Can only compile for one system type.")
ENDIF (${have_system} STRGREATER xx)

IF (ENABLE_ARCHREPO)
SET (ENABLE_LZMA_COMPRESSION ON)
ENDIF (ENABLE_ARCHREPO)

FIND_PACKAGE (EXPAT REQUIRED)
FIND_PACKAGE (ZLIB REQUIRED)
IF (ENABLE_LZMA_COMPRESSION)
FIND_PACKAGE (LZMA REQUIRED)
ENDIF (ENABLE_LZMA_COMPRESSION)
IF (ENABLE_BZIP2_COMPRESSION)
FIND_PACKAGE (BZip2 REQUIRED)
ENDIF (ENABLE_BZIP2_COMPRESSION)

IF (RPM5)
MESSAGE (STATUS "Enabling RPM 5 support")
ADD_DEFINITIONS (-DRPM5)
SET (ENABLE_RPMDB ON)
SET (ENABLE_RPMMD ON)
FIND_PACKAGE (PkgConfig REQUIRED)
PKG_CHECK_MODULES (RPM REQUIRED rpm)
INCLUDE_DIRECTORIES (${RPM_INCLUDE_DIRS})
ENDIF (RPM5)

IF (MULTI_SEMANTICS)
MESSAGE (STATUS "Enabling multi dist support")
ADD_DEFINITIONS (-DMULTI_SEMANTICS)
ENDIF (MULTI_SEMANTICS)

INCLUDE (CheckIncludeFile)
IF (ENABLE_RPMDB)
  FIND_LIBRARY (RPMDB_LIBRARY NAMES rpmdb)

  IF (NOT RPMDB_LIBRARY)
    FIND_LIBRARY (RPMDB_LIBRARY NAMES rpm)
  ENDIF (NOT RPMDB_LIBRARY)

  FIND_LIBRARY (RPMIO_LIBRARY NAMES rpmio)
  IF (RPMIO_LIBRARY)
    SET(RPMDB_LIBRARY ${RPMIO_LIBRARY} ${RPMDB_LIBRARY})
  ENDIF (RPMIO_LIBRARY)

  IF (RPM5)
    FIND_LIBRARY (RPMMISC_LIBRARY NAMES rpmmisc)
    IF (RPMMISC_LIBRARY)
      SET (RPMDB_LIBRARY ${RPMMISC_LIBRARY} ${RPMDB_LIBRARY})
    ENDIF (RPMMISC_LIBRARY)
  ENDIF (RPM5)

  IF (RPM5 OR FEDORA)
    FIND_LIBRARY (DB_LIBRARY NAMES db)
    IF (DB_LIBRARY)
      SET (RPMDB_LIBRARY ${DB_LIBRARY} ${RPMDB_LIBRARY})
    ENDIF (DB_LIBRARY)
  ENDIF (RPM5 OR FEDORA)
  CHECK_INCLUDE_FILE(rpm/db.h HAVE_RPM_DB_H)
  INCLUDE (CheckLibraryExists)
  CHECK_LIBRARY_EXISTS(rpmio pgpDigGetParams "" HAVE_PGPDIGGETPARAMS)
ENDIF (ENABLE_RPMDB)

INCLUDE (CheckFunctionExists)
INCLUDE (TestBigEndian)

CHECK_FUNCTION_EXISTS (strchrnul HAVE_STRCHRNUL)
CHECK_FUNCTION_EXISTS (fopencookie HAVE_FOPENCOOKIE)
CHECK_FUNCTION_EXISTS (funopen HAVE_FUNOPEN)
TEST_BIG_ENDIAN (WORDS_BIGENDIAN)

# should create config.h with #cmakedefine instead...
FOREACH (VAR HAVE_STRCHRNUL HAVE_FOPENCOOKIE HAVE_FUNOPEN WORDS_BIGENDIAN
  HAVE_RPM_DB_H HAVE_PGPDIGGETPARAMS
  ENABLE_RPMDB ENABLE_RPMDB_PUBKEY ENABLE_RPMMD ENABLE_SUSEREPO ENABLE_COMPS
  ENABLE_HELIXREPO ENABLE_MDKREPO ENABLE_ARCHREPO ENABLE_DEBIAN
  ENABLE_LZMA_COMPRESSION ENABLE_BZIP2_COMPRESSION)
  IF(${VAR})
    ADD_DEFINITIONS (-D${VAR}=1)
    SET (SWIG_FLAGS ${SWIG_FLAGS} -D${VAR})
  ENDIF (${VAR})
ENDFOREACH (VAR)

SET (PACKAGE "libsolv")
SET (VERSION "${LIBSOLV_MAJOR}.${LIBSOLV_MINOR}.${LIBSOLV_PATCH}")

ADD_DEFINITIONS (-D_FILE_OFFSET_BITS=64)
ADD_DEFINITIONS (-DVERSION=\\\"${VERSION}\\\")
CONFIGURE_FILE (src/solvversion.h.in src/solvversion.h)

SET (CPACK_PACKAGE_DESCRIPTION_SUMMARY "Package dependency solver library")
SET (CPACK_PACKAGE_VENDOR "SUSE")
SET (CPACK_PACKAGE_VERSION_MAJOR ${LIBSOLV_MAJOR})
SET (CPACK_PACKAGE_VERSION_MINOR ${LIBSOLV_MINOR})
SET (CPACK_PACKAGE_VERSION_PATCH ${LIBSOLV_PATCH})
SET (CPACK_GENERATOR "TBZ2")
SET (CPACK_SOURCE_GENERATOR "TBZ2")
SET (CPACK_SOURCE_PACKAGE_FILE_NAME "${PACKAGE}-${VERSION}")
SET (CPACK_SOURCE_TOPLEVEL_TAG "Linux-Source:")
SET (CPACK_TOPLEVEL_TAG "Linux-Source:")

# The following components are regex's to match anywhere (unless anchored)
# in absolute path + filename to find files or directories to be excluded
# from source tarball.
SET (CPACK_SOURCE_IGNORE_FILES
# temporary files
"\\\\.swp$"
# backup files
"~$"
# eclipse files
"\\\\.cdtproject$"
"\\\\.cproject$"
"\\\\.project$"
"\\\\.settings/"
# others
"\\\\.#"
"/#"
"/build/"
"/_build/"
"/\\\\.git/"
# used before
"/\\\\.libs/"
"/\\\\.deps/"
"\\\\.o$"
"\\\\.lo$"
"\\\\.la$"
"Makefile$"
"Makefile\\\\.in$"
# cmake cache files
"DartConfiguration.tcl$"
"CMakeCache.txt"
"CMakeFiles"
"cmake_install.cmake$"
"CMakeLists.txt.auto$"
"CTestTestfile.cmake"
"CPackConfig.cmake$"
"CPackSourceConfig.cmake$"
"libsolv.spec$"
)

INCLUDE(CPack)

####################################################################
# INCLUDES                                                         #
####################################################################

#SET (CMAKE_INCLUDE_DIRECTORIES_BEFORE ON)
INCLUDE_DIRECTORIES (${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_SOURCE_DIR} ${CMAKE_SOURCE_DIR}/src ${CMAKE_SOURCE_DIR}/ext ${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_BINARY_DIR}/src SYSTEM )

####################################################################

MESSAGE (STATUS "Looking for modules in ${CMAKE_MODULE_PATH}")

set (CMAKE_C_FLAGS     "${CMAKE_C_FLAGS} -Wall")
set (CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS} -O3")
set (CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS} -g -O3")
set (CMAKE_C_FLAGS_DEBUG     "${CMAKE_C_FLAGS} -g3 -O0")

SET (SYSTEM_LIBRARIES "-Wl,--as-needed" ${EXPAT_LIBRARY} ${ZLIB_LIBRARY})
IF (ENABLE_LZMA_COMPRESSION)
SET (SYSTEM_LIBRARIES ${SYSTEM_LIBRARIES} ${LZMA_LIBRARY})
ENDIF (ENABLE_LZMA_COMPRESSION)
IF (ENABLE_BZIP2_COMPRESSION)
SET (SYSTEM_LIBRARIES ${SYSTEM_LIBRARIES} ${BZIP2_LIBRARIES})
ENDIF (ENABLE_BZIP2_COMPRESSION)
IF (ENABLE_RPMDB)
SET (SYSTEM_LIBRARIES ${RPMDB_LIBRARY} ${SYSTEM_LIBRARIES})
ENDIF (ENABLE_RPMDB)

ADD_SUBDIRECTORY (src)
ADD_SUBDIRECTORY (ext)
ADD_SUBDIRECTORY (tools)
IF (ENABLE_PERL OR ENABLE_PYTHON OR ENABLE_RUBY)
    ADD_SUBDIRECTORY (bindings)
ENDIF (ENABLE_PERL OR ENABLE_PYTHON OR ENABLE_RUBY)
ADD_SUBDIRECTORY (examples)

MESSAGE (STATUS "Version: ${VERSION}")

####################################################################
# RPM SPEC                                                         #
####################################################################

MACRO (SPECFILE)
  MESSAGE (STATUS "Writing spec file...")
  CONFIGURE_FILE (${CMAKE_SOURCE_DIR}/package/libsolv.spec.in ${CMAKE_BINARY_DIR}/package/libsolv.spec @ONLY)
ENDMACRO (SPECFILE)

SPECFILE ()

SET (AUTOBUILD_COMMAND
  COMMAND ${CMAKE_COMMAND} -E remove ${CMAKE_BINARY_DIR}/package/*.tar.bz2
  COMMAND mkdir -p _CPack_Packages/${CPACK_TOPLEVEL_TAG}
  COMMAND ${CMAKE_MAKE_PROGRAM} package_source
  COMMAND ${CMAKE_COMMAND} -E copy ${CPACK_SOURCE_PACKAGE_FILE_NAME}.tar.bz2 ${CMAKE_BINARY_DIR}/package
  COMMAND ${CMAKE_COMMAND} -E remove ${CPACK_SOURCE_PACKAGE_FILE_NAME}.tar.bz2
  COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_SOURCE_DIR}/package/libsolv.changes" "${CMAKE_BINARY_DIR}/package/libsolv.changes"
)

ADD_CUSTOM_TARGET (srcpackage
  ${AUTOBUILD_COMMAND}
)

ADD_CUSTOM_TARGET (srcpackage_local
  ${AUTOBUILD_COMMAND}
)

ENABLE_TESTING()
ADD_SUBDIRECTORY (test)
