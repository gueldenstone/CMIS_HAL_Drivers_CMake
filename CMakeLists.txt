cmake_minimum_required(VERSION 3.17)
set(CMAKE_TRY_COMPILE_TARGET_TYPE "STATIC_LIBRARY")
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_LIST_DIR}/cmake)
project(STM32Cube
    LANGUAGES C ASM
)
if(NOT TARGET_NAME)
    message(STATUS "No Target defined in Parent Project, switching to default STM32F303CC")
    set(STM32_CHIP "STMF303CC")
    set(STM32_FAMILY "F3")
    set(STM32_TYPE "304xx")
    set(STM32_SIZE_CODE "C")
    set(STM32_ARM_CORE "core_cm4")
    set(USE_CMSIS TRUE)
    # set(UPDATE_CMSIS TRUE)
    set(USE_HAL_DRIVER TRUE)
    # set(UPDATE_HAL TRUE)

else()
    get_target_property(STM32_CHIP ${TARGET_NAME} STM32_CHIP)
    get_target_property(STM32_FAMILY ${TARGET_NAME} STM32_FAMILY)
    get_target_property(STM32_TYPE ${TARGET_NAME} STM32_TYPE)
    get_target_property(STM32_SIZE_CODE ${TARGET_NAME} STM32_SIZE_CODE)
    get_target_property(STM32_ARM_CORE ${TARGET_NAME} STM32_ARM_CORE)
    get_target_property(STM32_COMPILE_DEFINITIONS ${TARGET_NAME} COMPILE_DEFINITIONS)
endif()

# process name properties
string(REGEX REPLACE "^[sS][tT][mM]32[fFhHlL]([0-9][0-9][0-9]).+$" "\\1" TYPE ${STM32_CHIP})

string(TOLOWER ${STM32_FAMILY} STM32_FAMILY_LOW)
string(TOLOWER ${STM32_TYPE} STM32_TYPE_LOW)
string(TOLOWER ${STM32_SIZE_CODE} STM32_SIZE_CODE_LOW)

include(FetchContent)
# get HAL Drivers from GitHub
if(USE_HAL_DRIVER OR USE_LL_DRIVER)
    FetchContent_Declare(hal
        GIT_REPOSITORY https://github.com/STMicroelectronics/stm32${STM32_FAMILY_LOW}xx_hal_driver.git
        GIT_TAG master
        GIT_PROGRESS FALSE
        BUILD_COMMAND ""
        CONFIGURE_COMMAND ""
        INSTALL_COMMAND ""
        TEST_COMMAND ""
        SOURCE_DIR "${CMAKE_CURRENT_LIST_DIR}/Drivers/STM32${STM32_FAMILY}xx_HAL_Driver")
    
    if(NOT UPDATE_HAL)
        set(FETCHCONTENT_UPDATES_DISCONNECTED TRUE)
    else()
        set(FETCHCONTENT_UPDATES_DISCONNECTED FALSE)
    endif()
    message(STATUS "Downloading HAL Drivers")
    FetchContent_GetProperties(hal)
    FetchContent_Populate(hal)
    message(STATUS "The HAL Drivers have been written to: " ${hal_SOURCE_DIR})
endif()
    
