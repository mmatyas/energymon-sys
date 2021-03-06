set(LNAME energymon-osp)
set(LNAME_POLLING energymon-osp-polling)
set(SHMEM_PROVIDER ${LNAME_POLLING}-shmem-provider)
set(SOURCES ${LNAME}.c;../src/energymon-util.c)
set(SOURCES_POLLING ${SOURCES};../src/energymon-time-util.c)
set(DESCRIPTION "EnergyMon implementation for ODROID Smart Power")
set(DESCRIPTION_POLLING "EnergyMon implementation for ODROID Smart Power with Polling")

# Dependencies

find_package(PkgConfig)
if(${PKG_CONFIG_FOUND})
  pkg_search_module(HIDAPIUSB hidapi-libusb)
endif()
if(NOT HIDAPIUSB_FOUND) # doesn't seem to work if we use ${} syntax...
  # fail gracefully
  message(WARNING "${LNAME}: Missing libhidapi-libusb - skipping this project")
  return()
endif()
include_directories(${HIDAPIUSB_INCLUDE_DIRS})

# Must be set to OFF to prevent doing a try_run() during cross-compiling
set(THREADS_PTHREAD_ARG OFF)
find_package(Threads)
if(CMAKE_THREAD_LIBS_INIT)
  set(PKG_CONFIG_PRIVATE_LIBS "${CMAKE_THREAD_LIBS_INIT}")
endif()
if(LIBRT)
  set(PKG_CONFIG_PRIVATE_LIBS "${PKG_CONFIG_PRIVATE_LIBS} -lrt")
endif()

# Libraries

add_library(${LNAME} ${SOURCES})
target_link_libraries(${LNAME} -L${HIDAPIUSB_LIBDIR} ${HIDAPIUSB_LIBRARIES})

add_library(${LNAME_POLLING} ${SOURCES_POLLING})
target_link_libraries(${LNAME_POLLING} -L${HIDAPIUSB_LIBDIR} ${HIDAPIUSB_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT} ${LIBRT})

if(DEFAULT STREQUAL "osp" OR DEFAULT STREQUAL LNAME)
  BUILD_DEFAULT("${SOURCES}" "-L${HIDAPIUSB_LIBDIR};${HIDAPIUSB_LIBRARIES}" "")
  PKG_CONFIG("energymon-default" "${DESCRIPTION}" "${HIDAPIUSB_LIBRARIES}" "")
  install(TARGETS energymon-default DESTINATION lib)
elseif(DEFAULT STREQUAL "osp-polling" OR DEFAULT STREQUAL LNAME_POLLING)
  BUILD_DEFAULT("${SOURCES_POLLING}" "-L${HIDAPIUSB_LIBDIR};${HIDAPIUSB_LIBRARIES};${CMAKE_THREAD_LIBS_INIT};${LIBRT}" "ENERGYMON_OSP_USE_POLLING")
  PKG_CONFIG("energymon-default" "${DESCRIPTION_POLLING}" "${HIDAPIUSB_LIBRARIES}" "${PKG_CONFIG_PRIVATE_LIBS}")
  install(TARGETS energymon-default DESTINATION lib)
endif()

set_target_properties(${LNAME_POLLING} PROPERTIES COMPILE_DEFINITIONS "ENERGYMON_OSP_USE_POLLING")

# pkg-config

PKG_CONFIG("${LNAME}" "${DESCRIPTION}" "${HIDAPIUSB_LIBRARIES}" "")
PKG_CONFIG("${LNAME_POLLING}" "${DESCRIPTION_POLLING}" "${HIDAPIUSB_LIBRARIES}" "${PKG_CONFIG_PRIVATE_LIBS}")

# Binaries

add_executable(${SHMEM_PROVIDER} energymon-osp-polling-shmem-provider.c)
target_link_libraries(${SHMEM_PROVIDER} ${LNAME_POLLING})

# Install

install(TARGETS ${LNAME} ${LNAME_POLLING} DESTINATION lib)
install(TARGETS ${SHMEM_PROVIDER} DESTINATION bin)
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/${LNAME}.h ${CMAKE_CURRENT_SOURCE_DIR}/${LNAME_POLLING}.h
        DESTINATION include/${PROJECT_NAME})
