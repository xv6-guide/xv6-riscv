cmake_minimum_required(VERSION 3.28)

project(mkfs LANGUAGES C)

add_compile_options(-Werror -Wall)
include_directories(. ..)

add_executable(mkfs "mkfs.c")

install(TARGETS mkfs)
