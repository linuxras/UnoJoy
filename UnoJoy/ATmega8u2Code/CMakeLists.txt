cmake_minimum_required(VERSION 3.10)

set(FEXT "")
if(CMAKE_HOST_WIN32)
    set(FEXT ".exe")
endif()
# attempt to find avr-gcc
find_path(TOOLCHAIN_ROOT
	NAMES
    avr-gcc${FEXT}

	PATHS
		/usr/bin
		/usr/local/bin
		/bin
		$ENV{AVR_ROOT}
)

# Error, could not find toolchain
if(NOT TOOLCHAIN_ROOT)
    if(CMAKE_HOST_WIN32)
        message(WARNING "Try setting the AVR_ROOT environment variable to the directory that contains avr-gcc.exe")
    endif()
	message(FATAL_ERROR "Toolchain root could not be found!!!")
endif(NOT TOOLCHAIN_ROOT)

# program names
set(AVRCPP   ${TOOLCHAIN_ROOT}/avr-g++${FEXT})
set(AVRC     ${TOOLCHAIN_ROOT}/avr-gcc${FEXT})
set(AVRSTRIP ${TOOLCHAIN_ROOT}/avr-strip${FEXT})
set(OBJCOPY  ${TOOLCHAIN_ROOT}/avr-objcopy${FEXT})
set(OBJDUMP  ${TOOLCHAIN_ROOT}/avr-objdump${FEXT})
set(NM ${TOOLCHAIN_ROOT}/avr-nm${FEXT})
set(AVRSIZE  ${TOOLCHAIN_ROOT}/avr-size)
set(AVRDUDE  avrdude)

# Use AVR GCC toolchain
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_CXX_COMPILER ${AVRCPP})
set(CMAKE_C_COMPILER ${AVRC})
set(CMAKE_ASM_COMPILER ${AVRC})

# Set project name and target
project(UnoJoy C CXX ASM)

set(PRODUCT_NAME UnoJoy)

## AVR Chip Configuration
# 16Mhz, this should match the crystal on your board
set(F_CPU 16000000)
# CPU, you can find the list here:
# https://gcc.gnu.org/onlinedocs/gcc/AVR-Options.html
set(MCU at90usb1286)
# The programmer to use, read avrdude manual for list
set(PROG_TYPE stk500v2)
set(AVRDUDE_FLAGS "-p ${MCU} -P com1 -c ${PROG_TYPE}")

# Set source files
set(SRC
    UnoJoy.c
    usb_gamepad.c
)

# Pass defines to compiler
add_definitions(
    -DF_CPU=${F_CPU}UL
)

set(DEBUG dwarf-2)
# mmcu MUST be passed to bot the compiler and linker, this handle the linker
#set(CMAKE_EXE_LINKER_FLAGS -mmcu=${MCU})
set(CMAKE_EXE_LINKER_FLAGS "-Wl,-Map=${PRODUCT_NAME}.map,--cref -Wl,--relax -Wl,--gc-sections -L${CMAKE_BINARY_DIR}/lib -mmcu=${MCU}")
# Set assembler flags
#set(CMAKE_ASM_FLAGS "-g -Wa,-adhlns=${OBJDIR}/%:%.lst,-gstabs,--listing-cont-lines=100 -mmcu=${MCU} -DF_CPU=${F_CPU}")

add_compile_options(
    -std=gnu99 # C99 standard
#    -Os # optimize
    -g
    -g${DEBUG}
    -O2
    -Wall #Enable warnings
    -Wstrict-prototypes
    -funsigned-char
    -funsigned-bitfields
    -ffunction-sections
    -fpack-struct
    -fshort-enums
    -mmcu=${MCU}
)

# Set object files directory
#set(OBJDIR ${CMAKE_BINARY_DIR}/build)

# Set CFLAGS (optimization, standard, etc.)
#set(CMAKE_C_FLAGS "-g -O2 -Wall -Wstrict-prototypes -funsigned-char -funsigned-bitfields -ffunction-sections -fpack-struct -fshort-enums -mmcu=${MCU} -DF_CPU=${F_CPU}UL")

# Set C++ flags (if used)
#set(CMAKE_CXX_FLAGS "-g -O2 -Wall -funsigned-char -funsigned-bitfields -fpack-struct -fshort-enums -fno-exceptions -mmcu=${MCU} -DF_CPU=${F_CPU}UL")


# Add executable target
add_executable(${PRODUCT_NAME} ${SRC})

# Rename the output to .elf
set_target_properties(${PRODUCT_NAME} PROPERTIES OUTPUT_NAME ${PRODUCT_NAME}.elf)

# Set linker flags
#set(LINK_FLAGS "-Wl,-Map=${PRODUCT_NAME}.map,--cref -Wl,--relax -Wl,--gc-sections -L${CMAKE_BINARY_DIR}/lib")

# Link libraries
target_link_libraries(${PRODUCT_NAME}
    -lm # math library
    # Add other libraries here if necessary, such as printf libraries
)

# Define the HEX and ELF conversion
# Transform binary into hex file, we ignore the eeprom segments in the step
add_custom_target(hex ALL ${OBJCOPY} -O ihex -R .eeprom -R .fuse -R .lock -R .signature ${PRODUCT_NAME}.elf ${PRODUCT_NAME}.hex DEPENDS ${PRODUCT_NAME})

# Transform binary into hex file, this is the eeprom part (empty if you don't
# use eeprom static variables)
add_custom_target(eeprom ALL ${OBJCOPY} -j .eeprom --set-section-flags=.eeprom="alloc,load" --change-section-lma .eeprom=0 --no-change-warnings -O ihex ${PRODUCT_NAME}.elf ${PRODUCT_NAME}.eep DEPENDS ${PRODUCT_NAME})

add_custom_target(lss ${OBJDUMP} -h -S -z ${PRODUCT_NAME}.elf > ${PRODUCT_NAME}.lss DEPENDS ${PRODUCT_NAME})

add_custom_target(sym ${NM} -n ${PRODUCT_NAME}.elf > ${PRODUCT_NAME}.sym DEPENDS ${PRODUCT_NAME})

# Set up programming with avrdude
add_custom_target(upload ${AVRDUDE} ${AVRDUDE_FLAGS} -U flash:w:${PRODUCT_NAME}.hex DEPENDS hex)

add_custom_target(upload_eeprom ${AVRDUDE} ${AVRDUDE_FLAGS} -p ${MCU}  -U eeprom:w:${PRODUCT_NAME}.eep DEPENDS eeprom)

# Clean extra files
set_directory_properties(PROPERTIES ADDITIONAL_MAKE_CLEAN_FILES "${PRODUCT_NAME}.hex;${PRODUCT_NAME}.eep;${PRODUCT_NAME}.lst;${PRODUCT_NAME}.map")
