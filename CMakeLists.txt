cmake_minimum_required(VERSION 3.14)

project(libsoundio)

set(CMAKE_MODULE_PATH ${libsoundio_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

add_library(libsoundio STATIC "")
set_target_properties(libsoundio PROPERTIES
    LINKER_LANGUAGE C
    PREFIX ""
)

set(LIBSOUNDIO_VERSION_MAJOR 2)
set(LIBSOUNDIO_VERSION_MINOR 0)
set(LIBSOUNDIO_VERSION_PATCH 0)
set(LIBSOUNDIO_VERSION "${LIBSOUNDIO_VERSION_MAJOR}.${LIBSOUNDIO_VERSION_MINOR}.${LIBSOUNDIO_VERSION_PATCH}")

option(SOUNDIO_ENABLE_JACK "Enable JACK backend" ON)
option(SOUNDIO_ENABLE_PULSEAUDIO "Enable PulseAudio backend" ON)
option(SOUNDIO_ENABLE_ALSA "Enable ALSA backend" ON)
option(SOUNDIO_ENABLE_COREAUDIO "Enable CoreAudio backend" ON)
option(SOUNDIO_ENABLE_WASAPI "Enable WASAPI backend" ON)

set(SOUNDIO_SRC_DIR "${CMAKE_CURRENT_SOURCE_DIR}/src")
set(SOUNDIO_INC_DIR "${CMAKE_CURRENT_SOURCE_DIR}/include")
set(SOUNDIO_BIN_DIR "${CMAKE_CURRENT_BINARY_DIR}")

set(SOUNDIO_CONFIG_FILE_OUT "${SOUNDIO_BIN_DIR}/config.h")

set(SOUNDIO_SOURCES_PUBLIC
    "${SOUNDIO_INC_DIR}/soundio/soundio.h"
    "${SOUNDIO_INC_DIR}/soundio/endian.h"
)

set(SOUNDIO_SOURCES_PRIVATE
    "${SOUNDIO_SRC_DIR}/soundio.c"
    "${SOUNDIO_SRC_DIR}/util.c"
    "${SOUNDIO_SRC_DIR}/os.c"
    "${SOUNDIO_SRC_DIR}/dummy.c"
    "${SOUNDIO_SRC_DIR}/channel_layout.c"
    "${SOUNDIO_SRC_DIR}/ring_buffer.c"

    "${SOUNDIO_SRC_DIR}/ring_buffer.h"
    "${SOUNDIO_SRC_DIR}/os.h"
    "${SOUNDIO_SRC_DIR}/list.h"
    "${SOUNDIO_SRC_DIR}/atomics.h"
    "${SOUNDIO_SRC_DIR}/dummy.h"
    "${SOUNDIO_SRC_DIR}/soundio_internal.h"
    "${SOUNDIO_SRC_DIR}/soundio_private.h"
    "${SOUNDIO_SRC_DIR}/util.h"
)

if(SOUNDIO_ENABLE_JACK)
    find_package(JACK)
    if(JACK_FOUND)
        target_include_directories(libsoundio PRIVATE ${JACK_INCLUDE_DIR})
        target_link_libraries(libsoundio PRIVATE ${JACK_LIBRARY})

        set(SOUNDIO_SOURCES_PRIVATE "${SOUNDIO_SRC_DIR}/jack.c" ${SOUNDIO_SOURCES_PRIVATE})
        set(SOUNDIO_SOURCES_PUBLIC "${SOUNDIO_SRC_DIR}/jack.h" ${SOUNDIO_SOURCES_PUBLIC})

        set(SOUNDIO_HAVE_JACK true)
    else()
        set(SOUNDIO_HAVE_JACK false)
    endif()
else()
    set(SOUNDIO_HAVE_JACK false)
endif()

if(SOUNDIO_ENABLE_PULSEAUDIO)
    find_package(PulseAudio)
    if(PULSEAUDIO_FOUND)
        target_include_directories(libsoundio PRIVATE ${PULSEAUDIO_INCLUDE_DIR})
        target_link_libraries(libsoundio PRIVATE ${PULSEAUDIO_LIBRARY})

        set(SOUNDIO_SOURCES_PRIVATE "${SOUNDIO_SRC_DIR}/pulseaudio.c" ${SOUNDIO_SOURCES_PRIVATE})
        set(SOUNDIO_SOURCES_PUBLIC "${SOUNDIO_SRC_DIR}/pulseaudio.h" ${SOUNDIO_SOURCES_PUBLIC})

        set(SOUNDIO_HAVE_PULSEAUDIO true)
    else()
        set(SOUNDIO_HAVE_PULSEAUDIO false)
    endif()
else()
    set(SOUNDIO_HAVE_PULSEAUDIO false)
endif()

if(SOUNDIO_ENABLE_ALSA)
    find_package(ALSA)
    if(ALSA_FOUND)
        target_include_directories(libsoundio PRIVATE ${ALSA_INCLUDE_DIR})
        target_link_libraries(libsoundio PRIVATE ${ALSA_LIBRARY})

        set(SOUNDIO_SOURCES_PRIVATE "${SOUNDIO_SRC_DIR}/alsa.c" ${SOUNDIO_SOURCES_PRIVATE})
        set(SOUNDIO_SOURCES_PUBLIC "${SOUNDIO_SRC_DIR}/alsa.h" ${SOUNDIO_SOURCES_PUBLIC})

        set(SOUNDIO_HAVE_ALSA true)
    else()
        set(SOUNDIO_HAVE_ALSA false)
    endif()
else()
    set(SOUNDIO_HAVE_ALSA false)
endif()

# TODO: add backend coreaudio
if(SOUNDIO_ENABLE_COREAUDIO)
    # set(SOUNDIO_SOURCES_PRIVATE "${SOUNDIO_SRC_DIR}/coreaudio.c" ${SOUNDIO_SOURCES_PRIVATE})
    # set(SOUNDIO_SOURCES_PUBLIC "${SOUNDIO_SRC_DIR}/coreaudio.h" ${SOUNDIO_SOURCES_PUBLIC})

    set(SOUNDIO_HAVE_COREAUDIO false)
else()
    set(SOUNDIO_HAVE_COREAUDIO false)
endif()

# TODO: add backend
if(SOUNDIO_ENABLE_WASAPI)
    # set(SOUNDIO_SOURCES_PRIVATE "${SOUNDIO_SRC_DIR}/wasapi.c" ${SOUNDIO_SOURCES_PRIVATE})
    # set(SOUNDIO_SOURCES_PUBLIC "${SOUNDIO_SRC_DIR}/wasapi.h" ${SOUNDIO_SOURCES_PUBLIC})

    set(SOUNDIO_HAVE_WASAPI false)
else()
    set(SOUNDIO_HAVE_WASAPI false)
endif()

configure_file("${SOUNDIO_SRC_DIR}/config.h.in" ${SOUNDIO_CONFIG_FILE_OUT})
set(SOUNDIO_SOURCES_PRIVATE
    "${SOUNDIO_CONFIG_FILE_OUT}"
    ${SOUNDIO_SOURCES_PRIVATE}
)

target_sources(libsoundio
    PUBLIC
        ${SOUNDIO_SOURCES_PUBLIC}
    PRIVATE
        ${SOUNDIO_SOURCES_PRIVATE}
)

target_include_directories(libsoundio
    PUBLIC
        ${SOUNDIO_INC_DIR}
    PRIVATE
        ${SOUNDIO_SRC_DIR}
        ${SOUNDIO_BIN_DIR}
)