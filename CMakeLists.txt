#############################################
#
#  vcell-hy3s -- the Hy3S hybrid stochastic solvers used by the Virtual Cell
#  framework. Three executables from one Fortran source set, differing only in
#  the SDE integrator they compile in.
#
##############################################
cmake_minimum_required(VERSION 3.20)

if (POLICY CMP0057)
	cmake_policy(SET CMP0057 NEW)
endif ()

project(vcell-hy3s LANGUAGES C CXX Fortran)

#############################################
#  Declare options
##############################################
option(OPTION_TARGET_MESSAGING "Progress messaging (requires libcurl)" OFF)
option(OPTION_EXTRA_CONFIG_INFO "Print useful cmake debug info while configuring" OFF)

#############################################
#  Platform
##############################################
set(LINUX FALSE)
if (CMAKE_SYSTEM_NAME MATCHES "Linux")
	set(LINUX TRUE)
endif ()

# Windows would need gfortran under MSYS2/MinGW plus the vendored NetCDF built
# there. Not attempted; see README.md.
if (WIN32 AND NOT MINGW)
	message(FATAL_ERROR
			"On Windows this would need MSYS2/MinGW-w64 for gfortran. MSVC has no Fortran "
			"compiler, and the vendored NetCDF has not been built there.")
endif ()

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if (NOT DEFINED CMAKE_BUILD_TYPE OR CMAKE_BUILD_TYPE STREQUAL "")
	set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build" FORCE)
endif ()

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/modules/")
include(GetGitRevisionDescription)
git_describe(GIT_DESCRIBE)

#############################################
#  Fortran dialect
##############################################
# Hy3S was an Intel Fortran project. Its conditional compilation used Intel's
# !DEC$ IF directives, which gfortran does not implement -- it treats them as
# comments and compiles the guarded code regardless, so a messaging-free build
# would still try to link the messaging calls. Those 114 directives were
# converted to standard #if/#else/#endif when this repo was split out, which
# means the Fortran now has to go through the C preprocessor.
if (CMAKE_Fortran_COMPILER_ID STREQUAL "GNU")
	# -cpp is gfortran's spelling of Intel's -fpp. The line-length flag is for
	# the original sources, which assume a compiler with no 132-column limit.
	set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -cpp -ffree-line-length-none")
elseif (CMAKE_Fortran_COMPILER_ID MATCHES "Intel")
	set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -fpp")
else ()
	message(WARNING
			"Unrecognised Fortran compiler ${CMAKE_Fortran_COMPILER_ID}; the sources need "
			"the C preprocessor run over them and no flag is being added for that.")
endif ()

#############################################
#  Dependencies
##############################################
# NetCDF is vendored rather than taken from Conan. Conan Center publishes the C
# library only -- no .mod files and no netcdf-fortran recipe -- and Hy3S does
# `USE netcdf`. Building Unidata's netcdf-fortran through FetchContent does not
# work either: it references CMAKE_SOURCE_DIR internally and so insists on being
# the top-level project. The vendored 3.6.2 tree carries its own C, F77 and F90
# layers, is what this code was written against, and builds clean with modern
# gcc/gfortran.
add_subdirectory(netcdf)

if (OPTION_TARGET_MESSAGING)
	if (NOT EXISTS "${CMAKE_SOURCE_DIR}/vcell-messaging/CMakeLists.txt")
		message(FATAL_ERROR
				"The vcell-messaging submodule is empty. Run: git submodule update --init --recursive")
	endif ()
	find_package(CURL REQUIRED)
	set(THREADS_PREFER_PTHREAD_FLAG ON)
	find_package(Threads REQUIRED)
	add_subdirectory(vcell-messaging)
endif ()

add_subdirectory(Hy3S)

#############################################
#  Debug output
##############################################
if (OPTION_EXTRA_CONFIG_INFO)
	include(CMakePrintHelpers)
	cmake_print_variables(OPTION_TARGET_MESSAGING)
	cmake_print_variables(CMAKE_Fortran_COMPILER CMAKE_Fortran_COMPILER_ID CMAKE_Fortran_FLAGS)
	cmake_print_variables(CMAKE_C_COMPILER CMAKE_CXX_COMPILER)
endif ()
