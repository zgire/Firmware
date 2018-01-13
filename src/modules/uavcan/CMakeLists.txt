############################################################################
#
#   Copyright (c) 2015 PX4 Development Team. All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
#
# 1. Redistributions of source code must retain the above copyright
#    notice, this list of conditions and the following disclaimer.
# 2. Redistributions in binary form must reproduce the above copyright
#    notice, this list of conditions and the following disclaimer in
#    the documentation and/or other materials provided with the
#    distribution.
# 3. Neither the name PX4 nor the names of its contributors may be
#    used to endorse or promote products derived from this software
#    without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
# "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
# LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS
# FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE
# COPYRIGHT OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT,
# INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
# BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS
# OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED
# AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
# LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN
# ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
# POSSIBILITY OF SUCH DAMAGE.
#
############################################################################

px4_add_git_submodule(TARGET git_uavcan PATH "libuavcan")

include(ExternalProject)

if ("${OS}" MATCHES "nuttx")

	if(NOT config_uavcan_num_ifaces)
		message(FATAL_ERROR "config_uavcan_num_ifaces not set")
	endif()

	set(LIBUAVCAN_CMAKE_CXX_FLAGS "-DUAVCAN_CPP_VERSION=UAVCAN_CPP03 -DUAVCAN_STM32_NUTTX -DUAVCAN_STM32_NUM_IFACES=${config_uavcan_num_ifaces} -DUAVCAN_STM32_TIMER_NUMBER=5 -fno-exceptions -fno-rtti")
	add_definitions(-DUAVCAN_CPP_VERSION=UAVCAN_CPP03 -DUAVCAN_STM32_NUTTX -DUAVCAN_STM32_NUM_IFACES=${config_uavcan_num_ifaces} -DUAVCAN_STM32_TIMER_NUMBER=5)

	set(LIBUAVCAN_CMAKE_INCLUDES "-isystem${PX4_BINARY_DIR}/NuttX/nuttx/arch/arm/src/common -isystem${PX4_BINARY_DIR}/NuttX/nuttx/arch/arm/src/chip -isystem${PX4_BINARY_DIR}/NuttX/nuttx/include -isystem${PX4_BINARY_DIR}/NuttX/nuttx/include -isystem${PX4_BINARY_DIR}/NuttX/nuttx/include/cxx -isystem${PX4_BINARY_DIR}/NuttX/apps/include")

	set(LIBUAVCAN_CMAKE_FLAGS "${LIBUAVCAN_CMAKE_CXX_FLAGS} ${LIBUAVCAN_CMAKE_INCLUDES}")

	ExternalProject_Add(
		libuavcan
		SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/libuavcan
		GIT_SUBMODULES ${CMAKE_CURRENT_SOURCE_DIR}/libuavcan
		CMAKE_GENERATOR ${CMAKE_GENERATOR}
		CMAKE_ARGS
			-DUAVCAN_PLATFORM=stm32
			-DUAVCAN_USE_CPP03=1
			-DCMAKE_SYSTEM_PROCESSOR=${CMAKE_SYSTEM_PROCESSOR}
			-DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}
			-DCMAKE_CXX_FLAGS=${LIBUAVCAN_CMAKE_FLAGS}
			-DCMAKE_INSTALL_PREFIX=${PX4_BINARY_DIR}/libuavcan
		DEPENDS
			prebuild_targets
		#LOG_BUILD 1
		USES_TERMINAL_BUILD 1
	)

	add_library(uavcan STATIC IMPORTED)
	set_target_properties(uavcan PROPERTIES IMPORTED_LOCATION ${PX4_BINARY_DIR}/libuavcan/lib/libuavcan.a)
	
	add_library(uavcan_stm32_driver STATIC IMPORTED)
	set_target_properties(uavcan_stm32_driver PROPERTIES IMPORTED_LOCATION ${PX4_BINARY_DIR}/libuavcan/lib/libuavcan_stm32_driver.a)

else()
	set(UAVCAN_PLATFORM linux CACHE STRING "uavcan platform")
endif()

add_dependencies(libuavcan prebuild_targets)

px4_add_module(
	MODULE modules__uavcan
	MAIN uavcan
	STACK_MAIN 3200
	STACK_MAX 1500
	INCLUDES
		${PX4_SOURCE_DIR}/mavlink/include/mavlink
		${PX4_BINARY_DIR}/libuavcan/include
		${PX4_BINARY_DIR}/libuavcan/include/dsdlc_generated
		#libuavcan/libuavcan_drivers/posix/include
		libuavcan/libuavcan_drivers/stm32/driver/include
	SRCS
		# Main
		uavcan_main.cpp
		uavcan_servers.cpp
		uavcan_params.c

		# Actuators
		actuators/esc.cpp
		actuators/hardpoint.cpp

		# Sensors
		sensors/sensor_bridge.cpp
		sensors/gnss.cpp
		sensors/mag.cpp
		sensors/baro.cpp

	DEPENDS
		git_uavcan
		platforms__common
		uavcan
		uavcan_stm32_driver
	)

target_link_libraries(modules__uavcan nuttx_cxx uavcan uavcan_stm32_driver nuttx_cxx)
