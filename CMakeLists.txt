#
# This is the toplevel CMakeLists.txt that is used to build Couchbase
#
CMAKE_MINIMUM_REQUIRED(VERSION 3.12)

STRING(REGEX REPLACE "[ ]" "a" modified_source_dir "${CMAKE_SOURCE_DIR}")
IF (NOT modified_source_dir STREQUAL CMAKE_SOURCE_DIR)
  MESSAGE(FATAL_ERROR "Couchbase cannot be built in a source directory containing a space character")
ENDIF (NOT modified_source_dir STREQUAL CMAKE_SOURCE_DIR)

STRING(REGEX REPLACE "[ ]" "a" modified_binary_dir "${CMAKE_BINARY_DIR}")
IF (NOT modified_binary_dir STREQUAL CMAKE_BINARY_DIR)
  MESSAGE(FATAL_ERROR "Couchbase cannot be built in a build directory containing a space character")
ENDIF (NOT modified_binary_dir STREQUAL CMAKE_BINARY_DIR)

# The PROJECT() macro sets it's own default build type, so need to specify
# ours first.
IF (NOT DEFINED CMAKE_BUILD_TYPE)
   SET(CMAKE_BUILD_TYPE "RelWithDebInfo" CACHE STRING
       "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel."
       FORCE)
ENDIF ()

# Include our custom build type (DebugOptimized) in the list.
SET(CMAKE_BUILD_TYPE "${CMAKE_BUILD_TYPE}" CACHE STRING
    "Choose the type of build, options are: Debug DebugOptimized Release RelWithDebInfo MinSizeRel."
    FORCE)

#
# Choose deployment target on macOS.
# As of Mad-Hatter, minimum supported is High Sierra (10.13).
#
SET(CMAKE_OSX_DEPLOYMENT_TARGET "10.13" CACHE STRING
    "Minimum supported version of MacOS X")

PROJECT("Couchbase")

OPTION(COUCHBASE_KV_COMMIT_VALIDATION
       "Only build modules used by KV commit validation"
       OFF)

# This option should be be enabled in the early phases of development
# (e.g. when developing off master) and is used to guard additional
# runtime checks which are desirable to report during testing; but
# should not trigger in production.  As such this option should be set
# to OFF before we move into the Beta/pre-release phase.
#
# Example usage - asserting that statistical counters are of an
# expected range (don't underflow) - we would like to detect such
# bugs; but they arn't serious enough to warrant crashing in a
# customer environment.
OPTION(CB_DEVELOPMENT_ASSERTS
       "Enable additional asserts which abort execution if they fail."
       OFF)

# Always run with assertions for CV
IF(COUCHBASE_KV_COMMIT_VALIDATION)
    SET(CB_DEVELOPMENT_ASSERTS ON)
ENDIF(COUCHBASE_KV_COMMIT_VALIDATION)

IF (CB_DEVELOPMENT_ASSERTS)
   MESSAGE(STATUS "Enabling CB_DEVELOPMENT_ASSERTS")
   ADD_DEFINITIONS(-DCB_DEVELOPMENT_ASSERTS)
ENDIF (CB_DEVELOPMENT_ASSERTS)

IF ($ENV{JETBRAINS_IDE})
   SET(JETBRAINS_CLION_IDE True CACHE BOOL "Build Couchbase under CLion")
ENDIF ($ENV{JETBRAINS_IDE})

IF (DEFINED JETBRAINS_CLION_IDE)
   # CLion tries to autodetect all of the different configuration
   # types and generate projects for them. That cause a race condition
   # with the population pf the precompiled binaries we've got
   # making an incomplete configuration. To work around that,
   # let's just remove all of the other configurations and just
   # build a debug build
   SET(CMAKE_CONFIGURATION_TYPES Debug CACHE STRING "" FORCE)
   ADD_DEFINITIONS(-DJETBRAINS_CLION_IDE)
ENDIF (DEFINED JETBRAINS_CLION_IDE)

