cmake_minimum_required(VERSION 3.0)

include(GNUInstallDirs)
#include(CTest)

project(
    LZ4 
    VERSION 1.6.0
    LANGUAGES C
)

if (MSVC)
    if (NOT DEFINED ARCH)
        string(TOLOWER ${MSVC_C_ARCHITECTURE_ID} ARCH)
    endif()

    # use boost convention - you cant' get the VC compiler without VS
    if (NOT DEFINED MSVCVER)
        math(EXPR MSVCVER "(${MSVC_VERSION} / 10) - 60")
    endif()
    
    # add the runtime version and target architecture into the DLL name per MS
    # guidelines
    set(MSVC_SHARED_SUFFIX_BASE "-${PROJECT_VERSION_MAJOR}-vc${MSVCVER}-mt-${ARCH}")
    set(CMAKE_SHARED_LIBRARY_SUFFIX "${MSVC_SHARED_SUFFIX_BASE}.dll")

    set(VER_PRODUCTNAME_STR "LZ4 compression library <https://code.google.com/p/lz4/>")
    set(VER_LEGALCOPYRIGHT_STR "Copyright (C) 2011-2015, Yann Collet")
    set(VER_STR "${PROJECT_VERSION}")
    set(VER_NUM "${PROJECT_VERSION_MAJOR},${PROJECT_VERSION_MINOR},${PROJECT_VERSION_PATCH},0")  
endif()

function(add_win32_rc target)
    get_target_property(VER_INTERNALNAME_STR ${target} OUTPUT_NAME)

    if (VER_INTERNALNAME_STR MATCHES "VER_INTERNALNAME_STR-NOTFOUND")
        set(VER_INTERNALNAME_STR "${target}")
    endif()
    get_target_property(VER_FILEDESCRIPTION_STR ${target} DESCRIPTION)

    get_target_property(type ${target} TYPE)
    if (${type} STREQUAL "EXECUTABLE")
        set(VER_FILETYPE "VFT_APP")
        set(VER_ORIGINALFILENAME_STR "${VER_INTERNALNAME_STR}${CMAKE_EXECUTABLE_SUFFIX}")
    elseif (${type} STREQUAL "SHARED_LIBRARY")
        set(VER_FILETYPE "VFT_DLL")
        set(VER_ORIGINALFILENAME_STR "${CMAKE_SHARED_LIBRARY_PREFIX}${VER_INTERNALNAME_STR}${CMAKE_SHARED_LIBRARY_SUFFIX}")
    endif()

    set(OUT_FNAME "${CMAKE_CURRENT_BINARY_DIR}/${VER_INTERNALNAME_STR}.rc")
    configure_file(
         ${CMAKE_SOURCE_DIR}/version.rc.cmakein
         ${OUT_FNAME}
         @ONLY
     )         

    target_sources(${target} PRIVATE ${OUT_FNAME})

endfunction(add_win32_rc)

set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "LZ4 compression library")
set(CPACK_PACKAGE_VERSION_MAJOR 1)
set(CPACK_PACKAGE_VERSION_MINOR 6)
set(CPACK_PACKAGE_VERSION_PATCH r128)
set(VERSION_STRING	" \"${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}\" ")
include(CPack)

include (CheckTypeSize)
check_type_size("void *" SIZEOF_VOID_P)
IF( ${SIZEOF_VOID_P} STREQUAL  "8" )
    set (CMAKE_SYSTEM_PROCESSOR "64bit")
    MESSAGE( STATUS "64 bit architecture detected size of void * is " ${SIZEOF_VOID_P})
ENDIF()

option(BUILD_TOOLS "Build the command line tools" ON)

if(UNIX)
    add_definitions(-fPIC)
endif()

set(LZ4_DIR ../lib/)
set(PRG_DIR ../programs/)
set(LZ4_SRCS_LIB ${LZ4_DIR}lz4.c ${LZ4_DIR}lz4hc.c ${LZ4_DIR}lz4.h ${LZ4_DIR}lz4hc.h ${LZ4_DIR}lz4frame.c ${LZ4_DIR}xxhash.c)
set(LZ4_SRCS ${LZ4_DIR}lz4frame.c ${LZ4_DIR}xxhash.c ${PRG_DIR}bench.c ${PRG_DIR}lz4cli.c ${PRG_DIR}lz4io.c)

ADD_DEFINITIONS("-DLZ4_VERSION=\"${CPACK_PACKAGE_VERSION_PATCH}\"")
INCLUDE_DIRECTORIES (${LZ4_DIR})


# liblz4 shared
add_library(liblz4 SHARED ${LZ4_SRCS_LIB})
set_target_properties(liblz4 PROPERTIES
    OUTPUT_NAME lz4
    SOVERSION "${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}"
)

if (WIN32)
    set_property(TARGET liblz4 PROPERTY DESCRIPTION "lz4 compression dynamic link library")
    target_sources(liblz4 PRIVATE liblz4.def)
    add_win32_rc(liblz4)
endif()


# liblz4 static
add_library(lz4_static STATIC ${LZ4_SRCS_LIB})

install(TARGETS liblz4 lz4_static
    RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
    ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
)

# headers
install(
    FILES
        ${LZ4_DIR}/lz4.h
        ${LZ4_DIR}/lz4hc.h
    DESTINATION include
)

if(BUILD_TOOLS)
    add_executable(lz4 ${LZ4_SRCS})
    install(TARGETS lz4 RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}")
    target_link_libraries(lz4 liblz4)
    if (WIN32)
        set_property(TARGET lz4 PROPERTY DESCRIPTION "LZ4 command line interface")
        add_win32_rc(lz4)
    endif()

endif()


#warnings

if(CMAKE_COMPILER_IS_GNUCXX)
    ADD_DEFINITIONS("-Wall")
    ADD_DEFINITIONS("-Wextra")
    ADD_DEFINITIONS("-Wundef")
    ADD_DEFINITIONS("-Wshadow")
    ADD_DEFINITIONS("-Wcast-align")
    ADD_DEFINITIONS("-Wstrict-prototypes")
    ADD_DEFINITIONS("-std=c99")
endif()



