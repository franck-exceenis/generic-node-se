#  Copyright © 2020 The Things Industries B.V.
#  Licensed under the Apache License, Version 2.0 (the "License");
#  you may not use this file except in compliance with the License.
#  You may obtain a copy of the License at
#      http://www.apache.org/licenses/LICENSE-2.0
#  Unless required by applicable law or agreed to in writing, software
#  distributed under the License is distributed on an "AS IS" BASIS,
#  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#  See the License for the specific language governing permissions and
#  limitations under the License.

#-------------------
# Cmake Setup
#-------------------
cmake_minimum_required(VERSION 3.11.1)

if(NOT CMAKE_TOOLCHAIN_FILE)
    message(FATAL_ERROR "[ERRR] CMAKE_TOOLCHAIN_FILE not specified")
endif()

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Debug CACHE STRING "Choose the type of build." FORCE)
    message(WARNING "[WARN] CMAKE_BUILD_TYPE not specified: Using ${CMAKE_BUILD_TYPE} by default")
endif()

if(NOT TARGET_GROUP)
    set(TARGET_GROUP production CACHE STRING "Choose the target group." FORCE)
    message(WARNING "[WARN] TARGET_GROUP not specified: Using ${TARGET_GROUP} by default")
endif()

set(CMAKE_TRY_COMPILE_TARGET_TYPE "STATIC_LIBRARY")
#-------------------
# Project Setup
#-------------------
enable_language(C ASM)
project(main)
#-------------------
# MCU Setup
#-------------------
set(MCU STM32WLE5xx)
set(MCU_DIR target)
set(MCU_SPEC  cortex-m4)
set(STARTUP_FILE ${MCU_DIR}/startup_stm32wle5xx.s)
set(SYSTEM_FILE  ${MCU_DIR}/system_stm32wlxx.c)
set(LINKER_SCRIPT ${MCU_DIR}/stm32wle5xx_flash.ld)
#-------------------
# HAL Setup
#-------------------
set(HAL_DIR target/STM32WLxx_HAL_Driver)
#-------------------
# CMSIS Setup
#-------------------
# Set the path to the CMSIS folder
set(CMSIS_DIR target/STM32WLxx_HAL_Driver/CMSIS)
#-------------------
# General Flags
#-------------------
set(OBJECT_GEN_FLAGS " \
-fno-builtin \
-Wall \
-ffunction-sections -fdata-sections \
-fomit-frame-pointer \
" CACHE INTERNAL "Common flags for C/CXX/ASM/Linker")
#-------------------
# CFLAGS
#-------------------
set(CMAKE_C_FLAGS " \
" CACHE INTERNAL "C Compiler options")
#-------------------
# ASMFLAGS for cross
#-------------------
set(CMAKE_ASM_FLAGS " \
-x assembler-with-cpp \
" CACHE INTERNAL "ASM Compiler options")
#-------------------
# LFLAGS for cross
#-------------------
set(CMAKE_EXE_LINKER_FLAGS " \
-Wl,-Map=${PROJECT_NAME}.map \
-Wl,--print-memory-usage \
-Wl,--gc-sections \
" CACHE INTERNAL "Linker options")
#------------------
# Debug Flags
#------------------
set(CMAKE_C_FLAGS_DEBUG "-Og -g -gdwarf-3 -gstrict-dwarf " CACHE INTERNAL "C Compiler options for debug build type")
set(CMAKE_CXX_FLAGS_DEBUG "-Og -g -gdwarf-3 -gstrict-dwarf " CACHE INTERNAL "C++ Compiler options for debug build type")
set(CMAKE_ASM_FLAGS_DEBUG "-Og -g -gdwarf-3 -gstrict-dwarf " CACHE INTERNAL "ASM Compiler options for debug build type")
set(CMAKE_EXE_LINKER_FLAGS_DEBUG "" CACHE INTERNAL "Linker options for debug build type")
#------------------
# Release Flags
#-----------------
set(CMAKE_C_FLAGS_RELEASE "-Os -flto " CACHE INTERNAL "C Compiler options for release build type")
set(CMAKE_CXX_FLAGS_RELEASE "-Os -flto " CACHE INTERNAL "C++ Compiler options for release build type")
set(CMAKE_ASM_FLAGS_RELEASE "" CACHE INTERNAL "ASM Compiler options for release build type")
set(CMAKE_EXE_LINKER_FLAGS_RELEASE "-flto " CACHE INTERNAL "Linker options for release build type")
#------------------
# Cross Compilation Flags
#-----------------
if(CMAKE_CROSSCOMPILING STREQUAL "1")
    message(STATUS "[INFO] Cross compiling for ${MCU}")
    message(STATUS "[INFO] Startup file used is ${STARTUP_FILE}")
    # Control ARM Semihosting support
    if(NOT SEMIHOSTING)
        set(SEMIHOSTING 1)
        message(WARNING "[WARN] Semihosting support not specified: Enabling by default")
    endif()
    #-------------------
    # General Flags for cross
    #-------------------
    string(APPEND OBJECT_GEN_FLAGS " \
    -mcpu=${MCU_SPEC} \
    -mthumb \
    -mthumb-interwork \
    -mabi=aapcs \
    ${FLOAT_SPEC} \
    ")
    #-------------------
    # CFLAGS for cross
    #-------------------
    string(APPEND CMAKE_C_FLAGS " \
    ${OBJECT_GEN_FLAGS} \
    ")
    #-------------------
    # ASMFLAGS for cross
    #-------------------
    string(APPEND CMAKE_ASM_FLAGS " \
    ${OBJECT_GEN_FLAGS} \
    ")
    #-------------------
    # LFLAGS for cross
    #-------------------
    string(APPEND CMAKE_EXE_LINKER_FLAGS " \
    ${OBJECT_GEN_FLAGS} \
    ")
    find_file(LINKER_SCRIPT_PATH
        NAMES "${LINKER_SCRIPT}"
        PATHS
        ${CMAKE_CURRENT_LIST_DIR}
        )
    if(DEFINED LINKER_SCRIPT_PATH)
        message(STATUS "[INFO] Using linker file at ${LINKER_SCRIPT_PATH}")
        string(APPEND CMAKE_EXE_LINKER_FLAGS "-T${LINKER_SCRIPT_PATH} ")
    else()
        message(FATAL_ERROR "[ERRR] Could not find linker script ${LINKER_SCRIPT}")
    endif()

    if("${SEMIHOSTING}" STREQUAL "1")
        string(APPEND CMAKE_EXE_LINKER_FLAGS "--specs=rdimon.specs -lc -lrdimon ")
    else()
        string(APPEND CMAKE_EXE_LINKER_FLAGS "--specs=nosys.specs ")
    endif()
