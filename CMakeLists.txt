cmake_minimum_required(VERSION 3.7)

project(libsispopmq CXX C)

include(GNUInstallDirs)

set(SISPOPMQ_VERSION_MAJOR 1)
set(SISPOPMQ_VERSION_MINOR 2)
set(SISPOPMQ_VERSION_PATCH 0)
set(SISPOPMQ_VERSION "${SISPOPMQ_VERSION_MAJOR}.${SISPOPMQ_VERSION_MINOR}.${SISPOPMQ_VERSION_PATCH}")
message(STATUS "sispopmq v${SISPOPMQ_VERSION}")

set(SISPOPMQ_LIBVERSION 0)


option(BUILD_SHARED_LIBS "Build shared libraries instead of static ones" ON)


list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

configure_file(sispopmq/version.h.in sispopmq/version.h @ONLY)
configure_file(libsispopmq.pc.in libsispopmq.pc @ONLY)

add_library(sispopmq
    sispopmq/address.cpp
    sispopmq/auth.cpp
    sispopmq/bt_serialize.cpp
    sispopmq/connections.cpp
    sispopmq/jobs.cpp
    sispopmq/sispopmq.cpp
    sispopmq/proxy.cpp
    sispopmq/worker.cpp
)
set_target_properties(sispopmq PROPERTIES SOVERSION ${SISPOPMQ_LIBVERSION})

set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)
target_link_libraries(sispopmq PRIVATE Threads::Threads)

# libzmq is nearly impossible to link statically from a system-installed static library: it depends
# on a ton of other libraries, some of which are not all statically available.  If the caller wants
# to mess with this, so be it: they can set up a libzmq target and we'll use it.  Otherwise if they
# asked us to do things statically, don't even try to find a system lib and just build it.
set(sispopmq_build_static_libzmq OFF)
if(TARGET libzmq)
    target_link_libraries(sispopmq PUBLIC libzmq)
elseif(BUILD_SHARED_LIBS)
    include(FindPkgConfig)
    pkg_check_modules(libzmq libzmq>=4.3 IMPORTED_TARGET)

    if(libzmq_FOUND)
        # Debian sid includes a -isystem in the mit-krb package that, starting with pkg-config 0.29.2,
        # breaks cmake's pkgconfig module because it stupidly thinks "-isystem" is a path, so if we find
        # -isystem in the include dirs then hack it out.
        get_property(zmq_inc TARGET PkgConfig::libzmq PROPERTY INTERFACE_INCLUDE_DIRECTORIES)
        list(FIND zmq_inc "-isystem" broken_isystem)
        if(NOT broken_isystem EQUAL -1)
          list(REMOVE_AT zmq_inc ${broken_isystem})
          set_property(TARGET PkgConfig::libzmq PROPERTY INTERFACE_INCLUDE_DIRECTORIES ${zmq_inc})
        endif()

        target_link_libraries(sispopmq PUBLIC PkgConfig::libzmq)
    else()
        set(sispopmq_build_static_libzmq ON)
    endif()
else()
    set(sispopmq_build_static_libzmq ON)
endif()

if(sispopmq_build_static_libzmq)
    message(STATUS "libzmq >= 4.3 not found or static build requested, building bundled version")
    list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/local-libzmq")
    include(LocalLibzmq)
    target_link_libraries(sispopmq PUBLIC libzmq_vendor)
endif()

target_include_directories(sispopmq
    PUBLIC
        $<INSTALL_INTERFACE:>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/cppzmq>
)

target_compile_options(sispopmq PRIVATE -Wall -Wextra -Werror)
set_target_properties(sispopmq PROPERTIES
    CXX_STANDARD 17
    CXX_STANDARD_REQUIRED ON
    CXX_EXTENSIONS OFF
    POSITION_INDEPENDENT_CODE ON
)

function(link_dep_libs target linktype libdirs)
    foreach(lib ${ARGN})
        find_library(link_lib-${lib} NAMES ${lib} PATHS ${libdirs})
        message(STATUS "FIND ${lib} FOUND ${link_lib-${lib}}")
        if(link_lib-${lib})
            target_link_libraries(${target} ${linktype} ${link_lib-${lib}})
        endif()
    endforeach()
endfunction()

# If the caller has already set up a sodium target then we will just link to it, otherwise we go
# looking for it.
if(TARGET sodium)
    target_link_libraries(sispopmq PUBLIC sodium)
    if(sispopmq_build_static_libzmq)
        target_link_libraries(libzmq_vendor INTERFACE sodium)
    endif()
else()
    pkg_check_modules(sodium REQUIRED libsodium IMPORTED_TARGET)

    if(BUILD_SHARED_LIBS)
        target_link_libraries(sispopmq PUBLIC PkgConfig::sodium)
        if(sispopmq_build_static_libzmq)
            target_link_libraries(libzmq_vendor INTERFACE PkgConfig::sodium)
        endif()
    else()
        link_dep_libs(sispopmq PUBLIC "${sodium_STATIC_LIBRARY_DIRS}" ${sodium_STATIC_LIBRARIES})
        target_include_directories(sispopmq PUBLIC ${sodium_STATIC_INCLUDE_DIRS})
        if(sispopmq_build_static_libzmq)
            link_dep_libs(libzmq_vendor INTERFACE "${sodium_STATIC_LIBRARY_DIRS}" ${sodium_STATIC_LIBRARIES})
            target_link_libraries(libzmq_vendor INTERFACE ${sodium_STATIC_INCLUDE_DIRS})
        endif()
    endif()
endif()

add_library(sispopmq::sispopmq ALIAS sispopmq)

export(
    TARGETS sispopmq
    NAMESPACE sispopmq::
    FILE sispopmqTargets.cmake
)
install(
    TARGETS sispopmq
    EXPORT sispopmqConfig
    DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

install(
    FILES sispopmq/address.h
          sispopmq/auth.h
          sispopmq/base32z.h
          sispopmq/base64.h
          sispopmq/batch.h
          sispopmq/bt_serialize.h
          sispopmq/bt_value.h
          sispopmq/connections.h
          sispopmq/hex.h
          sispopmq/sispopmq.h
          sispopmq/message.h
          sispopmq/string_view.h
          ${CMAKE_CURRENT_BINARY_DIR}/sispopmq/version.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/sispopmq
)

option(SISPOPMQ_INSTALL_CPPZMQ "Install cppzmq header with sispopmq/ headers" ON)
if(SISPOPMQ_INSTALL_CPPZMQ)
  install(
    FILES cppzmq/zmq.hpp
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/sispopmq
  )
endif()


install(
    FILES ${CMAKE_CURRENT_BINARY_DIR}/libsispopmq.pc
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
)

if(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME)
  set(sispopmq_IS_TOPLEVEL_PROJECT TRUE)
else()
  set(sispopmq_IS_TOPLEVEL_PROJECT FALSE)
endif()

option(SISPOPMQ_BUILD_TESTS "Building and perform sispopmq tests" ${sispopmq_IS_TOPLEVEL_PROJECT})
if(SISPOPMQ_BUILD_TESTS)
    add_subdirectory(tests)
endif()

