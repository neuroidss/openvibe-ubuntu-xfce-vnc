project(Extras VERSION ${OPENVIBE_MAJOR_VERSION}.${OPENVIBE_MINOR_VERSION}.${OPENVIBE_PATCH_VERSION})

set(CMAKE_MODULE_PATH
	${CMAKE_MODULE_PATH_BASE}
	${CMAKE_CURRENT_SOURCE_DIR}/cmake-modules
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/cmake-modules
	# ${CMAKE_MODULE_PATH}
)

set(OV_GLOBAL_VERSION_STRING "${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH}")

set(OV_LAUNCHER_SOURCE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake-modules/launchers")
# OpenViBE-specific helper functions that are used by the CMakeLists of the subprojects.
#include("OvAddSingleProject")
include("OvAddProjects")
include("OvMessages")

add_definitions("-DOV_PROJECT_NAME=\"${PROJECT_NAME}\"")
add_definitions("-DOV_VERSION_MAJOR=\"${PROJECT_VERSION_MAJOR}\"")
add_definitions("-DOV_VERSION_MINOR=\"${PROJECT_VERSION_MINOR}\"")
add_definitions("-DOV_VERSION_PATCH=\"${PROJECT_VERSION_PATCH}\"")

set(OV_CONFIG_SUBDIR OpenVIBE CACHE STRING "Subdirectory under user directory when configuration and logs will be saved")

add_definitions("-DOV_CONFIG_SUBDIR=\"${OV_CONFIG_SUBDIR}\"")

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
if(WIN32)
	# Switch /arch:SSE2 enables vectorization. Remove if your CPU/compiler doesn't support it.
	if("${CMAKE_SIZEOF_VOID_P}" EQUAL "8")
	elseif("${CMAKE_SIZEOF_VOID_P}" EQUAL "4")
		set(OV_EIGEN_FLAGS "/arch:SSE2")
	endif()	
	# The following is needed to keep OpenMP from hogging all the cores. Note that this env var works only for VS2013+.
	# On VS2010, it may be better to disable OpenMP.
	set(OV_OMP_WAIT_POLICY "PASSIVE")
	# Disable /MP if you don't like VS using all the cores for compilation
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4355 /MP ${OV_EIGEN_FLAGS}")
	add_definitions("-DNOMINMAX -DBOOST_ALL_NO_LIB")
	# At least Windows Vista target - if you target Windows XP (0x0501), some components may not compile
	add_definitions("-D_WIN32_WINNT=0x0600")

	# Find out the compiler version, deps may use this
	set(MSVC_SERVICE_PACK "unknown")
	# message(STATUS "st ${CMAKE_CXX_COMPILER_VERSION}")
	if(${CMAKE_CXX_COMPILER_VERSION} VERSION_GREATER "18.0.0")
		set(MSVC_SERVICE_PACK "vc120")
	elseif(${CMAKE_CXX_COMPILER_VERSION} VERSION_GREATER "17.0.0")
		set(MSVC_SERVICE_PACK "vc110")
	elseif(${CMAKE_CXX_COMPILER_VERSION} VERSION_GREATER "16.0.0")
		set(MSVC_SERVICE_PACK "vc100")
	endif()

	string(REGEX MATCH "vc120.*" MSVC_VER120 ${MSVC_SERVICE_PACK})
	if(MSVC_VER120)
		# Boost 1.55 asio causes several compiler warnings on VS2013 we cannot do much about. try removing the following when boost is upgraded.
		add_definitions("-D_WINSOCK_DEPRECATED_NO_WARNINGS")

		# If OpenMP is available, using it will allow Eigen to use multiple cores in matrix math. Only used on VS120,
		# as VS100 allocates the cores too aggressively when OpenMP is enabled.
		include("FindOpenMP")
		if(OPENMP_FOUND)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
		endif()
	endif()
	message(STATUS "MSVC version is ${MSVC_SERVICE_PACK}")

	set(OV_WIN32_BOOST_VERSION "1_58")
elseif(UNIX)
	# If OpenMP is available, using it will allow Eigen to use multiple cores in matrix math.
	include("FindOpenMP")
	if(OPENMP_FOUND)
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
	endif()

	# Switch -msse2 enables vectorization. Remove if your CPU/compiler doesn't support it.
	#set(OV_EIGEN_FLAGS "-msse2")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall ${OV_EIGEN_FLAGS}")
	# This ENV var is needed to locate our custom IT++ on Linux
	set(ENV{PKG_CONFIG_PATH} "${LIST_DEPENDENCIES_PATH}/lib/pkgconfig")

	if (${CMAKE_SYSTEM_PROCESSOR} MATCHES i386|i586|i686)
		set ( BIT_MODE "32")
	else ()
		set ( BIT_MODE "64")
	endif ()

	if(EXISTS "/etc/debian_version")
		set (PLATFORM "Debian")
		add_definitions("-DTARGET_OS_Linux_Debian")
	endif(EXISTS "/etc/debian_version")

	if(EXISTS "/etc/fedora-release")
		set (PLATFORM "Fedora")
		add_definitions("-DTARGET_OS_Linux_Fedora")
	endif(EXISTS "/etc/fedora-release")
else()
	message(WARNING "Warning: unknown platform")
endif()

function(SET_BUILD_PLATFORM)
	if("${CMAKE_SIZEOF_VOID_P}" EQUAL "8")
		add_definitions(-DTARGET_ARCHITECTURE_x64)
		set(PLATFORM_TARGET "x64" PARENT_SCOPE)
	elseif("${CMAKE_SIZEOF_VOID_P}" EQUAL "4")
		add_definitions(-DTARGET_ARCHITECTURE_i386)
		set(PLATFORM_TARGET "x86" PARENT_SCOPE)
	else()
		add_definitions(-DTARGET_ARCHITECTURE_Unknown)
		set(PLATFORM_TARGET "unknown" PARENT_SCOPE)		
	endif()

	if(WIN32)
		add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
		add_definitions(-DTARGET_OS_Windows)
		add_definitions(-DTARGET_COMPILER_VisualStudio)
	elseif(APPLE)
		add_definitions(-fnon-call-exceptions)
		add_definitions(-DTARGET_OS_MacOS)
		add_definitions(-DTARGET_COMPILER_LLVM)
	elseif(UNIX)
		# add_definitions(-fvisibility=hidden) # This flag should be present... man gcc
		add_definitions(-fnon-call-exceptions)
		add_definitions(-DTARGET_OS_Linux)
		add_definitions(-DTARGET_COMPILER_GCC)
	endif()
endfunction()

# In extras/ we assume that everything is built onto the same platform. 
# It is very error prone to leave this to the modules.
SET_BUILD_PLATFORM()

# Print the used compilation parameters (for transparency)
get_directory_property(TMP_DEFINITIONS COMPILE_DEFINITIONS)
message(STATUS "Compilation flags used at source root: ")
message(STATUS "  COMPILE_DEFINITIONS = '${TMP_DEFINITIONS}'")
message(STATUS "  CMAKE_CXX_FLAGS = '${CMAKE_CXX_FLAGS}'")
message(STATUS "  CMAKE_CXX_FLAGS_RELEASE = '${CMAKE_CXX_FLAGS_RELEASE}'")
message(STATUS "  CMAKE_CXX_FLAGS_DEBUG = '${CMAKE_CXX_FLAGS_DEBUG}'")

# if want to compile tests, use the following.
set(OV_COMPILE_TESTS "true")

# if you don't want a GTK dependency, set the following. Note that most applications including Acquisition Server and Designer will NOT be built.
# This option is mostly of interest to hackers who are interested in using a minimal OpenViBE kernel in their own projects.
# set(OV_DISABLE_GTK "true")

# By setting SKIP[_FOLDER]* you can skip a subtree (example: SKIP_A_B_C skips folder a/b/c and all its subfolders if any)


# Only build the python plugin for one version of python.
# On windows, it can work with both, with the right configuration.
# On linux, compilation is easier, but runtime is not possible with both versions. Python.h is not namespaced and
# both plugins end up using the same Python API version (e.g. Python3 plugin calling python2 API).
set(SKIP_CONTRIB_PLUGINS_PROCESSING_PYTHON2 "1")
#set(SKIP_CONTRIB_PLUGINS_PROCESSING_PYTHON3 "1")

# Skipping demos can speed up compilation a lot, if you don't need them
#set(SKIP_APPLICATIONS_DEMOS "1")
#set(SKIP_CONTRIB_APPLICATIONS "1")

if(UNIX)
	# On Linux, the presence of matlab plugin .so can cause crashes in Simple DSP box as the two may use different versions of boost which are
	# then wrongly exposed to the other party due to the dlopen() flag RTLD_GLOBAL. Unfortunately this flag is needed or the python plugin
	# imports no longer work... You may enable the Matlab plugin compilation and possibly set the flag to RTLD_LOCAL in ovCKernelLoader.cpp,
	# but be aware of the consequences.
	message(STATUS "Note: Disabling Matlab plugin compilation by default")
	set(SKIP_PLUGINS_PROCESSING_MATLAB "1")
endif()

#set(SKIP_MODULES_AUTOMATON "1")
#set(SKIP_MODULES_EBML "1")
#set(SKIP_MODULES_FS "1")
#set(SKIP_MODULES_SOCKET "1")
#set(SKIP_MODULES_STREAM "1")
#set(SKIP_MODULES_SYSTEM "1")
#set(SKIP_MODULES_XML "1")
#set(SKIP_APPLICATIONS_PLATFORM_ACQUISITION-SERVER "1")
#set(SKIP_PLUGINS_PROCESSING_EXAMPLES "1")

# FIXME CERT > From OpenViBE designer repo in the future
# Designer is currently in another repository; taken from there
set(SKIP_APPLICATIONS_PLATFORM_DESIGNER "0")
set(SKIP_APPLICATIONS_EXAMPLES_CONVERT "0")		# Convert depends on Designer

# If you wish you can uncomment the next line and add a file with additional information called OVBranches.cmake to
# the cmake-modules folder. It makes modifying this very CMake file easier.
#include("OVCustomSettings")

# Used by the various Find* scripts to locate OpenViBE modules
# Used by the various Find* scripts to locate OpenViBE modules
set(OV_BASE_DIR ${CMAKE_CURRENT_SOURCE_DIR})

# needed for making visual studio projects when this script is called without CMAKE_BUILD_TYPE
set_property(DIRECTORY APPEND PROPERTY COMPILE_DEFINITIONS_DEBUG "TARGET_BUILDTYPE_Debug")
set_property(DIRECTORY APPEND PROPERTY COMPILE_DEFINITIONS_RELEASE "TARGET_BUILDTYPE_Release")
set_property(DIRECTORY APPEND PROPERTY COMPILE_DEFINITIONS_RELWITHDEBINFO "TARGET_BUILDTYPE_Release")

set(CMAKE_INSTALL_FULL_DOCDIR "${CMAKE_INSTALL_FULL_BINDIR}/../doc/html")

# Traverse these directories and build their components
add_subdirectory("modules/")
add_subdirectory("plugins/")
add_subdirectory("applications/")
add_subdirectory("externals/")
add_subdirectory("contrib/")

# make a copy of the dependency script, this is done so we can use the same launch scripts both in win install and win devel situations.
if(WIN32)
	find_path(OV_DEPENDENCY_CMD_PATH windows-dependencies.cmd PATHS ${LIST_DEPENDENCIES_PATH} NO_DEFAULT_PATH)
	if(NOT OV_DEPENDENCY_CMD_PATH)
		message(FATAL_ERROR "Dependency installer should have generated 'windows-dependencies.cmd'. Looked from ${LIST_DEPENDENCIES_PATH}.")
	endif()
	install(PROGRAMS ${OV_DEPENDENCY_CMD_PATH}/windows-dependencies.cmd DESTINATION ${DIST_BINDIR} RENAME "openvibe-set-env.cmd")
endif(WIN32)


# ----------------------
# Generate 'external application launcher' script
# ----------------------
if(WIN32)
	set(OV_EXT_CMD_FILE "openvibe-external-application-launcher.cmd")
elseif(UNIX)
	set(OV_EXT_CMD_FILE "openvibe-external-application-launcher")
endif()

configure_file("${OV_LAUNCHER_SOURCE_PATH}/${OV_EXT_CMD_FILE}-base" "${CMAKE_CURRENT_BINARY_DIR}/${OV_EXT_CMD_FILE}")
install(PROGRAMS ${CMAKE_CURRENT_BINARY_DIR}/${OV_EXT_CMD_FILE} DESTINATION ${DIST_BINDIR})

configure_file(test/CTestTestfile.cmake . @ONLY)

# Creates variables needed for .nsi generation
string(REPLACE "\." "" PROJECT_VERSION_SHORT ${PROJECT_VERSION})
set(ARCH_BITS "32")
set(ARCH_PROGFILES "$PROGRAMFILES")
if(${BUILD_ARCH} STREQUAL "x64")
  set(ARCH_BITS "64")
  set(ARCH_PROGFILES "$PROGRAMFILES64")
endif()
configure_file(scripts/windows-openvibe-x.x.x-setup.nsi.in scripts/windows-openvibe-${PROJECT_VERSION}-setup-${BUILD_ARCH}.nsi @ONLY)

# -----------------------------
# Install files
# -----------------------------
install(DIRECTORY signals DESTINATION ${DIST_DATADIR}/openvibe/scenarios/)
