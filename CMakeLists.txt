cmake_minimum_required(VERSION 3.2)

if(NOT BOARD)
	message(FATAL_ERROR "BOARD must be set (eg px4cannode-v1)")
endif()

include(configs/${BOARD}/${BOARD}.cmake)

set(CHIP_TYPE ${CHIP_TYPE} CACHE INTERNAL "system processor" FORCE)
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE) 
set(CMAKE_TOOLCHAIN_FILE ${CMAKE_SOURCE_DIR}/cmake/Toolchain-arm-none-eabi.cmake CACHE INTERNAL "toolchain file" FORCE)

project(${BOARD}_px4_nuttx_bootloader LANGUAGES ASM C CXX)

message(STATUS "PX4 NuttX bootloader: ${BOARD} ${CHIP_TYPE}")

# CMake build type
# Debug Release RelWithDebInfo MinSizeRel Coverage
if (NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE "MinSizeRel" CACHE STRING "Build type" FORCE)
endif()
set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug;Release;RelWithDebInfo;MinSizeRel")

message(STATUS "Build Type: ${CMAKE_BUILD_TYPE}")

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)

if (CMAKE_HOST_APPLE OR TRUE)
	# copy with rsync and create file dependencies
	set(cp_cmd "rsync")
	set(cp_opts)
	list(APPEND cp_opts
		-rp
		--inplace
	)
else()
	# copy with hard links
	# archive, recursive, force, link (hardlinks)
	set(cp_cmd "cp")
	set(cp_opts "-aRfl")
endif()

