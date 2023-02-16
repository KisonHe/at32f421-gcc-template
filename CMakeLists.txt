set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_VERSION 1)
cmake_minimum_required(VERSION 3.20)

# specify cross compilers and tools
set(CMAKE_C_COMPILER arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
set(CMAKE_ASM_COMPILER  arm-none-eabi-gcc)
set(CMAKE_AR arm-none-eabi-ar)
set(CMAKE_OBJCOPY arm-none-eabi-objcopy)
set(CMAKE_OBJDUMP arm-none-eabi-objdump)
set(SIZE arm-none-eabi-size)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

project(at32f421_demo C CXX ASM)
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_C_STANDARD 11)


if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    message("Debug mode")
elseif (CMAKE_BUILD_TYPE STREQUAL "Release")
    message("Release mode")
else ()
    message(FATAL_ERROR "请指定编译类型")
endif ()

add_compile_options(-Wall -Wextra)

#Uncomment for hardware floating point
# add_compile_definitions(ARM_MATH_CM4;ARM_MATH_MATRIX_CHECK;ARM_MATH_ROUNDING)
# add_compile_options(-mfloat-abi=hard -mfpu=fpv4-sp-d16)
# add_link_options(-mfloat-abi=hard -mfpu=fpv4-sp-d16)

#Uncomment for software floating point
add_compile_options(-mfloat-abi=soft)

add_compile_options(-mcpu=cortex-m4 -mthumb -mthumb-interwork)
add_compile_options(-ffunction-sections -fdata-sections -fno-common -fmessage-length=0)


add_compile_options(-DEMBEDDED -DAT32F421F8P7 -DUSE_STDPERIPH_DRIVER)

file(GLOB APP_MAIN "src/main.cpp")

file(GLOB COMMON_SOURCES_Normal
        "lib/core/*.*"
        )
file(GLOB COMMON_SOURCES_SINGLE
        "lib/cmsis/cm4/device_support/system_at32f421.c"
        "lib/drivers/src/at32f421_crm.c"
        "lib/drivers/src/at32f421_misc.c"
        "lib/drivers/src/at32f421_gpio.c"
        "lib/cmsis/cm4/device_support/startup/gcc/startup_at32f421.s"
        )
# https://www.freertos.org/FreeRTOS_Support_Forum_Archive/March_2018/freertos_FreeRTOS_port_for_ARM_M4_no_FPU_no_MPU_25989e7cj.html
# at32f421 has no fpu, If you have a Cortex-M4 without a floating point unit, or a Cortex-M3, then use the port files in FreeRTOS/source/portable/[compiler]/ARM_CM3
add_library(FreeRTOS STATIC
        lib/freertos/source/event_groups.c
        lib/freertos/source/list.c
        lib/freertos/source/queue.c
        lib/freertos/source/stream_buffer.c
        lib/freertos/source/tasks.c
        lib/freertos/source/timers.c
        lib/freertos/source/portable/GCC/ARM_CM3/port.c
        lib/freertos/source/portable/memmang/heap_4.c)

target_include_directories(FreeRTOS PUBLIC
        lib/freertos/config
        lib/freertos/source/portable/GCC/ARM_CM3
        lib/freertos/source/include)


include_directories(
        lib/cmsis/cm4/core_support 
        lib/cmsis/cm4/device_support 
        lib/core 
        lib/drivers/inc
        lib/freertos/source/include
        lib/freertos/config
)

if (CMAKE_BUILD_TYPE STREQUAL "Debug")

elseif (CMAKE_BUILD_TYPE STREQUAL "Release")
    #add_compile_options(-O2)   // in case O3 caused some problem
else ()
    message(FATAL_ERROR "未知编译类型：${CMAKE_BUILD_TYPE};请显式指定编译类型")
endif ()

list(APPEND SOURCES ${APP_MAIN} ${COMMON_SOURCES_Normal} ${COMMON_SOURCES_R} ${COMMON_SOURCES_SINGLE})

set(LINKER_SCRIPT ${CMAKE_SOURCE_DIR}/lib/cmsis/cm4/device_support/startup/gcc/linker/AT32F421x8_FLASH.ld)

add_link_options(-Wl,-gc-sections,--print-memory-usage,-Map=${PROJECT_BINARY_DIR}/${PROJECT_NAME}.map)
add_link_options(-mcpu=cortex-m4 -mthumb -mthumb-interwork)
#add_link_options(-T ${LINKER_SCRIPT})
add_link_options(-specs=nosys.specs)



add_executable(${PROJECT_NAME}.elf ${SOURCES} ${LINKER_SCRIPT})

target_link_options(${PROJECT_NAME}.elf PRIVATE -T ${LINKER_SCRIPT})

target_include_directories(${PROJECT_NAME}.elf PUBLIC "lib/SeggerRTT/RTT" "lib/SeggerRTT/Config")

# For Bootloader's NVIC
set(HEX_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.hex)
set(BIN_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.bin)
set(BOOTLOADER_BIN_FILE ${PROJECT_BINARY_DIR}/bootloader.bin)

add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
        COMMAND ${CMAKE_OBJCOPY} -Oihex $<TARGET_FILE:${PROJECT_NAME}.elf> ${HEX_FILE}
        COMMAND ${CMAKE_OBJCOPY} -Obinary $<TARGET_FILE:${PROJECT_NAME}.elf> ${BIN_FILE}
        COMMENT "Building ${HEX_FILE}
Building ${BIN_FILE}")

