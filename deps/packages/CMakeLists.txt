# Project for building the third-party dependencies for Couchbase Server.
#
# Note that this project is not intended to be run as part of a
# Couchbase Server build. It is included in the Couchbase Server
# source code for reference (as these are the steps taken to produce
# the pre-packaged dependencies which are downloaded as part of the
# Couchbase Server build) as well as to make it easier to build
# Couchbase Server on novel platforms.

#
# The following are the variables set by this script:
# HOST_ARCH
# PLATFORM

PROJECT (cbdeps_packages)

CMAKE_MINIMUM_REQUIRED (VERSION 2.8)

LIST (APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/../../cmake/Modules")
INCLUDE (PlatformIntrospection)
INCLUDE (CBDownloadDeps)

_DETERMINE_PLATFORM (cbcache_platform)
_DETERMINE_ARCH (cbcache_arch)

MESSAGE (STATUS "Set platform to ${cbcache_platform} for dependency downloads")
MESSAGE (STATUS "Set arch to ${cbcache_arch} for dependency downloads")

# Generate a .md5 file
MACRO (_ADD_MD5_FILE_STEP sourcefile md5file package)
  ExternalProject_Add_Step (${package} md5
    COMMAND "${CMAKE_COMMAND}"
    -D "FILE=${sourcefile}"
    -D "MD5FILE=${md5file}"
    -P "${PROJECT_SOURCE_DIR}/util/GenMD5.cmake"
    DEPENDEES package)
ENDMACRO (_ADD_MD5_FILE_STEP)


# Macro intended to be called by package sub-projects to provide
# the ExternalPackage "package" step.
MACRO (_ADD_PACKAGE_STEP)
  ExternalProject_Add_Step(${_dep_package} package
    COMMAND ${CMAKE_COMMAND} -E make_directory ${_output_dir}
    COMMAND ${CMAKE_COMMAND} -E tar czf ${_output_file}.tgz .
    COMMENT "Performing packaging step for '${_dep_package}' - writing artifacts to ${_output_dir}"
    DEPENDEES install
    WORKING_DIRECTORY <INSTALL_DIR>
    )
  _ADD_MD5_FILE_STEP(${_output_file}.tgz ${_output_file}.md5 ${_dep_package})

  if (CB_DOWNLOAD_DEPS_CACHE)
    SET(_cached_file "${CB_DOWNLOAD_DEPS_CACHE}/${_output_file_name}.tgz")
    SET(_cached_md5 "${_cached_file}.md5")
    SET(_target build-and-cache-${_dep_package})

    _CHECK_CACHED_DEP_FILE("${_cached_file}" "${_cached_md5}" _dep_found)
    if (_dep_found)
      ADD_CUSTOM_TARGET(${_target}
        COMMAND ${CMAKE_COMMAND} -E echo "Seems like you already have ${_cached_file}"
        COMMAND ${CMAKE_COMMAND} -E echo "If that is not the case please regenerate cmake files"
        COMMAND ${CMAKE_COMMAND} -E echo "If you want to rebuild the dependency, just delete the file")
    else ()
      ADD_CUSTOM_TARGET(${_target}
        COMMAND ${CMAKE_COMMAND} -E copy "${_output_file}.tgz" "${_cached_file}"
        COMMAND ${CMAKE_COMMAND} -E copy "${_output_file}.md5" "${_cached_md5}"
        DEPENDS ${_dep_package})
    endif ()

    ADD_DEPENDENCIES(build-and-cache-all ${_target})
  endif ()
ENDMACRO (_ADD_PACKAGE_STEP)

# Macro to standardize several steps in sub-projects.
# Args:
#   package - name of the third-party package. It is expected that a
#     a subdirectory of the current directory exists with this name,
#     containing a CMakeLists which defines an ExternalProject of
#     the same name. The ExternalProject must have all steps up to
#     and including "install" fully defined, and then call
#     _ADD_PACKAGE_STEP() to complete the packaging.
#   upstream_ver - the version of the upstream project to build. This
#     is strictly a label; it will be used to determine the resulting
#     cbdeps package name.
#   cb_branch - all source will be retrieved from
#      github.com/couchbasedeps/<package>. This parameter determines
#      the branch / tag / SHA to check out.
#   cb_release - a numeric index, which should be advanced whenever we
#      need to re-package a given dependency while maintaining the same
#      upstream_ver.
# If the CMake variable 'PACKAGE' is defined but not equal to package,
# this macro will do nothing. This allows incremental building of packages
# on a new platform.
#
MACRO (_ADD_DEP_PACKAGE package upstream_ver cb_branch cb_release)
  IF (NOT DEFINED PACKAGE OR "${PACKAGE}" STREQUAL "${package}")
    SET (_output_file_name "${package}-${PLATFORM}-${HOST_ARCH}-${upstream_ver}-cb${cb_release}")
    SET (_output_dir "${CMAKE_BINARY_DIR}/deps/${package}/${upstream_ver}-cb${cb_release}")
    SET (_output_file "${_output_dir}/${_output_file_name}")
    SET (_git_repo "git://github.com/couchbasedeps/${package}.git")
    SET (_git_rev ${cb_branch})
    SET (_dep_package ${package})
    SET (_dep_version ${upstream_ver})
    ADD_SUBDIRECTORY (${package})
  ENDIF (NOT DEFINED PACKAGE OR "${PACKAGE}" STREQUAL "${package}")
ENDMACRO (_ADD_DEP_PACKAGE)


#
# Standard code run on include to perform common checks and setup useful vars
#
_DETERMINE_ARCH(HOST_ARCH)
_DETERMINE_PLATFORM(PLATFORM)

#
# Check for autotools stuff, as many of the deps require them.
#
IF (NOT WIN32)
   FIND_PROGRAM(AUTOMAKE automake)
   IF (NOT AUTOMAKE)
      MESSAGE(WARNING "Building many packages requires automake")
   ENDIF (NOT AUTOMAKE)
   FIND_PROGRAM(AUTOCONF autoconf)
   IF (NOT AUTOCONF)
       MESSAGE(WARNING "Building many packages requires autoconf")
   ENDIF (NOT AUTOCONF)
   FIND_PROGRAM(LIBTOOLIZE libtoolize)
   IF (NOT LIBTOOLIZE)
       FIND_PROGRAM(GLIBTOOLIZE glibtoolize)
       IF (GLIBTOOLIZE)
           MESSAGE(FATAL_ERROR "No libtoolize found, but there is a glibtoolize. You should try to create a symbolic link")
       ENDIF (GLIBTOOLIZE)
       MESSAGE(WARNING "Building many packages requires libtoolize")
   ENDIF (NOT LIBTOOLIZE)
   FIND_PROGRAM(AUTORECONF autoreconf)
   IF (NOT AUTORECONF)
       MESSAGE(WARNING "Building many packages requires autoreconf")
   ENDIF (NOT AUTORECONF)
ENDIF (NOT WIN32)

# msbuild uses a different names for architecture (compared to HOST_ARCH)
if (WIN32)
  if(HOST_ARCH STREQUAL "amd64")
    SET(MSBUILD_ARCH "x64")
  elseif(HOST_ARCH STREQUAL "x86")
    SET(MSBUILD_ARCH "Win32")
  else(HOST_ARCH STREQUAL "amd64")
    SET(MSBUILD_ARCH ${HOST_ARCH})
  endif(HOST_ARCH STREQUAL "amd64")
endif(WIN32)

ADD_CUSTOM_TARGET(build-and-cache-all)

#
# And here is the set of "latest" dependencies.
#
# Note: Snappy 1.1.1-cb6 was used for Alice only;
# next master build (if needed) should be -cb7
_ADD_DEP_PACKAGE(snappy 1.1.1 1.1.1 5)
_ADD_DEP_PACKAGE(python-snappy c97d633 c97d633 2)
_ADD_DEP_PACKAGE(icu4c 59.1 r59.1 4)
_ADD_DEP_PACKAGE(libevent 2.1.8 release-2.1.8-stable-cb 10)
_ADD_DEP_PACKAGE(v8 7.1 7.1.321 5)
_ADD_DEP_PACKAGE(jemalloc 5.2.1 5.2.1 4)

_ADD_DEP_PACKAGE(erlang OTP-20.3.8.11 couchbase-madhatter 15)

_ADD_DEP_PACKAGE(libsqlite3 3.7.2 3.7.2 1)

_ADD_DEP_PACKAGE(flatbuffers 1.10.0 v1.10.0 5)

_ADD_DEP_PACKAGE(json 3.5.0 v3.5.0 2)
_ADD_DEP_PACKAGE(boost 1.67.0 boost-1.67.0 7)

# Note: we build a number of packages against openssl,
# including Erlang, libevent, folly, and grpc. Those packages
# have a DECLARE_DEP(openssl) in their own CMakeLists.txt.
# Similarly, curl (cbdeps V2) uses the 'cbdep' tool to
# download the openssl cbdeps package.
# None of those packages *include* openssl, so it isn't
# strictly necessary to rebuild them all when we update
# OpenSSL itself unless it includes a breaking API change.
# Additional Note: when updating OpenSSL itself, try to
# also fix the version included in our Anaconda Python
# package, in tlm/python/construct.yaml.
_ADD_DEP_PACKAGE(openssl 1.1.1d OpenSSL_1_1_1d 2)

_ADD_DEP_PACKAGE(libcouchbase 2.10.0 2.10.0 1)
IF (CMAKE_SYSTEM_NAME STREQUAL "Linux")
  IF (PLATFORM STREQUAL "fedora26" OR PLATFORM STREQUAL "fedora27")
    _ADD_DEP_PACKAGE(libcxx 4.0.0 release_40 1)
  ELSE ()
    _ADD_DEP_PACKAGE(libcxx 3.9.1 release_39 2)
  ENDIF ()
ENDIF ()
IF (WIN32)
  _ADD_DEP_PACKAGE(flex 2.5.4a master 1)
ELSE ()
  _ADD_DEP_PACKAGE(flex 2.6.4 master 6)
ENDIF ()
_ADD_DEP_PACKAGE(numactl 2.0.11 v2.0.11 3)

IF (NOT WIN32)
    _ADD_DEP_PACKAGE(rocksdb 5.18.3 5.18.3-couchbase 4)
ENDIF ()
_ADD_DEP_PACKAGE(lz4 1.8.0 v1.8.0 4)
# maven 3.5.2-cb3 and 3.5.2-cb4 exist as attempts to make Windows work,
# but are not correct.
_ADD_DEP_PACKAGE(maven 3.5.2 master 6)
_ADD_DEP_PACKAGE(openjdk-rt 1.8.0.171 UNUSED 2)

_ADD_DEP_PACKAGE(double-conversion 3.0.0 v3.0.0 4)
_ADD_DEP_PACKAGE(gflags 2.2.1 v2.2.1 4)
_ADD_DEP_PACKAGE(glog 0.3.5 v0.3.5 4)

_ADD_DEP_PACKAGE(folly v2019.08.12.00 v2019.08.12.00 2)
_ADD_DEP_PACKAGE(grpc 1.12.0 couchbase-v1.12.0 11)
_ADD_DEP_PACKAGE(pcre 8.43 v8.43 2)
_ADD_DEP_PACKAGE(opentracing-cpp v1.5.1 v1.5.1 4)