#------------------
# Native Compilation Flags
#-----------------
else()
    # Flags and options to set while compiling natively
    message(STATUS "[INFO] Compiling natively")
    #-------------------
    # General Flags for native
    #-------------------
    string(APPEND OBJECT_GEN_FLAGS " \
    --coverage \
    ")
    #-------------------
    # CFLAGS for native
    #-------------------
    string(APPEND CMAKE_C_FLAGS " \
    ${OBJECT_GEN_FLAGS} \
    ")
    #-------------------
    # ASMFLAGS for native
    #-------------------
    string(APPEND CMAKE_ASM_FLAGS " \
    ${OBJECT_GEN_FLAGS} \
    ")
    #-------------------
    # LFLAGS for native
    #-------------------
    string(APPEND CMAKE_EXE_LINKER_FLAGS " \
    ${OBJECT_GEN_FLAGS} \
    ")
endif()
# Actual build which will go onto the board
if(TARGET_GROUP STREQUAL production)
    #-------------------
    # HAL library
    #-------------------
    file(GLOB HAL_SRC
        "${HAL_DIR}/Src/*.c"
        )
    add_library(hal STATIC
        ${HAL_SRC}
        )
    target_include_directories(hal
        PUBLIC
        ${HAL_DIR}/Inc
        ${MCU_DIR}/Include
        ${CMSIS_DIR}/Core/Include
        PRIVATE
        ${PROJECT_SOURCE_DIR}/target
        )
    target_compile_definitions(hal
        PUBLIC
        ${MCU}
        )
    #-------------------
    # Main elf
    #-------------------
    file(GLOB MAIN_SRC
        "app/basic/*.c"
        "target/*.c"
        "lib/STNODE_BSP/*.c"
        "lib/Utilities/*.c"
        "lib/SHTC3/*.c"
        )
    set(SOURCES
        ${MAIN_SRC}
        ${STARTUP_FILE}
        ${SYSTEM_FILE}
        )
    add_executable(${PROJECT_NAME}.elf
        ${SOURCES}
        )
    target_include_directories(${PROJECT_NAME}.elf
        PUBLIC
        ${PROJECT_SOURCE_DIR}/target
        ${PROJECT_SOURCE_DIR}/app
        ${PROJECT_SOURCE_DIR}/app/basic
        ${PROJECT_SOURCE_DIR}/lib/STNODE_BSP
        ${PROJECT_SOURCE_DIR}/lib/Utilities
        ${PROJECT_SOURCE_DIR}/lib/SHTC3
        )
    target_link_libraries(${PROJECT_NAME}.elf
        PUBLIC
        hal
        )
    target_compile_definitions(${PROJECT_NAME}.elf
        PUBLIC
        ${MCU}
        SEMIHOSTING=${SEMIHOSTING}
        )
        # Create output in hex and binary format
        create_bin_output(${PROJECT_NAME})
        create_hex_output(${PROJECT_NAME})
        # Add additional files to the make clean
        set_property(DIRECTORY PROPERTY ADDITIONAL_MAKE_CLEAN_FILES
        "${PROJECT_NAME}.map"
        "${PROJECT_NAME}.bin"
        "${PROJECT_NAME}.hex"
    )
else()
    message(FATAL_ERROR "Given TARGET_GROUP unknown")
endif()

# Unset all cache
unset(SEMIHOSTING)
unset(CMAKE_TOOLCHAIN_FILE)
unset(CMAKE_BUILD_TYPE)
unset(TARGET_GROUP)