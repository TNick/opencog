#
# Master Opencog CMake file.
#
# General organization:
# -- preamble
# -- check for different compilers, OS'es
# -- search for various required & optional libraries/tools
# -- decide what to build based on above results.
# -- configure various config files.
# -- print pretty summary
#
# To specify the type of build call cmake as follows:
# > cmake ../src -DCMAKE_BUILD_TYPE="Debug"
# > cmake ../src -DCMAKE_BUILD_TYPE="Release"
# > cmake ../src -DCMAKE_BUILD_TYPE="Coverage"
# > cmake ../src -DCMAKE_BUILD_TYPE="Profile"
# Coverage refers to the coverage of tests over the source code
# Profile prepares the sources for profiling for speed
#
#
#
#

# ===============================================================
# Preamble; prepare Cmake the way we want

PROJECT(opencog)

CMAKE_MINIMUM_REQUIRED(VERSION 2.6)
IF (COMMAND CMAKE_POLICY)
	CMAKE_POLICY(SET CMP0003 NEW)
	CMAKE_POLICY(SET CMP0005 OLD)
ENDIF (COMMAND CMAKE_POLICY)

# includes
include("cmake/macros/Summary.cmake")

# add the 'lib' dir to cmake's module search path
SET(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/modules")

# To supress Cygwin warning "CMake no longer defines WIN32 on Cygwin!"
# Remove when CMake >= 2.8.4 is required to build the project
set(CMAKE_LEGACY_CYGWIN_WIN32 0)

# default build type
IF (CMAKE_BUILD_TYPE STREQUAL "")
	SET(CMAKE_BUILD_TYPE Release)
ENDIF (CMAKE_BUILD_TYPE STREQUAL "")
MESSAGE(STATUS "Build type: ${CMAKE_BUILD_TYPE}")

# ===============================================================
# Detect different compilers and OS'es, tweak flags as necessary.

IF (CMAKE_COMPILER_IS_GNUCXX)
  IF (APPLE)
    SET(CMAKE_C_FLAGS "-Wall -Wno-long-long -Wno-conversion")
    SET(CMAKE_C_FLAGS_DEBUG "-O0 -g")
    SET(CMAKE_C_FLAGS_PROFILE "-O0 -pg")
    SET(CMAKE_C_FLAGS_RELEASE "-O2 -g0")
    # Vital to do this otherwise unresolved symbols everywhere:
    SET(CMAKE_SHARED_LINKER_FLAGS "-Wl,-flat_namespace,-undefined,dynamic_lookup")
    SET(CMAKE_EXE_LINKER_FLAGS "-Wl,-flat_namespace,-undefined,dynamic_lookup")
  ELSE (APPLE)
    SET(CMAKE_C_FLAGS "-Wall -fPIC")
    SET(CMAKE_C_FLAGS_DEBUG "-ggdb3 -fstack-protector")
    SET(CMAKE_C_FLAGS_PROFILE "-O2 -g3 -fstack-protector -pg")
    SET(CMAKE_C_FLAGS_RELEASE "-O2 -g -fstack-protector")
  ENDIF (APPLE)

  # 1) -Wno-variadic-macros is to avoid warnings regarding using
  # variadic in macro OC_ASSERT (the warning warns that this is only
  # available from C99, lol!)
  #
  # 2) -fopenmp for multithreading support
  #
  # 3) -std=gnu++0x for C++0x and GNU extensions support
  SET(CMAKE_CXX_FLAGS "${CMAKE_C_FLAGS} -Wno-variadic-macros -fopenmp -std=gnu++0x")

  SET(CMAKE_CXX_FLAGS_DEBUG ${CMAKE_C_FLAGS_DEBUG})
  SET(CMAKE_CXX_FLAGS_PROFILE ${CMAKE_C_FLAGS_PROFILE})
  SET(CMAKE_CXX_FLAGS_RELEASE ${CMAKE_C_FLAGS_RELEASE})

  # Options for generating gcov code coverage output
  SET(CMAKE_C_FLAGS_COVERAGE "-O0 -g -fprofile-arcs -ftest-coverage -fno-inline")
  SET(CMAKE_CXX_FLAGS_COVERAGE "${CMAKE_C_FLAGS_COVERAGE} -fno-default-inline")
  # Might be needed for some combinations of ln and gcc
  IF (CMAKE_BUILD_TYPE STREQUAL "Coverage")
  LINK_LIBRARIES(gcov)
  ENDIF (CMAKE_BUILD_TYPE STREQUAL "Coverage")

ENDIF (CMAKE_COMPILER_IS_GNUCXX)

IF (CYGWIN)
   ADD_DEFINITIONS(-DCYGWIN)
ENDIF (CYGWIN)

IF (WIN32)
   ADD_DEFINITIONS(-DWIN32)
ENDIF (WIN32)

# this is for the case when the code is compiled under windows but not
# under cygwin
IF (WIN32 AND NOT UNIX)
   ADD_DEFINITIONS(-DWIN32_NOT_UNIX)
ENDIF (WIN32 AND NOT UNIX)

# check dependencies
IF (WIN32 AND NOT UNIX)
	FIND_PACKAGE(PThreads REQUIRED)
	FIND_PACKAGE(STLPort REQUIRED)
	INCLUDE_DIRECTORIES("${CMAKE_SOURCE_DIR}/include/win32" ${PTHREADS_INCLUDE_DIR} ${STLPORT_INCLUDE_DIR})
	LINK_LIBRARIES(${PTHREADS_LIBRARY} ${STLPORT_LIBRARIES})
	ADD_DEFINITIONS(-D_CRT_SECURE_NO_WARNINGS -D_CRT_NONSTDC_NO_DEPRECATE)
ELSE (WIN32 AND NOT UNIX)
	LINK_LIBRARIES(pthread)
	IF (NOT APPLE AND NOT WIN32)
		IF (CYGWIN)
			SET(CMAKE_SHARED_LINKER_FLAGS "-Wl")
		ELSE (CYGWIN)
			SET(CMAKE_SHARED_LINKER_FLAGS "-Wl,--enable-new-dtags")
		ENDIF (CYGWIN)
	ENDIF (NOT APPLE AND NOT WIN32)
ENDIF (WIN32 AND NOT UNIX)

ADD_DEFINITIONS(-DPROJECT_SOURCE_DIR=\\"${CMAKE_SOURCE_DIR}\\"
                -DPROJECT_BINARY_DIR=\\"${CMAKE_BINARY_DIR}\\")


# ===================================================================
# Check for existance of various required, optional packages.

# Check for boost. We need dynamic-linked, threaded libs by default.
SET(Boost_USE_STATIC_LIBS OFF)
SET(Boost_USE_MULTITHREADED ON)

# Required boost packages
# hypertable requires boost iostreams
# boost-1.49 no longer has a libboost_iostreams
# 1.46 is minimum for required filesystem support
# program_options needed by some combo utilities
FIND_PACKAGE(Boost 1.46 COMPONENTS date_time filesystem program_options regex serialization signals system thread REQUIRED)

IF(Boost_FOUND)
	SET(Boost_FOUND_SAVE 1)
ELSE(Boost_FOUND)
	MESSAGE(FATAL_ERROR "Boost 1.46 or newer is needed to build OpenCog!")
ENDIF(Boost_FOUND)

# Opencog won't compile with Boost 1.51, some kind of conflict with
# hash functions, see github bugs 1 and 36
IF(105100 EQUAL ${Boost_VERSION})
    MESSAGE(FATAL_ERROR "Boost version 1.51 will not work with OpenCog.  Please use a different version.")
ENDIF(105100 EQUAL ${Boost_VERSION})
MESSAGE(STATUS "Boost version ${Boost_VERSION} found.")

# Optional boost packages; can build without these.
FIND_PACKAGE(Boost 1.46 COMPONENTS python program_options QUIET)
FIND_PACKAGE(Boost 1.46 COMPONENTS math_c99 QUIET)

# Arghhh. Except cmake is treating above as required, not optional. #$%**&
IF(Boost_FOUND_SAVE)
	SET(Boost_FOUND 1)
ENDIF(Boost_FOUND_SAVE)

IF(Boost_PROGRAM_OPTIONS_FOUND)
  MESSAGE(STATUS "Found Boost::program_options")
ELSE(Boost_PROGRAM_OPTIONS_FOUND)
  MESSAGE(STATUS "Boost program_options missing: needed for Spatial tools (MapTools).")
ENDIF(Boost_PROGRAM_OPTIONS_FOUND)

IF(Boost_MATH_C99_FOUND)
	MESSAGE(STATUS "Found Boost::math")
ELSE(Boost_MATH_C99_FOUND)
	MESSAGE(STATUS "Boost math missing: needed to run combo learning tests.")
ENDIF(Boost_MATH_C99_FOUND)

FIND_PACKAGE(CURL)
IF (NOT CURL_FOUND)
	MESSAGE(STATUS "libcurl missing: needed for ubigraph module.")
ENDIF (NOT CURL_FOUND)

# XML parsing library
FIND_PACKAGE(EXPAT)
IF (EXPAT_FOUND)
	MESSAGE(STATUS "Expat XML parser found.")
	ADD_DEFINITIONS(-DHAVE_EXPAT)
	SET(HAVE_EXPAT 1)
ELSE (EXPAT_FOUND)
	MESSAGE(STATUS "Expat XML parser missing: needed for embodiment.")
ENDIF (EXPAT_FOUND)

# Gnu Scientific Library
FIND_PACKAGE(GSL)
IF (GSL_FOUND)
	MESSAGE(STATUS "GSL found.")
	ADD_DEFINITIONS(-DHAVE_GSL)
	SET(HAVE_GSL 1)
ELSE (GSL_FOUND)
	MESSAGE(STATUS "GSL missing: needed for the AtomSpace.")
ENDIF (GSL_FOUND)

# This is required for Guile
FIND_LIBRARY(GMP_LIBRARY gmp)
FIND_PATH(GMP_INCLUDE_DIR gmp.h)

# Gtk-3 required for visualization
FIND_PACKAGE(GTK3)
IF (GTK3_FOUND)
	# MESSAGE(STATUS "GTK3 found.")
	ADD_DEFINITIONS(-DHAVE_GTK)
	SET(HAVE_GTK 1)
ELSE (GTK3_FOUND)
	MESSAGE(STATUS "GTK missing: needed for the the gtk-visualizer.")
ENDIF (GTK3_FOUND)

# Gnu Guile scheme interpreter
# Versions prior to 1.8.6 have assorted serious problems with
# multi-threading.  If you need guile, you need this version.
FIND_PACKAGE(Guile 1.8.6)
IF (GUILE_FOUND AND GMP_LIBRARY AND GMP_INCLUDE_DIR)
	ADD_DEFINITIONS(-DHAVE_GUILE)
	SET(HAVE_GUILE 1)
	INCLUDE_DIRECTORIES(${GUILE_INCLUDE_DIR})
	SET(GUILE_DIR_MESSAGE "Guile was found.")
ELSE (GUILE_FOUND AND GMP_LIBRARY AND GMP_INCLUDE_DIR)
	SET(GUILE_DIR_MESSAGE "Guile was not found; the scheme shell will not be built.\nTo over-ride, make sure GUILE_LIBRARIES and GUILE_INCLUDE_DIRS are set.")
ENDIF (GUILE_FOUND AND GMP_LIBRARY AND GMP_INCLUDE_DIR)
MESSAGE(STATUS "${GUILE_DIR_MESSAGE}")

# Hypertable library, for atomspace persistence
FIND_PACKAGE(HyperTable)
IF (HYPERTABLE_FOUND)
	ADD_DEFINITIONS(-DHAVE_HYPERTABLE)
	SET(HAVE_HYPERTABLE 1)
	MESSAGE(STATUS "Hypertable was found.")

	# Don't bother looking for SIGAR unlesss hyperable was found.
	FIND_PACKAGE(SIGAR)
	IF (SIGAR_FOUND)
	ELSE (SIGAR_FOUND)
        MESSAGE(WARNING "SIGAR was not found! It is needed for Hypertable support.")
	ENDIF (SIGAR_FOUND)

ELSE (HYPERTABLE_FOUND)
	# The FindHypertable.cmake file already prints a message.
	# MESSAGE(STATUS "Hypertable was not found; hypertable support will not be built.\n   To over-ride, make sure HYPERTABLE_LIBRARIES and HYPERTABLE_INCLUDE_DIRS\n   are set.")
	MESSAGE(STATUS "Hypertable not found; the experimental distributed persistence DB needs it.")

ENDIF (HYPERTABLE_FOUND)


# Link-Gramar is needed for the Viterbi parser
# Only 4.7.12 or newer has the required read-dict API in it.
FIND_PACKAGE(LinkGrammar 4.7.12)
IF (LINK_GRAMMAR_FOUND)
	SET(HAVE_LINK_GRAMMAR 1)
	MESSAGE(STATUS "Link Grammar was found.")
	INCLUDE_DIRECTORIES(${LINK_GRAMMAR_INCLUDE_DIRS})
ELSE (LINK_GRAMMAR_FOUND)
	MESSAGE(STATUS "Link Grammar missing: needed for the Viterbi parser.")
ENDIF (LINK_GRAMMAR_FOUND)


# MPI is needed for MOSES on compute clusters.
FIND_PACKAGE(MPI)

IF (MPI_FOUND)
	ADD_DEFINITIONS(-DHAVE_MPI)
	SET(HAVE_MPI 1)
	MESSAGE(STATUS "MPI was found.")
	INCLUDE_DIRECTORIES(${MPI_INCLUDE_PATH})
ELSE (MPI_FOUND)
	MESSAGE(STATUS "MPI missing: needed for MPI-MOSES.")
ENDIF (MPI_FOUND)


# OpenGL needed for visualization...
FIND_PACKAGE(OpenGL)

IF (OPENGL_FOUND)
	MESSAGE(STATUS "OpenGL was found.")
ELSE (OPENGL_FOUND)
	MESSAGE(STATUS "OpenGL missing: needed for Spatial tools (MapExplorer, MapTools, etc.).")
ENDIF (OPENGL_FOUND)

# Google Protobuf library
# The protocol buffer compiler is a separately installable package, and
# it is also needed for building embodiment PAI. So check for that, too.
FIND_PACKAGE(Protobuf)
IF (PROTOBUF_FOUND
		AND PROTOBUF_LIBRARY
		AND PROTOBUF_INCLUDE_DIR
		AND PROTOBUF_PROTOC_EXECUTABLE)

	ADD_DEFINITIONS(-DHAVE_PROTOBUF)
	SET(HAVE_PROTOBUF 1)
	SET(PROTOBUF_DIR_MESSAGE "Protobuf was found.")

ELSE (PROTOBUF_FOUND
		AND PROTOBUF_LIBRARY
		AND PROTOBUF_INCLUDE_DIR
		AND PROTOBUF_PROTOC_EXECUTABLE)

	# If we are here, then we are missing either the libraries, or the
	# compiler. React appropriately.
	IF (PROTOBUF_PROTOC_EXECUTABLE)
		SET(PROTOBUF_DIR_MESSAGE "Protobuf was not found; the new message decoding functions would not work.\n   To over-ride, make sure PROTOBUF_LIBRARIES and PROTOBUF_INCLUDE_DIRS are set.")
	ELSE (PROTOBUF_PROTOC_EXECUTABLE)
		SET(PROTOBUF_DIR_MESSAGE "Protobuf compiler was not found; embodiment PAI can't be built without it.\n   To over-ride, make sure PROTOBUF_PROTOC_EXECUTABLE is set.")
	ENDIF (PROTOBUF_PROTOC_EXECUTABLE)

ENDIF (PROTOBUF_FOUND
		AND PROTOBUF_LIBRARY
		AND PROTOBUF_INCLUDE_DIR
		AND PROTOBUF_PROTOC_EXECUTABLE)

MESSAGE(STATUS "${PROTOBUF_DIR_MESSAGE}")

# Python needed for embodiment (!?) and for PLN
FIND_PACKAGE(PythonInterp)
FIND_PACKAGE(PythonLibs)
IF(PYTHONLIBS_FOUND)
	MESSAGE(STATUS "Python libs found.")
ELSE(PYTHONLIBS_FOUND)
	MESSAGE(STATUS "Python libs NOT found.")
ENDIF(PYTHONLIBS_FOUND)

# Python bindings using Cython
# Cython is used to generate bindings
FIND_PROGRAM(CYTHON_EXECUTABLE cython)
FIND_PROGRAM(NOSETESTS_EXECUTABLE nosetests)
IF (CYTHON_EXECUTABLE AND PYTHONLIBS_FOUND)
   # find python destination dir for python bindings
   ADD_DEFINITIONS(-DHAVE_CYTHON)
   SET(HAVE_CYTHON 1)
   EXECUTE_PROCESS(
	   COMMAND ${PYTHON_EXECUTABLE} -c "from distutils.sysconfig import *; print get_python_lib()"
		OUTPUT_VARIABLE PYTHON_DEST
   )

   # replace new line at end
   STRING(REPLACE "\n" "" PYTHON_DEST "${PYTHON_DEST}")
   IF ("${PYTHON_DEST}" STREQUAL "")
	   MESSAGE(FATAL_ERROR "Python destination dir not found")
   ELSE ("${PYTHON_DEST}" STREQUAL "")
	   MESSAGE(STATUS "Python destination dir found: ${PYTHON_DEST}" )
   ENDIF ("${PYTHON_DEST}" STREQUAL "")
ELSE (CYTHON_EXECUTABLE AND PYTHONLIBS_FOUND)
	IF(NOT CYTHON_EXECUTABLE)
		MESSAGE(STATUS "Cython executable not found.")
	ENDIF(NOT CYTHON_EXECUTABLE)
ENDIF (CYTHON_EXECUTABLE AND PYTHONLIBS_FOUND)

# SDL needed for the spatial tools
FIND_PACKAGE(SDL)

IF (SDL_FOUND)
	MESSAGE(STATUS "SDL was found.")
ELSE (SDL_FOUND)
	MESSAGE(STATUS "SDL missing: needed for Spatial tools (MapExplorer, MapTools, etc.).")
ENDIF (SDL_FOUND)

FIND_PACKAGE(SDL_gfx)

IF (SDLGFX_FOUND)
	MESSAGE(STATUS "SDL_gfx was found")
ELSE (SDLGFX_FOUND)
	MESSAGE(STATUS "SDL_gfx missing: needed for Spatial tools (MapExplorer, MapTools, etc.).")
ENDIF (SDLGFX_FOUND)

FIND_PACKAGE(TCL)

# /usr/bin/tclsh is expected for some embodiment tests, but is
# not otherwise important for anything...
IF (TCL_TCLSH)
	MESSAGE(STATUS "TCL shell was found here: ${TCL_TCLSH}")
ELSE (TCL_TCLSH)
	MESSAGE(STATUS "TCL shell missing: needed for some embodiment tests.")
ENDIF (TCL_TCLSH)



# Google Protobuf library
# The protocol buffer compiler is a separately installable package, and
# it is also needed for building embodiment PAI. So check for that, too.
FIND_PACKAGE(Protobuf)
IF (PROTOBUF_FOUND
		AND PROTOBUF_LIBRARY
		AND PROTOBUF_INCLUDE_DIR
		AND PROTOBUF_PROTOC_EXECUTABLE)

	ADD_DEFINITIONS(-DHAVE_PROTOBUF)
	SET(HAVE_PROTOBUF 1)
	SET(PROTOBUF_DIR_MESSAGE "Protobuf was found.")

ELSE (PROTOBUF_FOUND
		AND PROTOBUF_LIBRARY
		AND PROTOBUF_INCLUDE_DIR
		AND PROTOBUF_PROTOC_EXECUTABLE)

	# If we are here, then we are missing either the libraries, or the
	# compiler. React appropriately.
	IF (PROTOBUF_PROTOC_EXECUTABLE)
		SET(PROTOBUF_DIR_MESSAGE "Protobuf was not found; the new message decoding functions would not work.\n   To over-ride, make sure PROTOBUF_LIBRARIES and PROTOBUF_INCLUDE_DIRS are set.")
	ELSE (PROTOBUF_PROTOC_EXECUTABLE)
		SET(PROTOBUF_DIR_MESSAGE "Protobuf compiler was not found; embodiment PAI can't be built without it.\n   To over-ride, make sure PROTOBUF_PROTOC_EXECUTABLE is set.")
	ENDIF (PROTOBUF_PROTOC_EXECUTABLE)

ENDIF (PROTOBUF_FOUND
		AND PROTOBUF_LIBRARY
		AND PROTOBUF_INCLUDE_DIR
		AND PROTOBUF_PROTOC_EXECUTABLE)

MESSAGE(STATUS "${PROTOBUF_DIR_MESSAGE}")

# Python needed for embodiment (!?) and for PLN
FIND_PACKAGE(PythonInterp)
FIND_PACKAGE(PythonLibs)
IF(PYTHONLIBS_FOUND)
	MESSAGE(STATUS "Python libs found.")
ELSE(PYTHONLIBS_FOUND)
	MESSAGE(STATUS "Python libs NOT found.")
ENDIF(PYTHONLIBS_FOUND)

# Python bindings using Cython
# Cython is used to generate bindings
FIND_PROGRAM(CYTHON_EXECUTABLE cython)
FIND_PROGRAM(NOSETESTS_EXECUTABLE nosetests)
IF (CYTHON_EXECUTABLE AND PYTHONLIBS_FOUND)
   # find python destination dir for python bindings
   ADD_DEFINITIONS(-DHAVE_CYTHON)
   SET(HAVE_CYTHON 1)
   EXECUTE_PROCESS(
	   COMMAND ${PYTHON_EXECUTABLE} -c "from distutils.sysconfig import *; print get_python_lib()"
		OUTPUT_VARIABLE PYTHON_DEST
   )

   # replace new line at end
   STRING(REPLACE "\n" "" PYTHON_DEST "${PYTHON_DEST}")
   IF ("${PYTHON_DEST}" STREQUAL "")
	   MESSAGE(FATAL_ERROR "Python destination dir not found")
   ELSE ("${PYTHON_DEST}" STREQUAL "")
	   MESSAGE(STATUS "Python destination dir found: ${PYTHON_DEST}" )
   ENDIF ("${PYTHON_DEST}" STREQUAL "")
ELSE (CYTHON_EXECUTABLE AND PYTHONLIBS_FOUND)
	IF(NOT CYTHON_EXECUTABLE)
		MESSAGE(STATUS "Cython executable not found.")
	ENDIF(NOT CYTHON_EXECUTABLE)
ENDIF (CYTHON_EXECUTABLE AND PYTHONLIBS_FOUND)

# SDL needed for the spatial tools
FIND_PACKAGE(SDL)

IF (SDL_FOUND)
	MESSAGE(STATUS "SDL was found.")
ELSE (SDL_FOUND)
	MESSAGE(STATUS "SDL missing: needed for Spatial tools (MapExplorer, MapTools, etc.).")
ENDIF (SDL_FOUND)

FIND_PACKAGE(SDL_gfx)

IF (SDLGFX_FOUND)
	MESSAGE(STATUS "SDL_gfx was found")
ELSE (SDLGFX_FOUND)
	MESSAGE(STATUS "SDL_gfx missing: needed for Spatial tools (MapExplorer, MapTools, etc.).")
ENDIF (SDLGFX_FOUND)

FIND_PACKAGE(TCL)

# /usr/bin/tclsh is expected for some embodiment tests, but is
# not otherwise important for anything...
IF (TCL_TCLSH)
	MESSAGE(STATUS "TCL shell was found here: ${TCL_TCLSH}")
ELSE (TCL_TCLSH)
	MESSAGE(STATUS "TCL shell missing: needed for some embodiment tests.")
ENDIF (TCL_TCLSH)


# ==========================================================
# Decide what to build, based on the packages found.

IF(Boost_FOUND)
    SET(HAVE_UTIL 1)
    SET(HAVE_COMBOREDUCT 1)
    SET(HAVE_MOSES 1)
    SET(HAVE_FEATURE_SELECTION 1)
ENDIF(Boost_FOUND)

IF(GSL_FOUND AND HAVE_UTIL)
    SET(HAVE_ATOMSPACE 1)
    SET(HAVE_ATTENTION 1)
    SET(HAVE_PLN 1)
    SET(HAVE_DIMEMBED 1)
ENDIF(GSL_FOUND AND HAVE_UTIL)

IF(Boost_PROGRAM_OPTIONS_FOUND AND OPENGL_FOUND AND SDL_FOUND AND SDLGFX_FOUND)
	SET(HAVE_SPATIAL_TOOLS 1)
ENDIF(Boost_PROGRAM_OPTIONS_FOUND AND OPENGL_FOUND AND SDL_FOUND AND SDLGFX_FOUND)

IF(EXPAT_FOUND AND HAVE_ATOMSPACE)
	SET(HAVE_XML 1)
ENDIF(EXPAT_FOUND AND HAVE_ATOMSPACE)

IF(HAVE_ATOMSPACE)
	SET(HAVE_SERVER 1)
ENDIF(HAVE_ATOMSPACE)

IF(HAVE_SERVER) # because web depends on server
	SET(HAVE_WEB 1)
ENDIF(HAVE_SERVER)

IF(HAVE_ATOMSPACE AND ODBC_FOUND) # SQL persistance
	SET(HAVE_PERSIST 1)
ENDIF(HAVE_ATOMSPACE AND ODBC_FOUND)

# Here's the catch: much of embodiment relies on the MessagingSystem,
# which uses NMXML as the message format.  And NMXML needs expat,
# so embodiment can't build without expat.
IF(XERCESC_VERSION_OK AND HAVE_PROTOBUF AND HAVE_XML)
   SET(HAVE_EMBODIMENT 1)
ENDIF(XERCESC_VERSION_OK AND HAVE_PROTOBUF AND HAVE_XML)

IF (CURL_FOUND AND XMLRPC_FOUND)
	ADD_DEFINITIONS(-DHAVE_UBIGRAPH)
	SET(HAVE_UBIGRAPH 1)
ENDIF (CURL_FOUND AND XMLRPC_FOUND)

IF(HAVE_ATOMSPACE AND HAVE_GUILE AND HAVE_LINK_GRAMMAR)
	SET(HAVE_VITERBI 1)
ENDIF(HAVE_ATOMSPACE AND HAVE_GUILE AND HAVE_LINK_GRAMMAR)

