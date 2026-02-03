project(SimpleITK_dotnet)

include_directories(${SimpleITK_INCLUDE_DIRS})

##################
# setup SimpleITK
##################
# CSharp version requirements: http://msdn.microsoft.com/en-us/library/system.reflection.assemblyversionattribute.aspx
# major.minor[.build[.revision]] where all components are 16-bit unsigned integers
set(_build 0)
if (DEFINED SimpleITK_VERSION_POST AND NOT "${SimpleITK_VERSION_POST}" STREQUAL "")
    math(EXPR _build "${SimpleITK_VERSION_POST}")
elseif (DEFINED SimpleITK_VERSION_DEV AND NOT "${SimpleITK_VERSION_DEV}" STREQUAL "")
    math(EXPR _build "32768+${SimpleITK_VERSION_DEV}")
endif ()
if (_build GREATER 65535)
    message(WARNING "CSharp build component overflowed, setting to 65535 instead of ${_build}.")
    set(_build 65535)
endif ()

set(_revision 0)
if (DEFINED SimpleITK_VERSION_PATCH)
    math(EXPR _revision "${SimpleITK_VERSION_PATCH}<<8")
    if (DEFINED SimpleITK_VERSION_TWEAK)
        math(EXPR _revision "${_revision}+${SimpleITK_VERSION_TWEAK}")
    endif ()
endif ()
if (_revision GREATER 65535)
    message(WARNING "CSharp revision component overflowed, setting to 65535 instead of ${_revision}.")
    set(_revision 65535)
endif ()

set(SimpleITK_VERSION_CSHARP_AssemblyVersion "${SimpleITK_VERSION_MAJOR}.${SimpleITK_VERSION_MINOR}.${_build}.${_revision}")

######################
# Setting Swig module
######################

# Make sure the nested directory structure exists
set(DOTNET_SOURCE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/CSharpSources CACHE INTERNAL "")
set(DOTNET_BINARY_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/CSharpBinaries CACHE INTERNAL "")
file(MAKE_DIRECTORY ${DOTNET_SOURCE_DIRECTORY})
file(MAKE_DIRECTORY ${DOTNET_BINARY_DIRECTORY})

# Create swig target
find_package(SWIG REQUIRED)
include(UseSWIG)