file(GLOB_RECURSE copy_nuttx_files
	LIST_DIRECTORIES false
	${CMAKE_CURRENT_SOURCE_DIR}/nuttx/*)

# copy nuttx to build directory and create a null apps dir
add_custom_command(OUTPUT nuttx_copy.stamp
	COMMAND ${cp_cmd} ${cp_opts} ${CMAKE_CURRENT_SOURCE_DIR}/nuttx ${CMAKE_CURRENT_BINARY_DIR}
	COMMAND mkdir ${CMAKE_CURRENT_BINARY_DIR}/apps || true
	COMMAND echo 'dirlinks:' > ${CMAKE_CURRENT_BINARY_DIR}/apps/Makefile
	COMMAND echo 'preconfig:' >> ${CMAKE_CURRENT_BINARY_DIR}/apps/Makefile
	COMMAND echo 'clean:' >> ${CMAKE_CURRENT_BINARY_DIR}/apps/Makefile
	COMMAND echo 'distclean:' >> ${CMAKE_CURRENT_BINARY_DIR}/apps/Makefilecontext:
	COMMAND echo -n > ${CMAKE_CURRENT_BINARY_DIR}/apps/Kconfig
	COMMAND cmake -E touch nuttx_copy.stamp
	DEPENDS ${copy_nuttx_files}
	COMMENT "Copying NuttX/nuttx to ${CMAKE_CURRENT_BINARY_DIR}"
	)

set(NUTTX_DIR ${CMAKE_CURRENT_BINARY_DIR}/nuttx)
set(NUTTX_CONFIG_DIR ${CMAKE_SOURCE_DIR}/configs/${BOARD}/nuttx-config)

# copy PX4 board config into nuttx
file(GLOB_RECURSE board_config_files ${NUTTX_CONFIG_DIR})
add_custom_command(OUTPUT
			${NUTTX_DIR}/PX4_Config.mk
			${NUTTX_DIR}/PX4_Warnings.mk
			${NUTTX_DIR}/.config
			${NUTTX_DIR}/Make.defs
			${NUTTX_DIR}/configs/${BOARD}/bootloader/defconfig
	COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/include/PX4_Config.mk ${NUTTX_DIR}/PX4_Config.mk
	COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/include/PX4_Warnings.mk ${NUTTX_DIR}/PX4_Warnings.mk
	COMMAND ${CMAKE_COMMAND} -E copy ${NUTTX_CONFIG_DIR}/bootloader/defconfig ${NUTTX_DIR}/.config
	COMMAND ${CMAKE_COMMAND} -E remove -f ${NUTTX_DIR}/include/nuttx/config.h
	COMMAND ${CMAKE_COMMAND} -E copy ${NUTTX_CONFIG_DIR}/bootloader/Make.defs ${NUTTX_DIR}/Make.defs
	COMMAND ${cp_cmd} ${cp_opts} ${NUTTX_CONFIG_DIR}/ ${NUTTX_DIR}/configs/${BOARD}
	DEPENDS
		${CMAKE_SOURCE_DIR}/include/PX4_Config.mk
		${CMAKE_SOURCE_DIR}/include/PX4_Warnings.mk
		${NUTTX_CONFIG_DIR}/bootloader/defconfig
		${NUTTX_CONFIG_DIR}/bootloader/Make.defs
		${board_config_files}
		nuttx_copy.stamp
	WORKING_DIRECTORY ${NUTTX_DIR}/tools
	COMMENT "Copying NuttX config ${BOARD} and configuring"
	)
add_custom_target(nuttx_copy DEPENDS ${NUTTX_DIR}/.config)

set(ARCH_SRC "arch/arm/src")

# context
add_custom_command(OUTPUT ${NUTTX_DIR}/include/nuttx/version.h ${NUTTX_DIR}/include/nuttx/config.h
	COMMAND make --no-print-directory --silent include/nuttx/config.h > /dev/null
	COMMAND make --no-print-directory --silent include/nuttx/version.h > /dev/null
	COMMAND make --no-print-directory --silent include/math.h > /dev/null
	COMMAND make --no-print-directory --silent include/float.h > /dev/null
	COMMAND make --no-print-directory --silent include/stdarg.h > /dev/null
	COMMAND make --no-print-directory --silent include/arch > /dev/null
	COMMAND make --no-print-directory --silent include/arch/board > /dev/null
	COMMAND make --no-print-directory --silent include/arch/chip > /dev/null

	COMMAND make --no-print-directory --silent ${ARCH_SRC}/board > /dev/null
	COMMAND make --no-print-directory --silent ${ARCH_SRC}/chip > /dev/null

	COMMAND make --no-print-directory --silent -C configs dirlinks TOPDIR="${NUTTX_DIR}" > /dev/null
	COMMAND make --no-print-directory --silent -C configs context TOPDIR="${NUTTX_DIR}" > /dev/null
	COMMAND make --no-print-directory --silent -C libc context TOPDIR="${NUTTX_DIR}" > /dev/null

	DEPENDS nuttx_configure ${NUTTX_DIR}/.config
	WORKING_DIRECTORY ${NUTTX_DIR}
	)
add_custom_target(nuttx_configure DEPENDS ${NUTTX_DIR}/include/nuttx/version.h)

# library of NuttX libraries
add_library(nuttx_build INTERFACE)

# libboard.a
add_custom_command(OUTPUT ${NUTTX_DIR}/configs/${BOARD}/src/libboard.a
	COMMAND make --silent --no-print-directory -C board TOPDIR="${NUTTX_DIR}" libboard.a EXTRADEFINES=-D__KERNEL__ > /dev/null
	DEPENDS nuttx_configure
	WORKING_DIRECTORY ${NUTTX_DIR}/arch/arm/src
	)
add_custom_target(nuttx_board_build DEPENDS ${NUTTX_DIR}/configs/${BOARD}/src/libboard.a)
add_library(nuttx_board STATIC IMPORTED GLOBAL)
set_property(TARGET nuttx_board PROPERTY IMPORTED_LOCATION ${NUTTX_DIR}/configs/${BOARD}/src/libboard.a)

add_dependencies(nuttx_build nuttx_board_build)
target_link_libraries(nuttx_build INTERFACE nuttx_board)

# helper for all targets
function(add_nuttx_dir nuttx_lib nuttx_lib_dir kernel extra)
	file(GLOB_RECURSE nuttx_lib_files
		LIST_DIRECTORIES false
		${CMAKE_CURRENT_SOURCE_DIR}/nuttx/${nuttx_lib_dir}/*)

	add_custom_command(OUTPUT ${NUTTX_DIR}/${nuttx_lib_dir}/lib${nuttx_lib}.a
		COMMAND find ${nuttx_lib_dir} -type f -name *.o -o -name .built -delete
		COMMAND make -C ${nuttx_lib_dir} -j2 --silent --no-print-directory lib${nuttx_lib}.a TOPDIR="${NUTTX_DIR}" KERNEL=${kernel} EXTRADEFINES=${extra} > /dev/null
		DEPENDS ${nuttx_lib_files} nuttx_configure
		WORKING_DIRECTORY ${NUTTX_DIR}
		)
	add_custom_target(nuttx_${nuttx_lib}_build DEPENDS ${NUTTX_DIR}/${nuttx_lib_dir}/lib${nuttx_lib}.a)
	add_library(nuttx_${nuttx_lib} STATIC IMPORTED GLOBAL)
	set_property(TARGET nuttx_${nuttx_lib} PROPERTY IMPORTED_LOCATION ${NUTTX_DIR}/${nuttx_lib_dir}/lib${nuttx_lib}.a)

	add_dependencies(nuttx_build nuttx_${nuttx_lib}_build)
	target_link_libraries(nuttx_build INTERFACE nuttx_${nuttx_lib})
endfunction()

# add_nuttx_dir(NAME DIRECTORY KERNEL EXTRA)
add_nuttx_dir(arch arch/arm/src y -D__KERNEL__)
add_nuttx_dir(configs configs y -D__KERNEL__)
add_nuttx_dir(drivers drivers y -D__KERNEL__)
add_nuttx_dir(fs fs y -D__KERNEL__)
add_nuttx_dir(sched sched y -D__KERNEL__)
add_nuttx_dir(c libc n "")

add_compile_options($<$<COMPILE_LANGUAGE:CXX>:"-std=c99">)
add_compile_options($<$<COMPILE_LANGUAGE:CXX>:"-std=c++11 -fno-exceptions -fno-rtti -fno-threadsafe-statics">)

# build NuttX stm32_vectors.S
add_library(nuttx_startup ${CMAKE_SOURCE_DIR}/nuttx/arch/arm/src/stm32/gnu/stm32_vectors.S)
target_compile_definitions(nuttx_startup PRIVATE -D__ASSEMBLY__)
add_dependencies(nuttx_startup nuttx_arch_build)

add_compile_options(-fvisibility=hidden -include visibility.h
		-fstrict-aliasing
		-fomit-frame-pointer
		-funsafe-math-optimizations
		-fno-common
		-ffunction-sections
		-fdata-sections
		-fno-strength-reduce
		-fno-builtin-printf
#		-flto - not usable with debugging

		-g

		-Wall
		-Wextra
		-Werror

		-Wno-unused-parameter
		-Wno-sign-compare
	)

# Bring in common uavcan hardware identity definitions
include(${CMAKE_SOURCE_DIR}/cmake/uavcan_board_ident/${BOARD}.cmake)
add_definitions(
		-DHW_UAVCAN_NAME=${uavcanblid_name}
		-DHW_VERSION_MAJOR=${uavcanblid_hw_version_major}
		-DHW_VERSION_MINOR=${uavcanblid_hw_version_minor}
	-D__PX4_NUTTX
	-DCONFIG_WCHAR_BUILTIN
	-D__CUSTOM_FILE_IO__
)

include_directories(
	${CMAKE_SOURCE_DIR}/src/include
	${CMAKE_SOURCE_DIR}/configs/${BOARD}
	${CMAKE_BINARY_DIR}/nuttx/arch/arm/src/armv7-m
	${CMAKE_BINARY_DIR}/nuttx/arch/arm/src/chip
	${CMAKE_BINARY_DIR}/nuttx/arch/arm/src/common
	${CMAKE_BINARY_DIR}/nuttx/include
	)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PX4_BINARY_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG ${PX4_BINARY_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE ${PX4_BINARY_DIR})

set(fw_name ${BOARD}.elf)
add_executable(${fw_name} ${NUTTX_CONFIG_DIR}/src/empty.c)
add_dependencies(${fw_name} nuttx_build nuttx_startup)

add_subdirectory(src)
add_subdirectory(configs/${BOARD})
add_dependencies(${fw_name} drivers_bootloaders ${BOARD})

target_link_libraries(${fw_name}
	-T${NUTTX_CONFIG_DIR}/scripts/ld.script
	-Wl,-Map=${CMAKE_BINARY_DIR}/${BOARD}.map
	-Wl,-wrap,sched_process_timer
	-Wl,--warn-common
	-Wl,--gc-sections
	-Wl,--start-group
	${BOARD}
	drivers_bootloaders
	nuttx_arch
	nuttx_board
	nuttx_c
	nuttx_sched
	nuttx_startup
	-Wl,--end-group
	)

# oldconfig helper
add_custom_target(oldconfig
	COMMAND make --no-print-directory -C ${NUTTX_DIR} CONFIG_ARCH_BOARD=${BOARD} oldconfig
	COMMAND cp ${NUTTX_DIR}/.config ${NUTTX_CONFIG_DIR}/bootloader/defconfig
	COMMAND ${CMAKE_SOURCE_DIR}/Tools/nuttx_defconf_tool.sh ${NUTTX_CONFIG_DIR}/bootloader/defconfig
	DEPENDS nuttx_configure
	WORKING_DIRECTORY ${NUTTX_DIR}
	COMMENT "Running NuttX make oldconfig for ${BOARD} with bootloader"
	USES_TERMINAL
	)

# menuconfig helper
add_custom_target(menuconfig
	COMMAND make --no-print-directory -C ${NUTTX_DIR} CONFIG_ARCH_BOARD=${BOARD} menuconfig
	COMMAND cp ${NUTTX_DIR}/.config ${NUTTX_CONFIG_DIR}/bootloader/defconfig
	COMMAND ${CMAKE_SOURCE_DIR}/Tools/nuttx_defconf_tool.sh ${NUTTX_CONFIG_DIR}/bootloader/defconfig
	DEPENDS nuttx_configure
	WORKING_DIRECTORY ${NUTTX_DIR}
	COMMENT "Running NuttX make menuconfig for ${BOARD} with bootloader"
	USES_TERMINAL
	)

# qconfig helper
add_custom_target(qconfig
	COMMAND make --no-print-directory -C ${NUTTX_DIR} CONFIG_ARCH_BOARD=${BOARD} qconfig
	COMMAND cp ${NUTTX_DIR}/.config ${NUTTX_CONFIG_DIR}/bootloader/defconfig
	DEPENDS nuttx_configure
	WORKING_DIRECTORY ${NUTTX_DIR}
	COMMENT "Running NuttX make qconfig for ${BOARD} with bootloader"
	USES_TERMINAL
	)
