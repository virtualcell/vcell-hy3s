project(netcdf_f90 Fortran)

set(SRC_FILES 
	netcdf.f90
	typeSizes.f90
)

# The compiler comes from the toolchain. This used to hardcode Intel Fortran,
# which is what the whole of Hy3S was built with before the split.
# -fpp and /iface: are Intel Fortran spellings. The preprocessor flag for the
# compiler actually in use is set once at the top level, so nothing is needed
# here.

add_library(netcdf_f90 STATIC ${SRC_FILES})

