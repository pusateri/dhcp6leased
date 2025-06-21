cmake_minimum_required(VERSION 3.12)

project(dhcp6leased)

find_package(BISON REQUIRED)
if(BISON_FOUND)
    bison_target(dhcp6leased1 parse.y ${CMAKE_CURRENT_BINARY_DIR}/parse.tab.c)
    bison_target(dhcp6leased2 parse_lease.y ${CMAKE_CURRENT_BINARY_DIR}/parse_lease.tab.c COMPILE_FLAGS "-ppl")
endif()

find_package(PkgConfig)

include(CheckSymbolExists)
check_symbol_exists(timespecsub "sys/time.h" HAVE_TIMESPECSUB)
check_symbol_exists(reallocarray "stdlib.h" HAVE_REALLOCARRAY)
check_symbol_exists(__OpenBSD__ "stdio.h" HAVE_OPENBSD)

set(
    HEADER_FILES
    control.h
    dhcp6leased.h
    engine.h
    frontend.h
    log.h
)

if(NOT HAVE_OPENBSD)
    add_library(
        compat 
        compat/imsg-buffer.c
        compat/imsg.c
        compat/imsg.h
        compat/queue.h
        compat/reallocarray.c
        compat/reallocarray.h
    )
endif()

if(HAVE_REALLOCARRAY)
    list(APPEND CMAKE_C_FLAGS "-DHAVE_REALLOCARRAY")
endif()

if(HAVE_TIMESPECSUB)
    list(APPEND CMAKE_C_FLAGS "-DHAVE_TIMESPECSUB")
else()
    add_library(
        timespec
        timespec/timespec.c
        timespec/timespec.h
    )
endif()

list(JOIN CMAKE_C_FLAGS " " CMAKE_C_FLAGS)

add_executable(
    dhcp6leased
    control.c
    dhcp6leased.c
    engine.c
    frontend.c
    log.c
    printconf.c
    ${HEADER_FILES}
    ${BISON_dhcp6leased1_OUTPUTS}
    ${BISON_dhcp6leased2_OUTPUTS}
)

pkg_check_modules(LIBEVENT REQUIRED libevent)

target_include_directories(
    dhcp6leased
    PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}
    PRIVATE ${LIBEVENT_INCLUDE_DIRS}
)

if(NOT HAVE_OPENBSD)
    target_include_directories(
        dhcp6leased
        PRIVATE compat
    )
endif()

if(NOT HAVE_TIMESPECSUB)
    target_include_directories(
        dhcp6leased
        PRIVATE timespec
    )
endif()

if(HAVE_OPENBSD)
    target_link_libraries(dhcp6leased util)
else()
    target_link_libraries(dhcp6leased compat)
endif()

if(NOT HAVE_TIMESPECSUB)
    target_link_libraries(dhcp6leased timespec)
endif()

target_link_libraries(dhcp6leased ${LIBEVENT_LDFLAGS})
