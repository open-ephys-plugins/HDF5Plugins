cmake_minimum_required(VERSION 3.5.0)
if (NOT DEFINED GUI_BASE_DIR)
	if (DEFINED ENV{GUI_BASE_DIR})
		set(GUI_BASE_DIR $ENV{GUI_BASE_DIR})
	else()
		set(GUI_BASE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../../plugin-GUI)
	endif()
endif()

get_filename_component(PROJECT_FOLDER ${CMAKE_CURRENT_SOURCE_DIR} ABSOLUTE)
get_filename_component(COMMONLIB_NAME ${PROJECT_FOLDER} NAME)

project(OE_COMMONLIB_${COMMONLIB_NAME})
set(CMAKE_SHARED_LIBRARY_PREFIX "")
if(${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
	set(LINUX 1)
endif()

set_property(DIRECTORY APPEND PROPERTY COMPILE_DEFINITIONS
	"$<$<PLATFORM_ID:Windows>:JUCE_API=__declspec(dllimport)>"
	$<$<PLATFORM_ID:Windows>:_CRT_SECURE_NO_WARNINGS>
	$<$<PLATFORM_ID:Linux>:JUCE_DISABLE_NATIVE_FILECHOOSERS=1>
	$<$<CONFIG:Debug>:DEBUG=1>
	$<$<CONFIG:Debug>:_DEBUG=1>
	$<$<CONFIG:Release>:NDEBUG=1>
	)
	
set(SOURCE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/Source)
file(GLOB_RECURSE SRC_FILES LIST_DIRECTORIES false "${SOURCE_PATH}/*.cpp")
file(GLOB_RECURSE HEADER_FILES LIST_DIRECTORIES false "${SOURCE_PATH}/*.h")
list(APPEND SRC_FILES ${HEADER_FILES})
set(GUI_COMMONLIB_DIR ${GUI_BASE_DIR}/installed_libs)
list(APPEND CMAKE_PREFIX_PATH ${GUI_COMMONLIB_DIR})

add_library(${COMMONLIB_NAME} SHARED ${SRC_FILES})

target_compile_features(${COMMONLIB_NAME} PUBLIC cxx_auto_type cxx_generalized_initializers)
target_include_directories(${COMMONLIB_NAME} PUBLIC ${GUI_BASE_DIR}/JuceLibraryCode ${GUI_BASE_DIR}/JuceLibraryCode/modules ${GUI_BASE_DIR}/Plugins/Headers ${GUI_COMMONLIB_DIR}/include)

set(GUI_BIN_DIR ${GUI_BASE_DIR}/Build/$<IF:$<CONFIG:Debug>,Debug,Release>)

if (NOT CMAKE_LIBRARY_ARCHITECTURE)
	if (CMAKE_SIZEOF_VOID_P EQUAL 8)
		set(CMAKE_LIBRARY_ARCHITECTURE "x64")
	else()
		set(CMAKE_LIBRARY_ARCHITECTURE "x86")
	endif()
endif()

install(TARGETS ${COMMONLIB_NAME}
	LIBRARY DESTINATION ${GUI_COMMONLIB_DIR}/lib/${CMAKE_LIBRARY_ARCHITECTURE}
	CONFIGURATIONS ${CMAKE_CONFIGURATION_TYPES}
	ARCHIVE DESTINATION ${GUI_COMMONLIB_DIR}/lib/${CMAKE_LIBRARY_ARCHITECTURE}
	CONFIGURATIONS ${CMAKE_CONFIGURATION_TYPES}
	)
install(FILES ${HEADER_FILES} DESTINATION ${GUI_COMMONLIB_DIR}/include)
	
#Libraries and compiler options
if(MSVC)
	target_link_libraries(${COMMONLIB_NAME} ${GUI_BIN_DIR}/open-ephys.lib)
	target_compile_options(${COMMONLIB_NAME} PRIVATE /sdl-)
	
	install(TARGETS ${COMMONLIB_NAME} RUNTIME DESTINATION ${GUI_BIN_DIR}/shared CONFIGURATIONS ${CMAKE_CONFIGURATION_TYPES})
	list(APPEND CMAKE_PREFIX_PATH ${CMAKE_CURRENT_SOURCE_DIR}/../libs)
elseif(LINUX)
	set(GUI_BIN_DIR ${GUI_BASE_DIR}/Builds/Linux/build)

	target_link_libraries(${COMMONLIB_NAME} GL X11 Xext Xinerama asound dl freetype pthread rt)
	set_property(TARGET ${COMMONLIB_NAME} APPEND_STRING PROPERTY LINK_FLAGS
		"-fvisibility=hidden -fPIC -rdynamic -Wl,-rpath,'$$ORIGIN'")
	target_compile_options(${COMMONLIB_NAME} PRIVATE -fPIC -rdynamic)
	target_compile_options(${COMMONLIB_NAME} PRIVATE -O3) #enable optimization for linux debug
	
	install(TARGETS ${COMMONLIB_NAME} LIBRARY DESTINATION ${GUI_BIN_DIR}/shared)
elseif(APPLE)
	set(GUI_BIN_DIR ${GUI_BASE_DIR}/Builds/MacOSX/build/)
	set_target_properties(${COMMONLIB_NAME} PROPERTIES BUNDLE FALSE)
	set_property(TARGET ${COMMONLIB_NAME} APPEND_STRING PROPERTY LINK_FLAGS
	"-undefined dynamic_lookup")

	install(TARGETS ${COMMONLIB_NAME} DESTINATION $ENV{HOME}/Library/Application\ Support/open-ephys/shared)
	list(APPEND CMAKE_PREFIX_PATH /opt/local)
endif()

#create filters for vs and xcode

foreach( src_file IN ITEMS ${SRC_FILES})
	get_filename_component(src_path "${src_file}" PATH)
	file(RELATIVE_PATH src_path_rel "${SOURCE_PATH}" "${src_path}")
	string(REPLACE "/" "\\" group_name "${src_path_rel}")
	source_group("${group_name}" FILES "${src_file}")
endforeach()

if (APPLE) #this is needed by the unorthodox way hdf5 version 18 is installed
	set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH};/opt/local/lib/hdf5-18)
endif()
#additional libraries needed
find_package(HDF5 1.8 EXACT COMPONENTS CXX)
if(NOT HDF5_FOUND) #if package finding fails, try manually
#unset variables in case a wrong version has been found
unset(HDF5_LIBRARIES)
unset(HDF5_CXX_LIBRARIES)
unset(HDF5_INCLUDE_DIRS)
find_library(HDF5_LIBRARIES NAMES hdf5)
find_library(HDF5_CXX_LIBRARIES NAMES hdf5_cpp)
find_path(HDF5_INCLUDE_DIRS H5Cpp.h)
endif()

target_link_libraries(${COMMONLIB_NAME} ${HDF5_LIBRARIES} ${HDF5_CXX_LIBRARIES})
target_include_directories(${COMMONLIB_NAME} PRIVATE ${HDF5_INCLUDE_DIRS})

#on windows, copy the dlls when installing
if (MSVC)
	install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/../libs/bin/${CMAKE_LIBRARY_ARCHITECTURE}/ DESTINATION ${GUI_BIN_DIR}/Debug/bin/shared CONFIGURATIONS Debug)
	install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/../libs/bin/${CMAKE_LIBRARY_ARCHITECTURE}/ DESTINATION ${GUI_BIN_DIR}/Release/bin/shared CONFIGURATIONS Release)
endif()
