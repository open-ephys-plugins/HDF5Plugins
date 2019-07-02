cmake_minimum_required(VERSION 3.5.0)
project(OE_HDF5)

add_subdirectory(OpenEphysHDF5Lib)
add_subdirectory(KWIKFormat)
add_subdirectory(NWBFormat)
add_dependencies(KWIKFormat OpenEphysHDF5Lib)
add_dependencies(NWBFormat OpenEphysHDF5Lib)