IF (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  SET(CMAKE_INSTALL_PREFIX "${CMAKE_SOURCE_DIR}/install" CACHE STRING
      "The install location" FORCE)
  LIST(APPEND CMAKE_PREFIX_PATH "${CMAKE_INSTALL_PREFIX}")
ENDIF (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)

LIST (APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/tlm/cmake/Modules/")
SET (COUCHBASE_SERVER_BUILD True CACHE BOOL "Indicating this is a server build")

# Try to download the prebuilt 3rd-party dependencies by default
IF (NOT DEFINED CB_DOWNLOAD_DEPS)
   SET(CB_DOWNLOAD_DEPS True
       CACHE BOOL "Download prebuilt dependencies by default")
ENDIF (NOT DEFINED CB_DOWNLOAD_DEPS)

# Have to add deps first, before any other part of the build tries to
# FIND anything
ADD_SUBDIRECTORY(tlm/deps)

INCLUDE(CouchbaseDefaultValues)

# If the user has synced any closed-source code, assume building EE
IF (EXISTS "${PROJECT_SOURCE_DIR}/analytics")
  SET (_ee_found ON)
ELSE ()
  SET (_ee_found OFF)
ENDIF()
OPTION (BUILD_ENTERPRISE "Building Enterprise Edition" ${_ee_found})

# Build 'developer builds' by default
OPTION (CB_DEVELOPER_BUILD "Produce Developer builds" ON)

# 'Production builds' are those built by the official Couchbase build jobs
OPTION (CB_PRODUCTION_BUILD "Official production build" OFF)

# Add option to build tools only
SET(BUILD_ONLY_TOOLS False)
IF ($ENV{CB_TOOLS_ONLY})
  SET(BUILD_ONLY_TOOLS True)
ENDIF ($ENV{CB_TOOLS_ONLY})

MESSAGE(STATUS "Using cmake version: ${CMAKE_VERSION}")
MESSAGE(STATUS "Installing to ${CMAKE_INSTALL_PREFIX}")
MESSAGE(STATUS "Build type: ${CMAKE_BUILD_TYPE}")

INCLUDE(CouchbaseCompilerOptions)
INCLUDE(CouchbaseSanitizers)
INCLUDE(CouchbaseAsciiDocs)
INCLUDE(CouchbaseMemoryAllocator)
INCLUDE(CouchbaseCompression)

INCLUDE(FindCouchbaseBoost)
INCLUDE(FindCouchbaseFolly)
INCLUDE(FindCouchbaseOpenSSL)
INCLUDE(FindCouchbaseLibevent)
INCLUDE(FindCouchbaseCurl)
INCLUDE(FindCouchbaseLua)
INCLUDE(FindCouchbasePythonInterp)
INCLUDE(FindCouchbaseErlang)
INCLUDE(FindCouchbaseDtrace)
INCLUDE(FindCouchbaseGo)
INCLUDE(FindCouchbaseBreakpad)
INCLUDE(FindCouchbaseFlatbuffers)
INCLUDE(FindCouchbaseValgrind)
INCLUDE(FindCouchbaseV8)
INCLUDE(FindCouchbaseIcu)
INCLUDE(FindCouchbaseNlohmannJson)
INCLUDE(FindCouchbaseRocksDB)
INCLUDE(FindCouchbaseOpentracing)
INCLUDE(FindCouchbasePCRE)

if (BUILD_ENTERPRISE)
    # libuv is currently only used by the enterpise builds
    include(FindCouchbaseLibuv)
endif()

# Declare PRODUCT_VERSION in cache, but then override user-defined value
# if it is an empty string
SET (PRODUCT_VERSION "" CACHE STRING "Product version with build number")
IF ("${PRODUCT_VERSION}" STREQUAL "")
  SET (_product_version)
  IF (PYTHON_EXECUTABLE)
    # Try to determine product version from manifest
    EXECUTE_PROCESS(COMMAND ${PYTHON_EXECUTABLE} tlm/scripts/get_version.py
      WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
      OUTPUT_VARIABLE _product_version
      ERROR_QUIET
      OUTPUT_STRIP_TRAILING_WHITESPACE)
  ENDIF ()
  IF (NOT _product_version)
    SET (_product_version "0.0.0-9999")
  ENDIF ()
  SET (PRODUCT_VERSION ${_product_version} CACHE STRING "Product version with build number" FORCE)
ENDIF ("${PRODUCT_VERSION}" STREQUAL "")

IF (BUILD_ENTERPRISE)
  MESSAGE (STATUS "Building Couchbase Server ${PRODUCT_VERSION} Enterprise Edition")
  ADD_DEFINITIONS(-DCOUCHBASE_ENTERPRISE_EDITION=1)
ELSE ()
  MESSAGE (STATUS "Building Couchbase Server ${PRODUCT_VERSION} Community Edition")
ENDIF ()

IF (WIN32)
   SET(COUCHBASE_NETWORK_LIBS "Ws2_32")
ELSEIF ("${CMAKE_SYSTEM_NAME}" STREQUAL "SunOS")
   SET(COUCHBASE_NETWORK_LIBS socket nsl)
ENDIF (WIN32)
MESSAGE(STATUS "Linking with network libraries: ${COUCHBASE_NETWORK_LIBS}")

IF (NOT WIN32)
   SET(COUCHBASE_MATH_LIBS m)
ENDIF(NOT WIN32)

INCLUDE(CouchbaseCodeCoverage)

INCLUDE(TestBigEndian)
TEST_BIG_ENDIAN(WORDS_BIG_ENDIAN)
IF (WORDS_BIG_ENDIAN)
  MESSAGE(STATUS "Builing on big endian system")
  ADD_DEFINITIONS(-DWORDS_BIGENDIAN=1)
ENDIF(WORDS_BIG_ENDIAN)

ENABLE_TESTING()

IF (NOT BUILD_ONLY_TOOLS)
  INCLUDE_DIRECTORIES(BEFORE ${CMAKE_BINARY_DIR}/platform/include)
  INCLUDE_DIRECTORIES(AFTER SYSTEM ${NLOHMANN_JSON_INCLUDE_DIR})
ENDIF()

# Add a quick (and safe) way to nuke everything
ADD_CUSTOM_TARGET(reset
                  COMMAND repo forall -c "git clean -dfx"
                  VERBATIM)

SET (gtest_force_shared_crt ON CACHE BOOL
     "Use shared (DLL) run-time lib even when Google Test is built as static lib.")
ADD_SUBDIRECTORY(googletest EXCLUDE_FROM_ALL)

ADD_SUBDIRECTORY(benchmark EXCLUDE_FROM_ALL)

IF (NOT BUILD_ONLY_TOOLS)
  ADD_SUBDIRECTORY(tlm/python)
  ADD_SUBDIRECTORY(phosphor)
ENDIF()

OPTION(PHOSPHOR_DISABLE "Disable compilation of Phosphor trace points into components" OFF)

IF (PHOSPHOR_DISABLE)
    ADD_DEFINITIONS(-DPHOSPHOR_DISABLED=1)
    SET(PHOSPHOR_LIB "")
ENDIF (PHOSPHOR_DISABLE)

ADD_SUBDIRECTORY(third_party EXCLUDE_FROM_ALL)

IF (NOT BUILD_ONLY_TOOLS)
  ADD_SUBDIRECTORY(platform)
  ADD_SUBDIRECTORY(subjson)
  ADD_SUBDIRECTORY(kv_engine/include)
  ADD_SUBDIRECTORY(couchstore)
  IF (BUILD_ENTERPRISE AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/magma")
    ADD_SUBDIRECTORY(magma)
  ENDIF()
ENDIF()

IF (NOT COUCHBASE_KV_COMMIT_VALIDATION)
    ADD_SUBDIRECTORY(forestdb)
    SET (FORESTDB_INCLUDE_DIR "${ForestDB_SOURCE_DIR}/include")
    SET(FORESTDB_LIBRARY_DIR "${ForestDB_BINARY_DIR}")
    SET (FORESTDB_TARGET forestdb)
ENDIF()

IF (NOT BUILD_ONLY_TOOLS)
  ADD_SUBDIRECTORY(kv_engine)
  ADD_SUBDIRECTORY(sigar)
ENDIF()

IF (NOT COUCHBASE_KV_COMMIT_VALIDATION)
    IF (NOT BUILD_ONLY_TOOLS)
      ADD_SUBDIRECTORY(couchbase-cli)
      ADD_SUBDIRECTORY(couchdb)
      ADD_SUBDIRECTORY(geocouch)
      ADD_SUBDIRECTORY(query-ui)
    ENDIF()

    SET (GODEPSDIR "${CMAKE_CURRENT_SOURCE_DIR}/godeps")
    IF (NOT BUILD_ONLY_TOOLS)
      ADD_SUBDIRECTORY(ns_server ${CMAKE_CURRENT_SOURCE_DIR}/ns_server/build)

      ADD_SUBDIRECTORY(goproj/src/github.com/couchbase/query)
      ADD_SUBDIRECTORY(goproj/src/github.com/couchbase/indexing)
      ADD_SUBDIRECTORY(goproj/src/github.com/couchbase/goxdcr)
      ADD_SUBDIRECTORY(goproj/src/github.com/couchbase/gometa)
      ADD_SUBDIRECTORY(godeps/src/github.com/couchbase/cbauth)

      ADD_SUBDIRECTORY(goproj/src/github.com/couchbase/cbft)
      ADD_SUBDIRECTORY(goproj/src/github.com/couchbase/docloader)
      ADD_SUBDIRECTORY(goproj/src/github.com/couchbase/mobile-service)

      # Analytics CBGT connector
      IF (BUILD_ENTERPRISE AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/goproj/src/github.com/couchbase/cbas")
        ADD_SUBDIRECTORY(goproj/src/github.com/couchbase/cbas)
      ENDIF ()

      # Analytics Maven project
      IF (BUILD_ENTERPRISE AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/analytics/CMakeLists.txt")
        ADD_SUBDIRECTORY (analytics)
      ENDIF ()

      # Analytics UI
      IF (BUILD_ENTERPRISE AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/cbas-ui")
        ADD_SUBDIRECTORY (cbas-ui)
      ENDIF ()
    ENDIF()

    IF (EXISTS "${PROJECT_SOURCE_DIR}/goproj/src/github.com/couchbase/backup")
      ADD_SUBDIRECTORY(goproj/src/github.com/couchbase/backup)
    ENDIF()

    IF (BUILD_ENTERPRISE AND EXISTS "${PROJECT_SOURCE_DIR}/goproj/src/github.com/couchbase/plasma" AND NOT BUILD_ONLY_TOOLS)
      ADD_SUBDIRECTORY(goproj/src/github.com/couchbase/plasma)
    ENDIF()

    IF (BUILD_ENTERPRISE AND EXISTS "${PROJECT_SOURCE_DIR}/libcouchbase")
      SET (LCB_NO_PLUGINS ON CACHE BOOL "Don't build plugins" FORCE)
      SET (LCB_NO_TESTS ON CACHE BOOL "Don't run tests" FORCE)
      SET (LCB_BUILD_DTRACE OFF CACHE BOOL "Turn off tracing" FORCE)
      SET (LCB_INSTALL_HEADERS OFF CACHE BOOL "Don't install headers" FORCE)
      SET (LCB_INSTALL_PKGCONFIG OFF CACHE BOOL "Don't install libcouchbase.pc" FORCE)
      SET (CMAKE_INSTALL_LIBDIR "lib" CACHE STRING "Override libdir to violate FHS" FORCE)
      ADD_SUBDIRECTORY(libcouchbase)
    ENDIF()

    IF (NOT BUILD_ONLY_TOOLS)
      IF (BUILD_ENTERPRISE)
        ADD_SUBDIRECTORY(goproj/src/github.com/couchbase/eventing)
        ADD_SUBDIRECTORY(goproj/src/github.com/couchbase/eventing-ee)
      ENDIF()

      IF (BUILD_ENTERPRISE AND EXISTS "${PROJECT_SOURCE_DIR}/goproj/src/github.com/couchbase/cbsummary")
        ADD_SUBDIRECTORY(goproj/src/github.com/couchbase/cbsummary)
      ENDIF()
    ENDIF ()

    IF (EXISTS "${PROJECT_SOURCE_DIR}/voltron/CMakeLists.txt")
      ADD_SUBDIRECTORY(voltron)
    ENDIF ()
ENDIF (NOT COUCHBASE_KV_COMMIT_VALIDATION)

IF (NOT WIN32)
   CONFIGURE_FILE(tlm/memcached-wrapper.in ${CMAKE_BINARY_DIR}/tlm/memcached-wrapper)
ENDIF (NOT WIN32)
