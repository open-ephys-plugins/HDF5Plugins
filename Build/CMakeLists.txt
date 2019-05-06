cmake_minimum_required(VERSION 3.5.0)
project(OE_HDF5)

add_subdirectory(../OpenEphysHDF5Lib/Builds HDF5Lib)
add_subdirectory(../KWIKFormat/Builds KWIKFormat)
add_subdirectory(../NWBFormat/Builds NWBFormat)
add_dependencies(KWIKFormat OpenEphysHDF5Lib)
add_dependencies(NWBFormat OpenEphysHDF5Lib)