set(CMAKE_SWIG_OUTDIR ${DOTNET_SOURCE_DIRECTORY})
if (UNIX)
    set(CMAKE_SWIG_FLAGS -dllimport \"libSimpleITKDotnetNative\")
else (WIN32)
    set(CMAKE_SWIG_FLAGS -dllimport \"SimpleITKDotnetNative\")
endif ()

# 设置正确的 SWIG 库路径
set(SWIG_LIB_DIR "${SWIG_DIR}/Lib")
set(SWIG_CSHARP_LIB_DIR "${SWIG_DIR}/Lib/csharp")
message(STATUS "SWIG Lib directory: ${SWIG_LIB_DIR}")

if (WIN32)
    set(SWIG_EXTRA_ARGS)
elseif (UNIX)
    set(SWIG_EXTRA_ARGS -DSWIGWORDSIZE64)
endif ()

set(CMAKE_SWIG_FLAGS -I${CMAKE_CURRENT_SOURCE_DIR} -namespace \"itk.simple\" ${CMAKE_SWIG_GLOBAL_FLAGS} ${CMAKE_SWIG_FLAGS}
        -I"${SWIG_CSHARP_LIB_DIR}" -I"${SWIG_LIB_DIR}" ${SWIG_EXTRA_ARGS})
# set swig target
set(SWIG_MODULE_TARGET_NAME SimpleITK_SWIG_DOTNET_CODE)

#######################
##  DOTNET Setting  ##
#######################
if (BUILD_DOTNET) # From https://github.com/Mizux/cmake-swig.git
    # Targeted Framework Moniker
    # see: https://docs.microsoft.com/en-us/dotnet/standard/frameworks
    # see: https://learn.microsoft.com/en-us/dotnet/standard/net-standard
    # .Net Core 3.1 LTS is not available for osx arm64
    if (APPLE AND CMAKE_SYSTEM_PROCESSOR MATCHES "^(aarch64|arm64)")
        set(USE_DOTNET_CORE_31 OFF)
    else ()
        option(USE_DOTNET_CORE_31 "Use .Net Core 3.1 LTS support" OFF)
    endif ()
    message(STATUS ".Net: Use .Net Core 3.1 LTS support: ${USE_DOTNET_CORE_31}")

    option(USE_DOTNET_6 "Use .Net 6.0 LTS support" OFF)
    message(STATUS ".Net: Use .Net 6.0 LTS support: ${USE_DOTNET_6}")

    option(USE_DOTNET_7 "Use .Net 7.0 support" OFF)
    message(STATUS ".Net: Use .Net 7.0 support: ${USE_DOTNET_7}")

    option(USE_DOTNET_8 "Use .Net 8.0 LTS support" OFF)
    message(STATUS ".Net: Use .Net 8.0 support: ${USE_DOTNET_8}")

    option(USE_DOTNET_9 "Use .Net 9.0 support" OFF)
    message(STATUS ".Net: Use .Net 9.0 support: ${USE_DOTNET_9}")

    option(USE_DOTNET_10 "Use .Net 10.0 support" ON)
    message(STATUS ".Net: Use .Net 10.0 support: ${USE_DOTNET_10}")
endif ()

# Find dotnet cli
find_program(DOTNET_EXECUTABLE NAMES dotnet)
if (NOT DOTNET_EXECUTABLE)
    message(FATAL_ERROR "Check for dotnet Program: not found")
else ()
    message(STATUS "Found dotnet Program: ${DOTNET_EXECUTABLE}")
endif ()

# Needed by dotnet/CMakeLists.txt
set(DOTNET_PACKAGE_BASE SimpleITKDotnetUnmanaged)

# Add platform-specific suffix to package name
if (APPLE)
    set(DOTNET_PACKAGE_SUFFIX "macOS")
elseif (UNIX)
    set(DOTNET_PACKAGE_SUFFIX "Linux")
elseif (WIN32)
    set(DOTNET_PACKAGE_SUFFIX "Win")
else ()
    set(DOTNET_PACKAGE_SUFFIX "")
endif ()

set(DOTNET_PACKAGE ${DOTNET_PACKAGE_BASE}${DOTNET_PACKAGE_SUFFIX})
set(DOTNET_PACKAGES_DIR "${PROJECT_BINARY_DIR}/dotnet/packages")
file(MAKE_DIRECTORY ${DOTNET_PACKAGES_DIR})
set(DOTNET_PROJECT ${DOTNET_PACKAGE})
message(STATUS ".Net project: ${DOTNET_PROJECT}")
set(DOTNET_PROJECT_DIR ${PROJECT_BINARY_DIR}/${DOTNET_PROJECT})
message(STATUS ".Net project build path: ${DOTNET_PROJECT_DIR}")

# Runtime IDentifier
# see: https://docs.microsoft.com/en-us/dotnet/core/rid-catalog
if (CMAKE_SYSTEM_PROCESSOR MATCHES "^(aarch64|arm64)")
    set(DOTNET_PLATFORM arm64)
else ()
    set(DOTNET_PLATFORM x64)
endif ()

if (APPLE)
    set(DOTNET_RID osx-${DOTNET_PLATFORM})
elseif (UNIX)
    set(DOTNET_RID linux-${DOTNET_PLATFORM})
elseif (WIN32)
    set(DOTNET_RID win-${DOTNET_PLATFORM})
else ()
    message(FATAL_ERROR "Unsupported system !")
endif ()
message(STATUS ".Net RID: ${DOTNET_RID}")

# Targeted Framework Moniker
# see: https://docs.microsoft.com/en-us/dotnet/standard/frameworks
# Language Version
# see: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/configure-language-version
set(DOTNET_LANG "9.0" STRING "Specify the C# language version (default \"9.0\")")
if (USE_DOTNET_5)
    list(APPEND TFM "net5.0")
    set(DOTNET_LANG "10.0")
endif ()
if (USE_DOTNET_6)
    list(APPEND TFM "net6.0")
    set(DOTNET_LANG "10.0")
endif ()
if (USE_DOTNET_7)
    list(APPEND TFM "net7.0")
    set(DOTNET_LANG "11.0")
endif ()
if (USE_DOTNET_8)
    list(APPEND TFM "net8.0")
    set(DOTNET_LANG "12.0")
endif ()
if (USE_DOTNET_9)
    list(APPEND TFM "net9.0")
    set(DOTNET_LANG "13.0")
endif ()
if (USE_DOTNET_10)
    list(APPEND TFM "net10.0")
    set(DOTNET_LANG "14.0")
endif ()

list(LENGTH TFM TFM_LENGTH)
if (TFM_LENGTH EQUAL "0")
    message(FATAL_ERROR "No .Net SDK selected !")
endif ()

string(JOIN ";" DOTNET_TFM ${TFM})
message(STATUS ".Net TFM: ${DOTNET_TFM}")

# Create the native library
add_library(SimpleITKDotnetNative SHARED "")

target_link_libraries(SimpleITKDotnetNative PRIVATE ${SWIG_MODULE_TARGET_NAME})
set_target_properties(SimpleITKDotnetNative PROPERTIES
        POSITION_INDEPENDENT_CODE ON)
# On Windows, remove the "lib" prefix; on Unix/Linux, keep it
if (WIN32)
    set_target_properties(SimpleITKDotnetNative PROPERTIES PREFIX "")
endif ()
set_target_properties(SimpleITKDotnetNative PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${DOTNET_SOURCE_DIRECTORY})
# note: macOS is APPLE and also UNIX !
if (APPLE)
    set_target_properties(SimpleITKDotnetNative PROPERTIES INSTALL_RPATH "@loader_path")
    # Xcode fails to build if library doesn't contains at least one source file.
    if (XCODE)
        file(GENERATE
                OUTPUT ${PROJECT_BINARY_DIR}/SimpleITKDotnetNative/version.cpp
                CONTENT "namespace {char* version = \"${PROJECT_VERSION}\";}")
        target_sources(SimpleITKDotnetNative PRIVATE ${PROJECT_BINARY_DIR}/SimpleITKDotnetNative/version.cpp)
    endif ()
elseif (UNIX)
    set_target_properties(SimpleITKDotnetNative PROPERTIES INSTALL_RPATH "$ORIGIN")
endif ()

##################
# Add Swig module
##################
set_property(SOURCE ../SimpleITK.i PROPERTY CPLUSPLUS ON)
swig_add_library(${SWIG_MODULE_TARGET_NAME}
        TYPE OBJECT
        LANGUAGE csharp
        OUTPUT_DIR ${DOTNET_SOURCE_DIRECTORY}
        SOURCES ../SimpleITK.i)

get_directory_property(cmake_include_directories INCLUDE_DIRECTORIES)
target_include_directories(${SWIG_MODULE_TARGET_NAME} PRIVATE ${cmake_include_directories})
target_include_directories(${SWIG_MODULE_TARGET_NAME} PRIVATE ../../../Wrapping/Common)
target_include_directories(${SWIG_MODULE_TARGET_NAME} PRIVATE ../../../Wrapping/CSharp)
set_target_properties(${SWIG_MODULE_TARGET_NAME} PROPERTIES
        SWIG_USE_TARGET_INCLUDE_DIRECTORIES ON
        POSITION_INDEPENDENT_CODE ON)

target_link_libraries(${SWIG_MODULE_TARGET_NAME} ${SimpleITK_LIBRARIES})
if (APPLE)
    set_target_properties(${SWIG_MODULE_TARGET_NAME} PROPERTIES PREFIX "lib")
    set_target_properties(${SWIG_MODULE_TARGET_NAME} PROPERTIES SUFFIX ".dylib")
elseif (UNIX)
    set_target_properties(${SWIG_MODULE_TARGET_NAME} PROPERTIES PREFIX "lib")
    set_target_properties(${SWIG_MODULE_TARGET_NAME} PROPERTIES SUFFIX ".so")
else (WIN32)
    set_target_properties(${SWIG_MODULE_TARGET_NAME} PROPERTIES PREFIX "")
    set_target_properties(${SWIG_MODULE_TARGET_NAME} PROPERTIES SUFFIX ".dll")
    foreach (CMAKE_CONFIGURATION_TYPE ${CMAKE_CONFIGURATION_TYPES})
        string(TOUPPER ${CMAKE_CONFIGURATION_TYPE} CMAKE_CONFIGURATION_TYPE)
        set_target_properties(${SWIG_MODULE_TARGET_NAME}
                PROPERTIES LIBRARY_OUTPUT_DIRECTORY_${CMAKE_CONFIGURATION_TYPE} "${DOTNET_BINARY_DIRECTORY}")
        set_target_properties(${SWIG_MODULE_TARGET_NAME}
                PROPERTIES RUNTIME_OUTPUT_DIRECTORY_${CMAKE_CONFIGURATION_TYPE} "${DOTNET_BINARY_DIRECTORY}")
    endforeach ()
endif ()

# Configure AssemblyInfo.cs
#configure_file(
#        ${CMAKE_CURRENT_SOURCE_DIR}/../AssemblyInfo.cs.in
#        ${DOTNET_SOURCE_DIRECTORY}/AssemblyInfo.cs
#        @ONLY
#)


#######################
##  DOTNET WRAPPERS  ##
#######################
#file(COPY ${PROJECT_SOURCE_DIR}/dotnet/logo.png DESTINATION ${PROJECT_BINARY_DIR}/dotnet)
#set(DOTNET_LOGO_DIR "${PROJECT_BINARY_DIR}/dotnet")
#configure_file(${PROJECT_SOURCE_DIR}/dotnet/Directory.Build.props.in ${PROJECT_BINARY_DIR}/dotnet/Directory.Build.props)

############################
##  .Net Runtime Package  ##
############################
# *.csproj.in contains:
# CMake variable(s) (@PROJECT_NAME@) that configure_file() can manage and
# generator expression ($<TARGET_FILE:...>) that file(GENERATE) can manage.
# Use base package name for the template, but generate with platform-specific name
configure_file(
        ${PROJECT_SOURCE_DIR}/${DOTNET_PACKAGE_BASE}.csproj.in
        ${DOTNET_SOURCE_DIRECTORY}/${DOTNET_PACKAGE}.csproj.in
        @ONLY)

file(GENERATE
        OUTPUT ${DOTNET_SOURCE_DIRECTORY}/${DOTNET_PACKAGE}.csproj.in
        INPUT ${DOTNET_SOURCE_DIRECTORY}/${DOTNET_PACKAGE}.csproj.in)

add_custom_command(
        OUTPUT ${DOTNET_BINARY_DIRECTORY}/${DOTNET_PROJECT}.csproj
        COMMAND ${CMAKE_COMMAND} -E copy ${DOTNET_PACKAGE}.csproj.in ${DOTNET_PACKAGE}.csproj
        DEPENDS
        ${DOTNET_SOURCE_DIRECTORY}/${DOTNET_PACKAGE}.csproj.in
        WORKING_DIRECTORY ${DOTNET_SOURCE_DIRECTORY})

add_custom_command(
        OUTPUT ${DOTNET_BINARY_DIRECTORY}/timestamp
        COMMAND ${CMAKE_COMMAND} -E env --unset=TARGETNAME
        ${DOTNET_EXECUTABLE} build --nologo -c Release -p:Platform=${DOTNET_PLATFORM} ${DOTNET_PROJECT}.csproj
        COMMAND ${CMAKE_COMMAND} -E env --unset=TARGETNAME
        ${DOTNET_EXECUTABLE} pack --nologo -c Release -p:Platform=${DOTNET_PLATFORM} ${DOTNET_PROJECT}.csproj
        COMMAND ${CMAKE_COMMAND} -E touch ${DOTNET_BINARY_DIRECTORY}/timestamp
        DEPENDS
        #        ${PROJECT_BINARY_DIR}/dotnet/Directory.Build.props
        ${DOTNET_BINARY_DIRECTORY}/${DOTNET_PROJECT}.csproj
        ${SWIG_MODULE_TARGET_NAME}
        SimpleITKDotnetNative
        BYPRODUCTS
        ${DOTNET_BINARY_DIRECTORY}/bin
        ${DOTNET_BINARY_DIRECTORY}/obj
        VERBATIM
        COMMENT "Generate .Net native package ${DOTNET_BINARY_DIRECTORY} (${DOTNET_BINARY_DIRECTORY}/timestamp)"
        WORKING_DIRECTORY ${DOTNET_SOURCE_DIRECTORY})

add_custom_target(dotnet_native_package
        DEPENDS
        ${DOTNET_BINARY_DIRECTORY}/timestamp
        WORKING_DIRECTORY ${DOTNET_BINARY_DIRECTORY})

####################
##  .Net Package  ##
####################
#configure_file(
#        ${PROJECT_SOURCE_DIR}/${DOTNET_PROJECT}.csproj.in
#        ${DOTNET_PROJECT_DIR}/${DOTNET_PROJECT}.csproj.in
#        @ONLY)
#
#add_custom_command(
#        OUTPUT ${DOTNET_PROJECT_DIR}/${DOTNET_PROJECT}.csproj
#        COMMAND ${CMAKE_COMMAND} -E copy ${DOTNET_PROJECT}.csproj.in ${DOTNET_PROJECT}.csproj
#        DEPENDS
#        ${DOTNET_PROJECT_DIR}/${DOTNET_PROJECT}.csproj.in
#        WORKING_DIRECTORY ${DOTNET_PROJECT_DIR})
#
#add_custom_command(
#        OUTPUT ${DOTNET_PROJECT_DIR}/timestamp
#        COMMAND ${CMAKE_COMMAND} -E env --unset=TARGETNAME
#        ${DOTNET_EXECUTABLE} build --nologo -c Release -p:Platform=${DOTNET_PLATFORM} ${DOTNET_PROJECT}.csproj
#        COMMAND ${CMAKE_COMMAND} -E env --unset=TARGETNAME
#        ${DOTNET_EXECUTABLE} pack --nologo -c Release -p:Platform=${DOTNET_PLATFORM} ${DOTNET_PROJECT}.csproj
#        COMMAND ${CMAKE_COMMAND} -E touch ${DOTNET_PROJECT_DIR}/timestamp
#        DEPENDS
#        ${DOTNET_PROJECT_DIR}/${DOTNET_PROJECT}.csproj
#        ${DOTNET_BINARY_DIRECTORY}/timestamp
#        dotnet_native_package
#        BYPRODUCTS
#        ${DOTNET_PROJECT_DIR}/bin
#        ${DOTNET_PROJECT_DIR}/obj
#        VERBATIM
#        COMMENT "Generate .Net package ${DOTNET_PROJECT} (${DOTNET_PROJECT_DIR}/timestamp)"
#        WORKING_DIRECTORY ${DOTNET_PROJECT_DIR})
#
#add_custom_target(dotnet_package ALL
#        DEPENDS
#        ${DOTNET_PROJECT_DIR}/timestamp
#        WORKING_DIRECTORY ${DOTNET_PROJECT_DIR})