cmake_minimum_required ( VERSION 3.1.15 )
project ( OrbbecSDK VERSION 1.10.12 LANGUAGES C CXX )

# Include GNUInstallDirs for standard installation directories
include ( GNUInstallDirs )

# Determine host platform
include ( ${CMAKE_CURRENT_SOURCE_DIR}/cmake/DetermineHostPlatform.cmake )

if ( WIN32 )
    set ( ORBBEC_LIB_PREFIX "" )
    set ( ORBBEC_LIB_SUFFIX ".lib" )
    set ( ORBBEC_DLL_SUFFIX ".dll" )
elseif ( APPLE )
    set ( ORBBEC_LIB_PREFIX "lib" )
    set ( ORBBEC_LIB_SUFFIX ".dylib" )
    set ( ORBBEC_DLL_SUFFIX ".dylib" )
elseif ( UNIX )
    set ( ORBBEC_LIB_PREFIX "lib" )
    set ( ORBBEC_LIB_SUFFIX ".so" )
    set ( ORBBEC_DLL_SUFFIX ".so" )
endif ()

# OrbbecSDK library
set ( OrbbecSDK_LIBS_DIR ${CMAKE_CURRENT_SOURCE_DIR}/lib/${HOST_PLATFORM} )
set ( OrbbecSDK_LIBRARY ${OrbbecSDK_LIBS_DIR}/${ORBBEC_LIB_PREFIX}OrbbecSDK${ORBBEC_LIB_SUFFIX} )
set ( OrbbecSDK_DLL ${OrbbecSDK_LIBS_DIR}/${ORBBEC_LIB_PREFIX}OrbbecSDK${ORBBEC_DLL_SUFFIX} )
set ( OrbbecSDK_INCLUDE_DIRS "include" )
set ( OrbbecSDK_LIBS ${OrbbecSDK_LIBRARY} )
set ( OrbbecSDK_FOUND TRUE )

# Create an IMPORTED target for the pre-built library
add_library ( OrbbecSDK::OrbbecSDK SHARED IMPORTED GLOBAL )
set_target_properties ( OrbbecSDK::OrbbecSDK PROPERTIES
    IMPORTED_LOCATION ${OrbbecSDK_DLL}
    IMPORTED_IMPLIB ${OrbbecSDK_LIBRARY}
    INTERFACE_INCLUDE_DIRECTORIES "${CMAKE_CURRENT_SOURCE_DIR}/${OrbbecSDK_INCLUDE_DIRS}"
)

# Create a wrapper target that can be installed
add_library ( OrbbecSDK INTERFACE )
target_link_libraries ( OrbbecSDK INTERFACE OrbbecSDK::OrbbecSDK )
target_include_directories ( OrbbecSDK INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/${OrbbecSDK_INCLUDE_DIRS}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

# check if the project is being used directly or via add_subdirectory
if ( ${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_CURRENT_SOURCE_DIR} ) # project is used directly
    # Set the C++ standard to C++ 11 and C standard to C11
    set ( CMAKE_CXX_STANDARD 11 )
    set ( CMAKE_C_STANDARD 11 )

    # Options
    option ( BUILD_EXAMPLES "Build examples" ON )
    option ( INSTALL_LIBUDEV "Install libudev" OFF )

    # Compiler and Linker options
    set ( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC -O3" )
    set ( CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fPIC -g" )
    set ( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC -O3" )
    set ( CMAKE_BUILD_TYPE "Release" )

    # Set default install prefix if not specified
    if ( CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT )
        set ( CMAKE_INSTALL_PREFIX "/usr/local" CACHE PATH "Install prefix" FORCE )
    endif ()

    set ( CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib )
    set ( CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib )
    set ( CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin )

    if ( CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang" )
        add_compile_options ( -Wall -Wextra -Wpedantic )
    endif ()

    # paths
    set ( ORBBEC_EXAMPLES_DIR ${CMAKE_CURRENT_SOURCE_DIR}/examples/ )
    set ( ORBBEC_DOCS_DIR ${CMAKE_CURRENT_SOURCE_DIR}/doc/ )
    set ( ORBBEC_MISC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/misc/ )
    set ( ORBBEC_CONFIG_DIR ${ORBBEC_MISC_DIR}/config/ )
    set ( ORBBEC_SCRIPT_DIR ${ORBBEC_MISC_DIR}/scripts/ )

    # Subdirectories
    if ( BUILD_EXAMPLES )
        add_subdirectory ( examples )
    endif ()

    # Installation of headers, configs, libs
    include ( ${CMAKE_CURRENT_SOURCE_DIR}/cmake/InstallTargets.cmake )

    # Package configuration
    include ( CMakePackageConfigHelpers )

    configure_package_config_file (
        "${CMAKE_CURRENT_SOURCE_DIR}/cmake/OrbbecSDKConfig.cmake.in"
        "${CMAKE_CURRENT_BINARY_DIR}/OrbbecSDKConfig.cmake"
        INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/OrbbecSDK
        PATH_VARS CMAKE_INSTALL_INCLUDEDIR CMAKE_INSTALL_LIBDIR
    )

    write_basic_package_version_file (
        "${CMAKE_CURRENT_BINARY_DIR}/OrbbecSDKConfigVersion.cmake"
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion
    )

    install ( FILES
        "${CMAKE_CURRENT_BINARY_DIR}/OrbbecSDKConfig.cmake"
        "${CMAKE_CURRENT_BINARY_DIR}/OrbbecSDKConfigVersion.cmake"
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/OrbbecSDK
    )

    install ( TARGETS OrbbecSDK
        EXPORT OrbbecSDKTargets
        INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    )

    install ( EXPORT OrbbecSDKTargets
        FILE OrbbecSDKTargets.cmake
        NAMESPACE OrbbecSDK::
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/OrbbecSDK
    )

    # Install the actual library files
    install ( FILES ${OrbbecSDK_LIBRARY} ${OrbbecSDK_DLL}
        DESTINATION ${CMAKE_INSTALL_LIBDIR}
    )

    # Install header files
    install ( DIRECTORY ${OrbbecSDK_INCLUDE_DIRS}
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
        FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp"
    )

    # OrbbecSDK.pc.in
    if ( UNIX )
        configure_file (
            "${CMAKE_CURRENT_SOURCE_DIR}/OrbbecSDK.pc.in"
            "${CMAKE_CURRENT_BINARY_DIR}/OrbbecSDK.pc"
            @ONLY
        )
        install ( FILES
            "${CMAKE_CURRENT_BINARY_DIR}/OrbbecSDK.pc"
            DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig/
        )
    endif ()

    # Uninstall target
    include ( ${CMAKE_CURRENT_SOURCE_DIR}/cmake/UninstallTarget.cmake )
endif ()