# get CMSIS Drivers from GitHub
if(USE_CMSIS)
    FetchContent_Declare(cmsis_core
        GIT_REPOSITORY https://github.com/STMicroelectronics/cmsis_core.git
        GIT_TAG master
        GIT_PROGRESS FALSE
        BUILD_COMMAND ""
        CONFIGURE_COMMAND ""
        INSTALL_COMMAND ""
        TEST_COMMAND ""
        SOURCE_DIR "${CMAKE_CURRENT_LIST_DIR}/Drivers/CMSIS")
    
    FetchContent_Declare(cmsis_device
        GIT_REPOSITORY https://github.com/STMicroelectronics/cmsis_device_${STM32_FAMILY_LOW}.git
        GIT_TAG master
        GIT_PROGRESS FALSE
        BUILD_COMMAND ""
        CONFIGURE_COMMAND ""
        INSTALL_COMMAND ""
        TEST_COMMAND ""
        SOURCE_DIR "${CMAKE_CURRENT_LIST_DIR}/Drivers/CMSIS/Device/ST/STM32${STM32_FAMILY}xx")
    
    if(NOT UPDATE_CMSIS)
        set(FETCHCONTENT_UPDATES_DISCONNECTED TRUE)
    else()
        set(FETCHCONTENT_UPDATES_DISCONNECTED FALSE)
    endif()
    message(STATUS "Downloading CMSIS Core Drivers")
    FetchContent_GetProperties(cmsis_core)
    FetchContent_Populate(cmsis_core)
    message(STATUS "The CMSIS Core Drivers have been written to: " ${cmsis_core_SOURCE_DIR})
    message(STATUS "Downloading CMSIS Device Drivers")
    FetchContent_GetProperties(cmsis_device)
    FetchContent_Populate(cmsis_device)
    message(STATUS "The CMSIS Device Drivers have been written to: " ${cmsis_device_SOURCE_DIR})
endif()

# Set Common Headers
set(CMSIS_COMMON_HEADERS
    arm_common_tables.h
    arm_const_structs.h
    arm_math.h
    core_cmFunc.s
    core_cmInstr.h
    core_cmSimd.h
)

add_library(CMSIS INTERFACE)
target_sources(CMSIS
    INTERFACE
        # Sources
        ${cmsis_device_SOURCE_DIR}/Source/Templates/system_stm32${STM32_FAMILY_LOW}xx.c
        ${cmsis_device_SOURCE_DIR}/Source/Templates/gcc/startup_stm32f303xe.s
        # Header
        ${cmsis_core_SOURCE_DIR}/Core/Include/${STM32_ARM_CORE}.h
        ${cmsis_device_SOURCE_DIR}/Include/stm32${STM32_FAMILY_LOW}xx.h
        ${cmsis_device_SOURCE_DIR}/Include/system_stm32${STM32_FAMILY_LOW}xx.h
)
target_include_directories(CMSIS
    INTERFACE
        ${cmsis_device_SOURCE_DIR}/Include
        ${cmsis_core_SOURCE_DIR}/Core/Include
)

get_target_property(TARGET_COMP_DEF ${TARGET_NAME} COMPILE_DEFINITIONS)
if(TARGET_COMP_DEF STREQUAL USE_LL_DRIVER OR USE_HAL_DRIVER)
    include(hal_ll_select)
    add_library(HAL_DRIVER INTERFACE)
    select_hal_driver()
    target_sources(HAL_DRIVER
        INTERFACE
            # Sources
            ${HAL_SRCS}
            # Header
            ${HAL_HEADERS}
    )

    target_include_directories(HAL_DRIVER
        INTERFACE
            ${hal_SOURCE_DIR}/Inc
    )
    if(TARGET_NAME)
        get_target_property(TARGET_INCLUDE_DIR ${TARGET_NAME} INCLUDE_DIRECTORIES)
        message(STATUS "THE INCLUDE DIR IS: ${TARGET_INCLUDE_DIR}")
        file(COPY ${hal_SOURCE_DIR}/Inc/stm32f3xx_hal_conf_template.h DESTINATION ${TARGET_INCLUDE_DIR})
        file(RENAME ${TARGET_INCLUDE_DIR}/stm32f3xx_hal_conf_template.h ${TARGET_INCLUDE_DIR}/stm32f3xx_hal_conf.h )
    endif()
    add_library(LL_DRIVER INTERFACE)
    select_ll_driver()
    target_sources(LL_DRIVER
        INTERFACE
            # Sources
            ${LL_SRCS}
            # Header
            ${LL_HEADERS}
    )
    target_include_directories(LL_DRIVER
        INTERFACE
            ${hal_SOURCE_DIR}/Inc
    )
endif()
message(STATUS "THE INCLUDE DIR FOR HAL IS: ${hal_SOURCE_DIR}/Inc")


