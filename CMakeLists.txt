cmake_minimum_required(VERSION 4.0)

project(zstd
    VERSION 1.5.7
    DESCRIPTION "Fast real-time compression algorithm"
    LANGUAGES C
)

if(WIN32)
    enable_language(RC)
endif()
if(NOT DEFINED CMAKE_BUILD_TYPE AND NOT DEFINED CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE "Release")
endif()
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_VISIBILITY_INLINES_HIDDEN ON)
set(CMAKE_C_VISIBILITY_PRESET "hidden")
if(NOT "x${CMAKE_C_FLAGS}" STREQUAL "x")
    if(MSVC)
        string(REGEX REPLACE " [/-]W[01234] " " " CMAKE_C_FLAGS ${CMAKE_C_FLAGS})
    else()
        string(REGEX REPLACE " -W(all)?(extra)? " " " CMAKE_C_FLAGS ${CMAKE_C_FLAGS})
        string(REGEX REPLACE " -[W]?pedantic " " " CMAKE_C_FLAGS ${CMAKE_C_FLAGS})
    endif()
endif()
string(APPEND CMAKE_C_FLAGS " -w ")

set(__top_level ON)
if("x${CMAKE_SOURCE_DIR}" STREQUAL "x${PROJECT_SOURCE_DIR}")
    message("ZSTD: Building as a top-level project.")
else()
    set(__top_level OFF)
    message("ZSTD: Building as a subproject.")
endif()

option(ZSTD_BUILD_SHARED "Build shared library edition of ZSTD." ON)
option(ZSTD_BUILD_STATIC "Build static library edition of ZSTD." ON)
option(ZSTD_INSTALL "Enable installation for ZSTD." OFF)
if(NOT ZSTD_BUILD_SHARED AND NOT ZSTD_BUILD_STATIC)
    message(SEND_ERROR "You MUST build either the shared library or the static library or both of them!")
endif()
set(__targets)
if(ZSTD_BUILD_SHARED)
    add_library(${PROJECT_NAME}_shared SHARED)
    add_library(${PROJECT_NAME}::${PROJECT_NAME}_shared ALIAS ${PROJECT_NAME}_shared)
    set(__dllname ${PROJECT_NAME})
    if(NOT MSVC)
        set(__dllname lib${PROJECT_NAME})
    endif()
    set_target_properties(${PROJECT_NAME}_shared PROPERTIES OUTPUT_NAME ${__dllname})
    target_compile_definitions(${PROJECT_NAME}_shared PRIVATE
        ZSTD_DLL_EXPORT=1
    )
    if(WIN32)
        target_sources(${PROJECT_NAME}_shared PRIVATE ${PROJECT_NAME}.rc)
    endif()
    list(APPEND __targets ${PROJECT_NAME}_shared)
endif()
if(ZSTD_BUILD_STATIC)
    add_library(${PROJECT_NAME}_static STATIC)
    add_library(${PROJECT_NAME}::${PROJECT_NAME}_static ALIAS ${PROJECT_NAME}_static)
    list(APPEND __targets ${PROJECT_NAME}_static)
endif()
if(ZSTD_BUILD_SHARED AND NOT ZSTD_BUILD_STATIC)
    add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME}_shared)
endif()
if(ZSTD_BUILD_STATIC AND NOT ZSTD_BUILD_SHARED)
    add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME}_static)
endif()

if(NOT MSVC)
    add_library(${PROJECT_NAME}_asm STATIC)
    target_sources(${PROJECT_NAME}_asm PRIVATE
        ./zstd/decompress/huf_decompress_amd64.S
    )
endif()

include(GNUInstallDirs)
file(GLOB_RECURSE __sources LIST_DIRECTORIES false CONFIGURE_DEPENDS
    "./zstd/*.h" "./zstd/*.c"
)
foreach(__target IN LISTS __targets)
    target_sources(${__target} PRIVATE ${__sources})
    target_include_directories(${__target} PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}
    )
    target_compile_definitions(${__target} PRIVATE
        XXH_NAMESPACE=ZSTD_ # Always hide XXHash symbols
        ZSTD_LEGACY_SUPPORT=0
        ZSTD_MULTITHREAD
    )
    if(MSVC)
        target_compile_definitions(${__target} PRIVATE
            ZSTD_DISABLE_ASM
            ZSTD_HEAPMODE=0
        )
    endif()
    if(__top_level)
        if(WIN32)
            target_compile_definitions(${__target} PRIVATE
                UNICODE _UNICODE
                NOMINMAX
               _USE_MATH_DEFINES
                WIN32_LEAN_AND_MEAN
            )
        endif()
        if(MSVC)
            target_compile_options(${__target} PRIVATE
                /utf-8 /Zc:__cplusplus
                $<$<NOT:$<CONFIG:Debug>>:/Gw /Gy /Zc:inline>
            )
            target_link_options(${__target} PRIVATE
                $<$<NOT:$<CONFIG:Debug>>:/OPT:REF /OPT:ICF>
            )
        else()
            target_compile_options(${__target} PRIVATE
                $<$<NOT:$<CONFIG:Debug>>:-ffunction-sections -fdata-sections>
            )
            target_link_options(${__target} PRIVATE
                $<$<NOT:$<CONFIG:Debug>>:-Wl,--gc-sections>
            )
        endif()
    endif()
    set_target_properties(${__target} PROPERTIES
        VERSION ${PROJECT_VERSION}
    )
    set(__dep_visibility PRIVATE)
    get_target_property(__type ${__target} TYPE)
    if("x${__type}" STREQUAL "xSTATIC_LIBRARY")
        set(__dep_visibility PUBLIC)
    else()
        set_target_properties(${__target} PROPERTIES
            SOVERSION ${PROJECT_VERSION_MAJOR}
        )
    endif()
    if(NOT MSVC)
        target_link_libraries(${__target} ${__dep_visibility}
            ${PROJECT_NAME}_asm
        )
    endif()
    if(ZSTD_INSTALL)
        install(TARGETS ${__target}
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        )
        install(
            FILES zstd/zstd.h zstd/zdict.h zstd/zstd_errors.h
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
        )
    endif()
endforeach()
