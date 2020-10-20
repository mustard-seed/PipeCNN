cmake_minimum_required (VERSION 2.8)
set (PROJECT_NAME "pipeCNN")

project (${PROJECT_NAME})
include (${CMAKE_CURRENT_SOURCE_DIR}/config.cmake)

############################################
#First, we must be able to find Intel FPGA OpenCL SDK
#############################################
if(DEFINED ENV{INTELFPGAOCLSDKROOT})
else()
    message(FATAL_ERROR "The environment variable INTELFPGAOCLSDKROOT isn't set.")
endif()

set (PREPROCESSOR_DEFS 
    -DDUMMY
)

#########################
#Obtain the host source files
#########################
file(GLOB HOST_SRC
        ${CMAKE_CURRENT_SOURCE_DIR}/project/host/*.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/common/*.cpp
)

###########################
#Add the include files
###########################
include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/common
    ${CMAKE_CURRENT_SOURCE_DIR}/project/host
)

include_directories(
    $ENV{INTELFPGAOCLSDKROOT}/host/include
)

##################################
#Add the RTL directories and libraries
##################################
set (rtl_directory ${CMAKE_CURRENT_SOURCE_DIR}/project/device/RTL
)

set (rtl_lib -l rtl_lib.aoclib)
INCLUDE_DIRECTORIES (${rtl_directory})
###########################################################
#Intel FPGA OpenCL SDK Compile and linkage configurations
###########################################################
if (BOARD_NAME MATCHES "DE10Standard")
    ##Add library directories for DE10Standard
    message (STATUS "Linking against DE10-Standard libraries")
    message (STATUS "$ENV{INTELFPGAOCLSDKROOT}/host/arm32/lib")
    message (STATUS "$ENV{INTELFPGAOCLSDKROOT}/board/de10_standard/arm32/lib")
    file (GLOB intelfpga_host_libraries
        $ENV{INTELFPGAOCLSDKROOT}/host/arm32/lib/*.so
        )
    file (GLOB intelfpga_device_libraries
        $ENV{INTELFPGAOCLSDKROOT}/board/de10_standard/arm32/lib/*.so
        )
    add_definitions(-DC5SOC)
    list (APPEND PREPROCESSOR_DEFS -DC5SOC)
elseif (BOARD_NAME MATCHES "A10REF")
    ##Add library directories for a10_ref
    add_definitions(-DARRIA10)
    list (APPEND PREPROCESSOR_DEFS -DARRIA10)
    message (STATUS "Linking against A10-Ref libraries")
elseif (BOARD_NAME MATCHES "A10PAC")
    add_definitions(-DARRIA10 -DA10PAC)
    ##Add library directories for a10_pac
    list (APPEND PREPROCESSOR_DEFS -DARRIA10)
    message (STATUS "Linking against A10-Pac libraries")
elseif (BOARD_NAME MATCHES "DE5ANET")
    add_definitions(-DARRIA10)
    ##Add library directories for de5a_net
    list (APPEND PREPROCESSOR_DEFS -DARRIA10)
    message (STATUS "Linking against DE5A-NET libraries")
else()
    message (STATUS "The board is ${BOARD_NAME}")
    message (FATAL_ERROR "Unsupported board!")
endif()

#######################################
##Add extra library for the host below
######################################
set (OpenCV_DIR /home/jamesliu/arm_opencv/Build CACHE STRING "Path of OpenCV")
find_package (OpenCV REQUIRED )

include_directories(${OpenCV_INCLUDE_DIRS})
add_definitions(-DUSE_OPENCV)
#####################################
##Add unit test for the host library below
#####################################
add_definitions(-DFPGA_DEVICE)
add_executable (${PROJECT_NAME}_host ${HOST_SRC})

if ("${intelfpga_host_libraries}" STREQUAL "")
    #Hack for non arm boards
    execute_process(
            COMMAND aocl link-config
            OUTPUT_VARIABLE aocl_link_config
        )
    #Need to remove the newline, or the make file will complain "missing separator"
    string(REGEX REPLACE "\n$" "" aocl_link_config "${aocl_link_config}")
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${aocl_link_config}")
else()
    target_link_libraries(${PROJECT_NAME}_host
         ${intelfpga_host_libraries}
         ${intelfpga_device_libraries}
         )
endif()

target_link_libraries(${PROJECT_NAME}_host
         rt
         pthread
        ${OpenCV_LIBS}
    )
set_target_properties (${PROJECT_NAME}_host PROPERTIES "CMAKE_CXX_FLAGS" "${CMAKE_CXX_FLAGS}")


####################################
##Generate the hardware targets
####################################

include (aoc.cmake)

# Find the kernel source files. Note, GLOB isn't the recommended way. One needs to touch the CMakesList.txt to effect changes
#file(GLOB_RECURSE kernel_files
#     ${CMAKE_CURRENT_SOURCE_DIR}/project/device/*.cl
#)
list(APPEND kernel_files ${CMAKE_CURRENT_SOURCE_DIR}/project/device/conv_pipe.cl)

add_aoc_target (
    TARGET_NAME ${PROJECT_NAME} 
    TARGET_TYPE    EMULATION
    SOURCES_LIST ${kernel_files} 
    HEADER_DIR ${CMAKE_CURRENT_SOURCE_DIR}/host 
    RTL_DIR ${rtl_directory} 
    PREPROCESSOR_DEFS_LIST ${PREPROCESSOR_DEFS}
    RTL_LIB ${rtl_lib}
    BOARD_NAME ${BOARD_NAME}
    )

add_aoc_target (
    TARGET_NAME ${PROJECT_NAME} 
    TARGET_TYPE    RTL_ONLY
    SOURCES_LIST ${kernel_files} 
    HEADER_DIR ${CMAKE_CURRENT_SOURCE_DIR}/host 
    RTL_DIR ${rtl_directory} 
    PREPROCESSOR_DEFS_LIST ${PREPROCESSOR_DEFS}
    RTL_LIB ${rtl_lib}
    BOARD_NAME ${BOARD_NAME}
    )

add_aoc_target (
    TARGET_NAME ${PROJECT_NAME} 
    TARGET_TYPE    NORMAL_HW
    SOURCES_LIST ${kernel_files} 
    HEADER_DIR ${CMAKE_CURRENT_SOURCE_DIR}/host 
    RTL_DIR ${rtl_directory} 
    PREPROCESSOR_DEFS_LIST ${PREPROCESSOR_DEFS}
    RTL_LIB ${rtl_lib}
    BOARD_NAME ${BOARD_NAME}
    )

add_aoc_target (
    TARGET_NAME ${PROJECT_NAME} 
    TARGET_TYPE    RELEASE_HW
    SOURCES_LIST ${kernel_files} 
    HEADER_DIR ${CMAKE_CURRENT_SOURCE_DIR}/host 
    RTL_DIR ${rtl_directory} 
    PREPROCESSOR_DEFS_LIST ${PREPROCESSOR_DEFS}
    RTL_LIB ${rtl_lib}
    BOARD_NAME ${BOARD_NAME}
    )

add_aoc_target (
    TARGET_NAME ${PROJECT_NAME} 
    TARGET_TYPE    PROFILE_HW
    SOURCES_LIST ${kernel_files} 
    HEADER_DIR ${CMAKE_CURRENT_SOURCE_DIR}/host 
    RTL_DIR ${rtl_directory} 
    PREPROCESSOR_DEFS_LIST ${PREPROCESSOR_DEFS}
    RTL_LIB ${rtl_lib}
    BOARD_NAME ${BOARD_NAME}
    )





