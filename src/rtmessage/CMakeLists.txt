##########################################################################
# If not stated otherwise in this file or this component's LICENSE
# file the following copyright and licenses apply:
#
# Copyright 2019 RDK Management
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
##########################################################################

set(OPENSSL_REQUIRED FALSE)

if (WITH_SPAKE2)
  message("spake2 enabled")
  set(OPENSSL_REQUIRED TRUE)
  add_definitions(-DWITH_SPAKE2=1)
  if (BUILD_FOR_DESKTOP)
    add_definitions(-DWITH_SPAKE2_TEST_PIN=1)
    include_directories(${CMAKE_INSTALL_PREFIX}/include)
    link_directories(${CMAKE_INSTALL_PREFIX}/lib)
  endif (BUILD_FOR_DESKTOP)
  find_library(SPAKE2_LIBRARY spake2plus HINTS "${LIBEXCHANGER_PATH}/Release/src")
  find_library(PWD_LIBRARY pwd HINTS "${LIBEXCHANGER_PATH}/password/src")
endif (WITH_SPAKE2)

if (MSG_ROUNDTRIP_TIME)
  message("Message rountrip time calculation enabled")
  add_definitions(-DMSG_ROUNDTRIP_TIME=1)
endif (MSG_ROUNDTRIP_TIME)

if (OPENSSL_REQUIRED)
  message("openssl enabled")
  if( OPENSSL_ROOT_DIR )
    if( OPENSSL_VERSION )
        find_package(OpenSSL "${OPENSSL_VERSION}" EXACT REQUIRED PATH "${OPENSSL_ROOT_DIR}")
    else()
        find_library(SSL_LIBRARY ssl HINTS "${OPENSSL_ROOT_DIR}/lib")
        find_library(CRYPTO_LIBRARY crypto HINTS "${OPENSSL_ROOT_DIR}/lib")
        find_package(OpenSSL REQUIRED PATH "${OPENSSL_ROOT_DIR}")
    endif()
  else()
    if( OPENSSL_VERSION )
        find_package(OpenSSL "${OPENSSL_VERSION}" EXACT REQUIRED )
    else()
        find_package(OpenSSL REQUIRED )
    endif()
    include_directories("${OPENSSL_INCLUDE_DIR}")
  endif()
  include_directories("${OPENSSL_INCLUDE_DIR}")
endif (OPENSSL_REQUIRED)

add_library(
      rtMessage
      SHARED
      rtConnection.c
      rtCipher.c
      rtLog.c
      rtError.c
      rtMessageHeader.c
      rtEncoder.c
      rtMessage.c
      rtSocket.c
      rtVector.c
      rtList.c
      rtTime.c
      rtSemaphore.c
      rtRetainable.c
      rtBase64.c
      rtThreadPool.c
      rtHashMap.c
      rtMemory.c
      local_benchmarking.c)

include_directories(${CMAKE_CURRENT_SOURCE_DIR})
add_dependencies(rtMessage cjson)
target_link_libraries(rtMessage -pthread ${CJSON_LIBRARIES})

if (ENABLE_RDKLOGGER)
    add_dependencies(rdklogger)
    target_link_libraries(-lrdkloggers)
endif (ENABLE_RDKLOGGER)

if (OPENSSL_REQUIRED)
      target_link_libraries(rtMessage ${SSL_LIBRARY} ${CRYPTO_LIBRARY})
endif (OPENSSL_REQUIRED)

if (WITH_SPAKE2)
      include_directories(${LIBEXCHANGER_PATH}/src ${LIBEXCHANGER_PATH}/password/src ${LIBSYSWRAPPER_PATH}/source)
      target_link_libraries(rtMessage ${SPAKE2_LIBRARY})
endif (WITH_SPAKE2)


if (BUILD_RBUS_DAEMON)
    message ("Building rtrouted")
    add_executable(rtrouted rtrouted.c rtRoutingTree.c)
    target_link_libraries(rtrouted rtMessage)

    if (INCLUDE_BREAKPAD)
      add_dependencies(rtrouted ${BREAKPAD_PATH})
      target_link_libraries(rtrouted -lbreakpadwrap)
    endif (INCLUDE_BREAKPAD)

    # rtrouted_diag
    add_executable(rtrouted_diag diag_probe.c)
    add_dependencies(rtrouted_diag rtMessage)
    target_link_libraries(rtrouted_diag rtMessage)
endif (BUILD_RBUS_DAEMON)


install (TARGETS rtMessage rtrouted rtrouted_diag RUNTIME DESTINATION bin LIBRARY DESTINATION lib)
