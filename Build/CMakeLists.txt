cmake_minimum_required(VERSION 3.5.0)
project(OE_HDF5)

add_subdirectory(../OpenEphysHDF5Lib/Build HDF5Lib)
add_subdirectory(../KWIKFormat/Build KWIKFormat)
add_subdirectory(../NWBFormat/Build NWBFormat)
add_dependencies(KWIKFormat OpenEphysHDF5Lib)
add_dependencies(NWBFormat OpenEphysHDF5Lib)
