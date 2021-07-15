cmake_minimum_required(VERSION 3.8)
project(certify VERSION 0.1 LANGUAGES CXX)

list(INSERT CMAKE_MODULE_PATH 0 ${PROJECT_SOURCE_DIR}/CMakeModules)

if (CMAKE_VERSION VERSION_LESS 3.11)
    # Latest FindBoost.cmake has likely been updated to detect Boost version not yet released
    if (NOT EXISTS "${CMAKE_BINARY_DIR}/cmake/FindBoost.cmake")
        message(STATUS "Downloading FindBoost.cmake from https://gitlab.kitware.com/cmake/ release branch")
        file(
            DOWNLOAD
            "https://gitlab.kitware.com/cmake/cmake/raw/release/Modules/FindBoost.cmake"
            "${CMAKE_BINARY_DIR}/cmake/FindBoost.cmake")
    endif()
    list(INSERT CMAKE_MODULE_PATH 0 ${CMAKE_BINARY_DIR}/cmake)
endif()

include(FetchContent)

find_package(Boost 1.67
	COMPONENTS
		system
		filesystem
		date_time
		unit_test_framework
	REQUIRED)

find_package (Threads)

if(NOT ((TARGET OpenSSL::SSL) AND (TARGET OpenSSL::Crypto)))
    # OpenSSL
    if(UNIX)
        find_package(OpenSSL REQUIRED)
    else()
        FetchContent_Declare(openssl
            GIT_REPOSITORY https://github.com/LonghronShen/openssl-cmake.git
            GIT_TAG c773c832e0039f22aff68c88e65d8a37534045b7)

        FetchContent_GetProperties(openssl)
        if(NOT openssl_POPULATED)
            FetchContent_Populate(openssl)
            add_subdirectory(${openssl_SOURCE_DIR} ${openssl_BINARY_DIR} EXCLUDE_FROM_ALL)

            set(OPENSSL_FOUND TRUE CACHE BOOL "OPENSSL_FOUND" FORCE)

            add_library(OpenSSL::Crypto ALIAS crypto)
            add_library(OpenSSL::SSL ALIAS ssl)
        endif()
    endif()
endif()

add_library(certify::core INTERFACE)

target_compile_features(certify::core INTERFACE cxx_std_11)

target_include_directories(certify::core INTERFACE
    $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include>
    "${CMAKE_CURRENT_LIST_DIR}/include"
)

if(MSVC)
	target_link_libraries(certify::core INTERFACE Crypt32.lib)
endif()

target_link_libraries(
    	certify::core
    INTERFACE
        Boost::system
        Boost::filesystem
        Boost::date_time
        Threads::Threads
        OpenSSL::SSL
        OpenSSL::Crypto)

option(CERTIFY_INSTALL_PACKAGES "Install packages." OFF)

if(CERTIFY_INSTALL_PACKAGES)
    include(CMakePackageConfigHelpers)
    write_basic_package_version_file(
        "certifyConfigVersion.cmake"
        COMPATIBILITY AnyNewerVersion)

    install(FILES
                "netutilsConfig.cmake"
                "${CMAKE_BINARY_DIR}/certifyConfigVersion.cmake"
            DESTINATION lib/cmake/certify)

    install(DIRECTORY ${CMAKE_SOURCE_DIR}/include/
            DESTINATION include
            FILES_MATCHING PATTERN "*.hpp")

    install(TARGETS certify::core
            EXPORT certifyTargets
            INCLUDES DESTINATION include)

    install(EXPORT certifyTargets
            FILE certifyTargets.cmake
            NAMESPACE certify::
            DESTINATION lib/cmake/certify)
endif()

option(CERTIFY_BUILD_TESTING "Build unit tests for the project." OFF)

include(CTest)
if(CERTIFY_BUILD_TESTING)
    enable_testing()
    add_subdirectory(tests)
endif()
