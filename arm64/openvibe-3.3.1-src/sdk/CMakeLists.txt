project(OpenVIBE_SDK VERSION ${OPENVIBE_MAJOR_VERSION}.${OPENVIBE_MINOR_VERSION}.${OPENVIBE_PATCH_VERSION})
# Add cmake-modules to path
set(CMAKE_MODULE_PATH
	${CMAKE_MODULE_PATH_BASE}
	${CMAKE_SOURCE_DIR}/sdk/cmake-modules
	# ${CMAKE_MODULE_PATH}
	)
include("SetProjectVersion")

set_version()
OV_SET_PROJECT_VERSION()
message(STATUS "BUILDING VERSION OpenViBE SDK: ${OV_GLOBAL_VERSION_STRING} Branch : ${OV_PROJECT_BRANCH}~${OV_PROJECT_COMMITHASH}")

# Default is to build to dist/. If you wish a custom install, set your own MAKE_INSTALL_PREFIX when you call CMake. Safest to do under a fakeroot.
message(STATUS "Appending compilation flags...")


set(OV_SOURCE_DEPENDENCIES_PATH "${CMAKE_SOURCE_DIR}/sdk/dependencies-source")

# ----------------------
# Build CMake flags
# ----------------------
set(BRAND_NAME "OpenViBE" CACHE STRING "Brand with which titles and documentation will be prefixed")
string(TIMESTAMP COPYRIGHT_DATE "2012-%Y")
if(BRAND_NAME STREQUAL "NeuroRT")
	set(COMPANY_NAME "Mensia Technologies SA")
else()
	set(COMPANY_NAME "Inria")
endif()

# Options and cache variables

option(PUBLISH_DOC_ASSETS "Publish plugin documentation assets" ON)
option(BUILD_VALIDATION_TEST "Build the validation tests" ON)
option(BUILD_UNIT_TEST "Build the unit tests" ON)


if(NOT(${Flag_VerboseOutput}))
	set(CMAKE_INSTALL_MESSAGE LAZY)
endif()

# If OpenMP is available, using it will allow Eigen to use multiple cores in matrix math.
if(NOT(APPLE))
	include("FindOpenMP")
	if(OPENMP_FOUND)
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
	endif(OPENMP_FOUND)
endif()

add_definitions("-DOV_CONFIG_SUBDIR=\"${OV_CONFIG_SUBDIR}\"")
add_definitions("-DOV_PROJECT_NAME=\"${PROJECT_NAME}\"")
add_definitions("-DOV_VERSION_MAJOR=\"${OV_GLOBAL_VERSION_MAJOR}\"")
add_definitions("-DOV_VERSION_MINOR=\"${OV_GLOBAL_VERSION_MINOR}\"")
add_definitions("-DOV_VERSION_PATCH=\"${OV_GLOBAL_VERSION_PATCH}\"")

if(WIN32)
	add_definitions("-DNOMINMAX -DBOOST_ALL_NO_LIB")
	# The following is needed to keep OpenMP from hogging all the cores. Note that this works only for VS2013+. On VS2010, it may be better to disable OpenMP.
	set(OV_OMP_WAIT_POLICY "PASSIVE")
	# Disable /MP if you don't like VS using all the cores for compilation
	
	# Switch /arch:SSE2 enables vectorization. Remove if your CPU/compiler doesn't support it.
	# x64 build do not provide SSE2 option because 64bit processors always supports SSE vectorization.
	# So the /arch:SSE2 is not recognized for x64 build we have to only activate it for x86.
	if("${CMAKE_SIZEOF_VOID_P}" EQUAL "8")
	elseif("${CMAKE_SIZEOF_VOID_P}" EQUAL "4")
		set(OV_EIGEN_FLAGS "/arch:SSE2")
	endif()
	
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4355 /MP ${OV_EIGEN_FLAGS}")
	set(OV_WIN32_BOOST_VERSION "1_58")
elseif(UNIX)
	# Switch -msse2 enables vectorization. Remove if your CPU/compiler doesn't support it.
	#set(OV_EIGEN_FLAGS "-msse2")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall ${OV_EIGEN_FLAGS}")
else()
	message(WARNING "Warning: unknown platform")
endif()

# Print the used compilation parameters (for transparency)
get_directory_property(TMP_DEFINITIONS COMPILE_DEFINITIONS)
message(STATUS "Compilation flags used at source root: ")
message(STATUS "  COMPILE_DEFINITIONS = '${TMP_DEFINITIONS}'")
message(STATUS "  CMAKE_CXX_FLAGS = '${CMAKE_CXX_FLAGS}'")
message(STATUS "  CMAKE_CXX_FLAGS_RELEASE = '${CMAKE_CXX_FLAGS_RELEASE}'")
message(STATUS "  CMAKE_CXX_FLAGS_DEBUG = '${CMAKE_CXX_FLAGS_DEBUG}'")

