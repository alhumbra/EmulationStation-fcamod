cmake_minimum_required(VERSION 2.8)

option(GLES "Set to ON if targeting Embedded OpenGL" ${GLES})
option(GL "Set to ON if targeting Desktop OpenGL" ${GL})
option(RPI "Set to ON to enable the Raspberry PI video player (omxplayer)" ${RPI})
option(CEC "Set to ON to enable CEC" ${CEC})

project(emulationstation-all)

#-------------------------------------------------------------------------------
#add local find scripts to CMAKE path
LIST(APPEND CMAKE_MODULE_PATH
    ${CMAKE_CURRENT_SOURCE_DIR}/CMake/Utils
    ${CMAKE_CURRENT_SOURCE_DIR}/CMake/Packages
)

#-------------------------------------------------------------------------------
#set up OpenGL system variable
if(GLES)
    set(GLSystem "Embedded OpenGL" CACHE STRING "The OpenGL system to be used")
elseif(GL)
    set(GLSystem "Desktop OpenGL" CACHE STRING "The OpenGL system to be used")
#-------------------------------------------------------------------------------
#check if we're running on Raspberry Pi
elseif(EXISTS "${CMAKE_FIND_ROOT_PATH}/opt/vc/include/bcm_host.h")
    MESSAGE("bcm_host.h found")
    set(BCMHOST found)
    set(GLSystem "Embedded OpenGL" CACHE STRING "The OpenGL system to be used")
#-------------------------------------------------------------------------------
#check if we're running on OSMC Vero4K
elseif(EXISTS "${CMAKE_FIND_ROOT_PATH}/opt/vero3/lib/libMali.so")
    MESSAGE("libMali.so found")
    set(VERO4K found)
    set(GLSystem "Embedded OpenGL" CACHE STRING "The OpenGL system to be used")
#-------------------------------------------------------------------------------
#check if we're running on olinuxino / odroid / etc
elseif(EXISTS "${CMAKE_FIND_ROOT_PATH}/usr/lib/libMali.so" OR
    EXISTS "${CMAKE_FIND_ROOT_PATH}/usr/lib/arm-linux-gnueabihf/libMali.so" OR
    EXISTS "${CMAKE_FIND_ROOT_PATH}/usr/lib/aarch64-linux-gnu/libMali.so" OR
    EXISTS "${CMAKE_FIND_ROOT_PATH}/usr/local/lib/aarch64-linux-gnu/libmali.so" OR
    EXISTS "${CMAKE_FIND_ROOT_PATH}/usr/lib/arm-linux-gnueabihf/mali-egl/libmali.so" OR
    EXISTS "${CMAKE_FIND_ROOT_PATH}/usr/lib/arm-linux-gnueabihf/libmali.so")
    MESSAGE("libMali.so found")
    set(GLSystem "Embedded OpenGL" CACHE STRING "The OpenGL system to be used")
else()
    set(GLSystem "Desktop OpenGL" CACHE STRING "The OpenGL system to be used")
endif(GLES)

set_property(CACHE GLSystem PROPERTY STRINGS "Desktop OpenGL" "Embedded OpenGL")

#finding necessary packages
#-------------------------------------------------------------------------------
if(${GLSystem} MATCHES "Desktop OpenGL")
    find_package(OpenGL REQUIRED)
else()
    find_package(OpenGLES REQUIRED)
endif()
find_package(Freetype REQUIRED)
find_package(FreeImage REQUIRED)
find_package(SDL2 REQUIRED)
find_package(SDL2MIXER REQUIRED)
find_package(CURL REQUIRED)
find_package(VLC REQUIRED)
find_package(RapidJSON REQUIRED)

#add libCEC support
if(CEC)
    find_package(libCEC REQUIRED)
endif()

# i18n
# if(MSVC)
# 	find_package(Intl REQUIRED)
# endif()

#add ALSA for Linux
if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    find_package(ALSA REQUIRED)
endif()

#-------------------------------------------------------------------------------
#set up compiler flags and excutable names
if(DEFINED BCMHOST OR RPI)
    add_definitions(-D_RPI_)
endif()

if(DEFINED VERO4K)
    add_definitions(-D_VERO4K_)
endif()

if(DEFINED libCEC_FOUND)
    add_definitions(-DHAVE_LIBCEC)
endif()

#-------------------------------------------------------------------------------

if(MSVC)
    set(CMAKE_DEBUG_POSTFIX "d")
    add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
    add_definitions(-D_CRT_NONSTDC_NO_DEPRECATE)
    add_definitions(-DNOMINMAX)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP") #multi-processor compilation
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /MP") #multi-processor compilation
endif()

if(CMAKE_COMPILER_IS_GNUCXX)
    #check for G++ 4.7+
    execute_process(COMMAND ${CMAKE_CXX_COMPILER} -dumpversion OUTPUT_VARIABLE G++_VERSION)
    if (G++_VERSION VERSION_LESS 4.7)
            message(SEND_ERROR "You need at least G++ 4.7 to compile EmulationStation!")
    endif()

    #set up compiler flags for GCC
if (CMAKE_BUILD_TYPE MATCHES Debug)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wno-attributes -O0") #support C++11 for std::, optimize
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -O0")
else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wno-attributes -O2") #support C++11 for std::, optimize
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -O2")  #-s = strip binary
endif()
endif()

if(${GLSystem} MATCHES "Desktop OpenGL")
    add_definitions(-DUSE_OPENGL_21)
else()
    add_definitions(-DUSE_OPENGLES_10)
endif()

