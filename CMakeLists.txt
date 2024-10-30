cmake_minimum_required(VERSION 3.28)

# configure compiler and linker first
find_program(CMAKE_C_COMPILER NAMES
  "riscv64-unknown-linux-gnu-gcc" "riscv64-unknown-elf-gcc" "riscv64-linux-gnu-gcc"
  REQUIRED)

get_filename_component(C_COMPILER_BIN ${CMAKE_C_COMPILER} NAME)
string(REPLACE "-gcc" "" CROSS_COMPILE ${C_COMPILER_BIN})
find_program(CMAKE_LINKER "${CROSS_COMPILE}-ld" REQUIRED)
find_program(OBJDUMP "${CROSS_COMPILE}-objdump" REQUIRED)
find_program(OBJCOPY "${CROSS_COMPILE}-objcopy" REQUIRED)

# check for qemu
find_program(QEMU "qemu-system-riscv64" REQUIRED)

# skip compiling a test program
set(CMAKE_C_COMPILER_WORKS 1)

project("xv6-riscv" C ASM)

add_compile_options(
  # enables common warnings
  -Wall

  # treat warnings as errors
  -Werror

  # basic optimizations
  -O

  # prevents the compiler from omitting the frame pointer,
  # which can aid in debugging and generating accurate stack traces
  -fno-omit-frame-pointer
  -ggdb
  -gdwarf-2

  # Generate code for the medium-any code model.
  # The program and its statically defined symbols must be within any single
  # 2 GiB address range. Programs can be statically or dynamically linked.
  -mcmodel=medany

  # This flag tells the compiler that the program is running in a "freestanding"
  # environment, meaning there is no standard library or operating system. It
  # prevents the compiler from assuming that standard library functions like
  # printf or exit are available.
  -ffreestanding

  # disable position-independent code
  -fno-pie -no-pie

  # disables the generation of stack protection code
  -fno-stack-protector
)

add_link_options(
  # configure the linker to align sections to 4 KB boundaries, optimizing memory
  # layout and potentially reducing binary size, especially useful in
  # resource-constrained environments
  -z max-page-size=4096

  # revent the compiler from linking against any standard libraries, such as the
  # C standard library, and skips the standard startup files
  -nostdlib

  # tell the linker to remove unused code and data sections from the final
  # binary
  -Wl,--gc-sections

  # prevent embedding a unique identifier in the binary
  # which would move the entry code to other than 0x80000000
  -Wl,--build-id=none

  # Prevent from using a dynamic linker, making it suitable for
  # statically linked binaries.
  -Wl,--no-dynamic-linker
)

# find headers in directory of source file and kernel
include_directories("." "kernel")

function(add_executable_with_asm_and_sym target_name)
  add_executable(${target_name} ${ARGN})
  add_custom_target(${target_name}_asm ALL
    COMMAND ${OBJDUMP} -S $<TARGET_FILE:${target_name}> > ${target_name}.asm
    DEPENDS ${target_name}
    BYPRODUCTS ${target_name}.asm
  )

  add_custom_target(${target_name}_sym ALL
    COMMAND ${OBJDUMP} -t $<TARGET_FILE:${target_name}> |
            sed -e '1,/SYMBOL TABLE/d' -e 's/ .* / /' -e '/^$$/d'
            > ${target_name}.sym
    DEPENDS ${target_name}
    BYPRODUCTS ${target_name}.sym
  )

  install(FILES ${CMAKE_BINARY_DIR}/${target_name}.asm DESTINATION asm)
  install(FILES ${CMAKE_BINARY_DIR}/${target_name}.sym DESTINATION sym)
endfunction()

# build kernel
set(KERNEL_SOURCE
  kernel/entry.S # entry must come first to be located at 0x80000000
  kernel/string.c
  kernel/kalloc.c
  kernel/main.c
  kernel/vm.c
  kernel/proc.c
  kernel/swtch.S
  kernel/trampoline.S
  kernel/trap.c
  kernel/syscall.c
  kernel/sysproc.c
  kernel/bio.c
  kernel/fs.c
  kernel/log.c
  kernel/sleeplock.c
  kernel/file.c
  kernel/pipe.c
  kernel/exec.c
  kernel/sysfile.c
  kernel/kernelvec.S
  kernel/plic.c
  kernel/virtio_disk.c

  kernel/start.c
  kernel/console.c
  kernel/printf.c
  kernel/uart.c
  kernel/spinlock.c
)

add_executable_with_asm_and_sym("kernel" ${KERNEL_SOURCE})
find_file(KERNEL_LD NAMES "kernel.ld" PATHS kernel REQUIRED)
target_link_options("kernel" PRIVATE -T ${KERNEL_LD} -static)
install(TARGETS "kernel" DESTINATION .)

add_library("initcode" OBJECT user/initcode.S)
install(TARGETS "initcode" DESTINATION .)


# build userspace
find_package(Perl REQUIRED)
add_custom_target(usys ALL
  COMMAND ${Perl_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/user/usys.pl > usys.S
  DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/user/usys.pl
  BYPRODUCTS usys.S)

SET(ULIB_SOURCE
  usys.S
  user/ulib.c
  user/printf.c
  user/umalloc.c
)

add_library(ulib OBJECT ${ULIB_SOURCE})
find_file(USER_LD NAMES "user.ld" PATHS user REQUIRED)
target_link_options(ulib PRIVATE -T ${USER_LD})

SET(USER_SOURCE
	user/cat.c
	user/echo.c
	user/forktest.c
	user/grep.c
	user/init.c
	user/kill.c
	user/ln.c
	user/ls.c
	user/mkdir.c
	user/rm.c
	user/sh.c
	user/stressfs.c
	user/usertests.c
	user/grind.c
	user/wc.c
	user/zombie.c
)

# for each user/source.c, create an executable _source
foreach(source ${USER_SOURCE})
  get_filename_component(name ${source} NAME_WE)
  add_executable_with_asm_and_sym(_${name} ${source})
  target_link_options(_${name} PRIVATE -T ${USER_LD})
  target_link_libraries(_${name} PRIVATE ulib)
  list(APPEND USER _${name})
endforeach()

# make filesystem image
include(ExternalProject)
ExternalProject_Add(mkfs
  SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/mkfs
  CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
)

find_file(README NAMES README PATHS . REQUIRED)
add_custom_target(fs_img
  ALL ${CMAKE_INSTALL_PREFIX}/bin/mkfs fs.img ${README} ${USER}
  DEPENDS mkfs ${README} ${USER}
  BYPRODUCTS fs.img
)
install(FILES ${CMAKE_BINARY_DIR}/fs.img DESTINATION .)

# show how to run qemu
string(CONCAT QEMU
  "message(STATUS \"\n\nqemu-system-riscv64 -machine virt -bios none "
  "-kernel ${CMAKE_INSTALL_PREFIX}/kernel "
  "-m 128M -smp 3 -nographic "
  "-global virtio-mmio.force-legacy=false "
  "-drive file=${CMAKE_INSTALL_PREFIX}/fs.img,if=none,format=raw,id=x0 "
  "-device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0\n\")"
)

# see initcode[] in kernel/proc.c
string(CONCAT INITCODE
  "message(STATUS \"\n\n${OBJCOPY} --dump-section .text=/dev/stdout "
  "$<TARGET_OBJECTS:initcode> | od -t xC -\n\")"
)

install(CODE "${INITCODE}")
install(CODE "${QEMU}")
