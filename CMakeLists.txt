# Check ESP-IDF version and error out if it is not in the supported range.
set(min_supported_idf_version "5.3.0")
set(max_supported_idf_version "5.5.99")
set(idf_version "${IDF_VERSION_MAJOR}.${IDF_VERSION_MINOR}.${IDF_VERSION_PATCH}")

if ("${idf_version}" AND NOT "$ENV{ARDUINO_SKIP_IDF_VERSION_CHECK}")
  if (idf_version VERSION_LESS min_supported_idf_version)
    message(FATAL_ERROR "Arduino-esp32 requires IDF >= ${min_supported_idf_version}")
  endif()
  if (idf_version VERSION_GREATER max_supported_idf_version)
    message(FATAL_ERROR "Arduino-esp32 requires IDF <= ${max_supported_idf_version}")
  endif()
endif()

# --- Core sources (ponechane iba potrebne pre Arduino API + I2C) ---
set(CORE_SRCS
  cores/esp32/base64.cpp
  cores/esp32/cbuf.cpp
  cores/esp32/esp32-hal-cpu.c
  cores/esp32/esp32-hal-gpio.c
  cores/esp32/esp32-hal-i2c.c
  cores/esp32/esp32-hal-i2c-ng.c
  cores/esp32/esp32-hal-time.c
  cores/esp32/esp32-hal-timer.c
  cores/esp32/Esp.cpp
  cores/esp32/FunctionalInterrupt.cpp
  cores/esp32/HardwareSerial.cpp
  cores/esp32/IPAddress.cpp
  cores/esp32/MacAddress.cpp
  cores/esp32/main.cpp
  cores/esp32/Print.cpp
  cores/esp32/Stream.cpp
  cores/esp32/StreamString.cpp
  cores/esp32/WMath.cpp
  cores/esp32/WString.cpp
)

# --- Arduino libraries: minimalisticky set ---
set(ARDUINO_ALL_LIBRARIES
  OpenThread
  Wire
  Preferences
)

# OpenThread sources
set(ARDUINO_LIBRARY_OpenThread_SRCS
  libraries/OpenThread/src/OThread.cpp
  libraries/OpenThread/src/OThreadCLI.cpp
  libraries/OpenThread/src/OThreadCLI_Util.cpp)

# Preferences (NVS wrapper)
set(ARDUINO_LIBRARY_Preferences_SRCS
  libraries/Preferences/src/Preferences.cpp)

# Wire (I2C)
set(ARDUINO_LIBRARY_Wire_SRCS
  libraries/Wire/src/Wire.cpp)

# --- Generic handling of libraries ---
set(ARDUINO_LIBRARIES_SRCS)
set(ARDUINO_LIBRARIES_REQUIRES)
set(ARDUINO_LIBRARIES_INCLUDEDIRS)

foreach(libname IN LISTS ARDUINO_ALL_LIBRARIES)
  if(ARDUINO_LIBRARY_${libname}_SRCS)
    list(APPEND ARDUINO_LIBRARIES_SRCS ${ARDUINO_LIBRARY_${libname}_SRCS})
  endif()
  if(ARDUINO_LIBRARY_${libname}_REQUIRES)
    list(APPEND ARDUINO_LIBRARIES_REQUIRES ${ARDUINO_LIBRARY_${libname}_REQUIRES})
  endif()
  if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/libraries/${libname}/src)
    list(APPEND ARDUINO_LIBRARIES_INCLUDEDIRS libraries/${libname}/src)
  endif()
endforeach()

set(includedirs variants/${CONFIG_ARDUINO_VARIANT}/ cores/esp32/ ${ARDUINO_LIBRARIES_INCLUDEDIRS})
set(srcs ${CORE_SRCS} ${ARDUINO_LIBRARIES_SRCS})
set(priv_includes cores/esp32/libb64)

set(requires spi_flash esp_partition esp_adc esp_ringbuf esp_driver_gptimer driver)
set(priv_requires nvs_flash app_update)
# Openthread for C6/H2
if(IDF_TARGET STREQUAL "esp32c6" OR IDF_TARGET STREQUAL "esp32h2" OR IDF_TARGET STREQUAL "esp32c5")
  list(APPEND requires openthread)
endif()

idf_component_register(
  INCLUDE_DIRS ${includedirs}
  PRIV_INCLUDE_DIRS ${priv_includes}
  SRCS ${srcs}
  REQUIRES ${requires}
  PRIV_REQUIRES ${priv_requires}
)

# --- Arduino defines ---
string(TOUPPER ${CONFIG_ARDUINO_VARIANT} idf_target_caps)
string(REPLACE "-" "_" idf_target_for_macro "${idf_target_caps}")
target_compile_options(${COMPONENT_TARGET} PUBLIC
    -DARDUINO=10812
    -DARDUINO_${idf_target_for_macro}_DEV
    -DARDUINO_ARCH_ESP32
    -DARDUINO_BOARD="${idf_target_caps}_DEV"
    -DARDUINO_VARIANT="${CONFIG_ARDUINO_VARIANT}"
    -DESP32=ESP32)

# --- Autostart handling ---
if(CONFIG_AUTOSTART_ARDUINO)
  target_link_libraries(${COMPONENT_LIB} INTERFACE "-u _Z5setupv -u _Z4loopv")
endif()