#-------------------------------------------------------------------------------
#add include directories
set(COMMON_INCLUDE_DIRS
    ${FREETYPE_INCLUDE_DIRS}
    ${FreeImage_INCLUDE_DIRS}
    ${SDL2_INCLUDE_DIR}
	${SDLMIXER_INCLUDE_DIR}
    ${CURL_INCLUDE_DIR}
    ${VLC_INCLUDE_DIR}
    ${RAPIDJSON_INCLUDE_DIRS}
    ${CMAKE_CURRENT_SOURCE_DIR}/external
    ${CMAKE_CURRENT_SOURCE_DIR}/es-core/src
)

if(DEFINED SCREENSCRAPER_SOFTNAME)
    add_definitions(-DSCREENSCRAPER_SOFTNAME="${SCREENSCRAPER_SOFTNAME}")
endif()

if(DEFINED SCREENSCRAPER_DEV_LOGIN)
    add_definitions(-DSCREENSCRAPER_DEV_LOGIN="${SCREENSCRAPER_DEV_LOGIN}")
endif()

if(DEFINED GAMESDB_APIKEY)
    add_definitions(-DGAMESDB_APIKEY="${GAMESDB_APIKEY}")
endif()

# if(MSVC)
#         LIST(APPEND COMMON_INCLUDE_DIRS
#             ${Intl_INCLUDE_DIRS}
#         )
# endif()

#add libCEC_INCLUDE_DIR
if(DEFINED libCEC_FOUND)
    LIST(APPEND COMMON_INCLUDE_DIRS
        ${libCEC_INCLUDE_DIR}
    )
endif()

#add ALSA for Linux
if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    LIST(APPEND COMMON_INCLUDE_DIRS
        ${ALSA_INCLUDE_DIRS}
    )
endif()

if(DEFINED BCMHOST)
    LIST(APPEND COMMON_INCLUDE_DIRS
        "${CMAKE_FIND_ROOT_PATH}/opt/vc/include"
        "${CMAKE_FIND_ROOT_PATH}/opt/vc/include/interface/vcos"
        "${CMAKE_FIND_ROOT_PATH}/opt/vc/include/interface/vmcs_host/linux"
        "${CMAKE_FIND_ROOT_PATH}/opt/vc/include/interface/vcos/pthreads"
    )
#add include directory for Vero4K
elseif(DEFINED VERO4K)
    LIST(APPEND COMMON_INCLUDE_DIRS
        "${CMAKE_FIND_ROOT_PATH}/opt/vero3/include"
    )
else()
    if(${GLSystem} MATCHES "Desktop OpenGL")
        LIST(APPEND COMMON_INCLUDE_DIRS
            ${OPENGL_INCLUDE_DIR}
        )
    else()
        LIST(APPEND COMMON_INCLUDE_DIRS
            ${OPENGLES_INCLUDE_DIR}
        )
    endif()
endif()

#-------------------------------------------------------------------------------
#define libraries and directories
if(DEFINED BCMHOST)
    link_directories(
        "${CMAKE_FIND_ROOT_PATH}/opt/vc/lib"
    )
elseif(DEFINED VERO4K)
    link_directories(
        "${CMAKE_FIND_ROOT_PATH}/opt/vero3/lib"
    )
endif()

set(COMMON_LIBRARIES
    ${FREETYPE_LIBRARIES}
    ${FreeImage_LIBRARIES}	
    ${SDL2_LIBRARY}
	${SDLMIXER_LIBRARY}
    ${CURL_LIBRARIES}
    ${VLC_LIBRARIES}
    pugixml
    nanosvg
    go2
)

# if(MSVC)
#         LIST(APPEND COMMON_LIBRARIES
#             ${Intl_LIBRARIES}
#         )
# endif()

#add libCEC_LIBRARIES
if(DEFINED libCEC_FOUND)
if(DEFINED BCMHOST)
    LIST(APPEND COMMON_LIBRARIES
        vchiq_arm
    )
endif()
    LIST(APPEND COMMON_LIBRARIES
        dl
        ${libCEC_LIBRARIES}
    )
endif()

#add ALSA for Linux
if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    LIST(APPEND COMMON_LIBRARIES
        ${ALSA_LIBRARY}
    )
endif()

if(DEFINED BCMHOST)
    LIST(APPEND COMMON_LIBRARIES
        bcm_host
        brcmEGL
        ${OPENGLES_LIBRARIES}
    )
elseif(DEFINED VERO4K)
    LIST(APPEND COMMON_LIBRARIES
        EGL
        ${OPENGLES_LIBRARIES}
    )
else()
    if(MSVC)
        LIST(APPEND COMMON_LIBRARIES
            winmm
        )
    endif()
    if(${GLSystem} MATCHES "Desktop OpenGL")
        LIST(APPEND COMMON_LIBRARIES
            ${OPENGL_LIBRARIES}
        )
    else()
        LIST(APPEND COMMON_LIBRARIES
            EGL
            ${OPENGLES_LIBRARIES}
        )
    endif()
endif()

#-------------------------------------------------------------------------------
# set up build directories
set(dir ${CMAKE_CURRENT_SOURCE_DIR})
set(EXECUTABLE_OUTPUT_PATH ${dir} CACHE PATH "Build directory" FORCE)
set(LIBRARY_OUTPUT_PATH ${dir} CACHE PATH "Build directory" FORCE)


#-------------------------------------------------------------------------------
# add each component

add_subdirectory("external")
add_subdirectory("es-core")
add_subdirectory("es-app")