function(SET_BUILD_PLATFORM)
	if("${CMAKE_SIZEOF_VOID_P}" EQUAL "8")
		add_definitions(-DTARGET_ARCHITECTURE_x64)
	elseif("${CMAKE_SIZEOF_VOID_P}" EQUAL "4")
		add_definitions(-DTARGET_ARCHITECTURE_i386)
		SET(PLATFORM_TARGET "x86")
	else()
		add_definitions(-DTARGET_ARCHITECTURE_Unknown)
	endif()

	if(WIN32)
		add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
		add_definitions(-DTARGET_OS_Windows)
		add_definitions(-DTARGET_COMPILER_VisualStudio)
	elseif(APPLE)
		add_definitions(-fnon-call-exceptions)
		add_definitions(-DTARGET_OS_MacOS)
		# ADD_DEFINITIONS(-DTARGET_ARCHITECTURE_x64)
		add_definitions(-DTARGET_COMPILER_LLVM)
	elseif(UNIX)
		# ADD_DEFINITIONS(-fvisibility=hidden) # This flag should be present... man gcc
		add_definitions(-fnon-call-exceptions)
		add_definitions(-DTARGET_OS_Linux)
		add_definitions(-DTARGET_COMPILER_GCC)
	endif()
endfunction()

# Custom cmakelist can be used to overwrite the default compilation & packaging parameters
# e.g. OEM_DISTRIBUTION
if(EXISTS "${CMAKE_SOURCE_DIR}/sdk/CustomCMakeLists.txt")
	message(STATUS "Found custom build settings")
	include("${CMAKE_SOURCE_DIR}/sdk/CustomCMakeLists.txt")
endif()

#################################################################
string(TOLOWER ${BRAND_NAME} LOWER_BRAND_NAME)
add_definitions(-DBRAND_NAME="${BRAND_NAME}")
if(${BRAND_NAME} STREQUAL "NeuroRT")
	add_definitions(-DOV_LOCAL_SYMBOLS)
endif()

if(OV_DISPLAY_ERROR_LOCATION)
	add_definitions(-DOV_DISPLAY_ERROR_LOCATION)
endif()
#################################################################

set(OV_LAUNCHER_SOURCE_PATH "${CMAKE_SOURCE_DIR}/sdk/cmake-modules/launchers")


# Used by the various Find* scripts to locate OpenViBE modules
set(OV_BASE_DIR ${CMAKE_SOURCE_DIR}/sdk)
set(OV_BASE_BIN_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(OV_LIBRARY_DIR  ${CMAKE_CURRENT_BINARY_DIR}/lib)

# needed for making visual studio projects when this script is called without CMAKE_BUILD_TYPE
set_property(DIRECTORY APPEND PROPERTY COMPILE_DEFINITIONS_DEBUG "TARGET_BUILDTYPE_Debug")
set_property(DIRECTORY APPEND PROPERTY COMPILE_DEFINITIONS_RELEASE "TARGET_BUILDTYPE_Release")
set_property(DIRECTORY APPEND PROPERTY COMPILE_DEFINITIONS_RELWITHDEBINFO "TARGET_BUILDTYPE_Release")

# OpenViBE-specific helper functions that are used by the CMakeLists of the subprojects.
include("OvAddProjects")
include("OvDelayLoad")

# Traverse these directories and build their components
add_subdirectory("build-tool/")

add_subdirectory("common/")

if(NOT(SKIP_OPENVIBE))
    add_subdirectory("openvibe/")
endif()

if(NOT(SKIP_KERNEL))
    add_subdirectory("kernel/")
endif()

add_subdirectory("modules/")

if(NOT(SKIP_TOOLKIT))
    add_subdirectory("toolkit/")
endif()

if(NOT(SKIP_PLUGINS))
    add_subdirectory("plugins/")
endif()

if(NOT(SKIP_APPLICATIONS))
    add_subdirectory("applications/")
endif()

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/.install-sdk" DESTINATION ${DIST_ROOT})


option(BUILD_TESTING "Include and build unit tests" ON)

if(BUILD_TESTING)
	set(OVT_LOG_DIR "${CMAKE_CURRENT_BINARY_DIR}/test/log" CACHE PATH "Path to directory containing test logs")
	set(OVT_TEST_DATA_DIR "${LIST_DEPENDENCIES_PATH}/test-input" CACHE PATH "Path to test input data files")
endif()

# add the scripts to the project so IDEs using the CMake file are aware of them
file(GLOB_RECURSE script_files scripts/*.cmd scripts/*.sh scripts/*.nsi scripts/*.cmake)
add_custom_target(openvibe-scripts SOURCES ${script_files})
set_target_properties(openvibe-scripts PROPERTIES
	FOLDER ${MISC_FOLDER}
)

if(BUILD_UNIT_TEST) 
	add_subdirectory("unit-test/")
endif()
if(BUILD_VALIDATION_TEST) 
	add_subdirectory("validation-test/")
endif()
