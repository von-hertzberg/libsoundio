# libsoundio

C library providing cross-platform audio input and output. The API is
suitable for real-time software such as digital audio workstations as well
as consumer software such as music players.

This library is an abstraction; however in the delicate balance between
performance and power, and API convenience, the scale is tipped closer to
the former. Features that only exist in some sound backends are exposed.

## Note on this Fork

This fork is intended to ease the use of libsoundio as part of a cmake
project using `add_subdirectory`.

## Features and Limitations

 * Supported operating systems:
   - Windows 7+
   - MacOS 10.10+
   - Linux 3.7+
 * Supported backends:
   - [JACK](http://jackaudio.org/)
   - [PulseAudio](http://www.freedesktop.org/wiki/Software/PulseAudio/)
   - [ALSA](http://www.alsa-project.org/)
   - [CoreAudio](https://developer.apple.com/library/mac/documentation/MusicAudio/Conceptual/CoreAudioOverview/Introduction/Introduction.html)
   - [WASAPI](https://msdn.microsoft.com/en-us/library/windows/desktop/dd371455%28v=vs.85%29.aspx)
   - Dummy (silence)
 * Exposes both raw devices and shared devices. Raw devices give you the best
   performance but prevent other applications from using them. Shared devices
   are default and usually provide sample rate conversion and format
   conversion.
 * Exposes both device id and friendly name. id you could save in a config file
   because it persists between devices becoming plugged and unplugged, while
   friendly name is suitable for exposing to users.
 * Supports optimal usage of each supported backend. The same API does the
   right thing whether the backend has a fixed buffer size, such as on JACK and
   CoreAudio, or whether it allows directly managing the buffer, such as on
   ALSA, PulseAudio, and WASAPI.
 * C library. Depends only on the respective backend API libraries and libc.
   Does *not* depend on libstdc++, and does *not* have exceptions, run-time type
   information, or [setjmp](http://latentcontent.net/2007/12/05/libpng-worst-api-ever/).
 * Errors are communicated via return codes, not logging to stdio.
 * Supports channel layouts (also known as channel maps), important for
   surround sound applications.
 * Ability to monitor devices and get an event when available devices change.
 * Ability to get an event when the backend is disconnected, for example when
   the JACK server or PulseAudio server shuts down.
 * Detects which input device is default and which output device is default.
 * Ability to connect to multiple backends at once. For example you could have
   an ALSA device open and a JACK device open at the same time.
 * Meticulously checks all return codes and memory allocations and uses
   meaningful error codes.
 * Exposes extra API that is only available on some backends. For example you
   can provide application name and stream names which is used by JACK and
   PulseAudio.

## Synopsis

Complete program to emit a sine wave over the default device using the best
backend:

```c
#include <soundio/soundio.h>

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <math.h>

static const float PI = 3.1415926535f;
static float seconds_offset = 0.0f;
static void write_callback(struct SoundIoOutStream *outstream,
        int frame_count_min, int frame_count_max)
{
    const struct SoundIoChannelLayout *layout = &outstream->layout;
    float float_sample_rate = outstream->sample_rate;
    float seconds_per_frame = 1.0f / float_sample_rate;
    struct SoundIoChannelArea *areas;
    int frames_left = frame_count_max;
    int err;

    while (frames_left > 0) {
        int frame_count = frames_left;

        if ((err = soundio_outstream_begin_write(outstream, &areas, &frame_count))) {
            fprintf(stderr, "%s\n", soundio_strerror(err));
            exit(1);
        }

        if (!frame_count)
            break;

        float pitch = 440.0f;
        float radians_per_second = pitch * 2.0f * PI;
        for (int frame = 0; frame < frame_count; frame += 1) {
            float sample = sinf((seconds_offset + frame * seconds_per_frame) * radians_per_second);
            for (int channel = 0; channel < layout->channel_count; channel += 1) {
                float *ptr = (float*)(areas[channel].ptr + areas[channel].step * frame);
                *ptr = sample;
            }
        }
        seconds_offset = fmodf(seconds_offset +
            seconds_per_frame * frame_count, 1.0f);

        if ((err = soundio_outstream_end_write(outstream))) {
            fprintf(stderr, "%s\n", soundio_strerror(err));
            exit(1);
        }

        frames_left -= frame_count;
    }
}

int main(int argc, char **argv) {
    int err;
    struct SoundIo *soundio = soundio_create();
    if (!soundio) {
        fprintf(stderr, "out of memory\n");
        return 1;
    }

    if ((err = soundio_connect(soundio))) {
        fprintf(stderr, "error connecting: %s", soundio_strerror(err));
        return 1;
    }

    soundio_flush_events(soundio);

    int default_out_device_index = soundio_default_output_device_index(soundio);
    if (default_out_device_index < 0) {
        fprintf(stderr, "no output device found");
        return 1;
    }

    struct SoundIoDevice *device = soundio_get_output_device(soundio, default_out_device_index);
    if (!device) {
        fprintf(stderr, "out of memory");
        return 1;
    }

    fprintf(stderr, "Output device: %s\n", device->name);

    struct SoundIoOutStream *outstream = soundio_outstream_create(device);
    outstream->format = SoundIoFormatFloat32NE;
    outstream->write_callback = write_callback;

    if ((err = soundio_outstream_open(outstream))) {
        fprintf(stderr, "unable to open device: %s", soundio_strerror(err));
        return 1;
    }

    if (outstream->layout_error)
        fprintf(stderr, "unable to set channel layout: %s\n", soundio_strerror(outstream->layout_error));

    if ((err = soundio_outstream_start(outstream))) {
        fprintf(stderr, "unable to start device: %s", soundio_strerror(err));
        return 1;
    }

    for (;;)
        soundio_wait_events(soundio);

    soundio_outstream_destroy(outstream);
    soundio_device_unref(device);
    soundio_destroy(soundio);
    return 0;
}
```

### Backend Priority

When you use `soundio_connect`, libsoundio tries these backends in order.
If unable to connect to that backend, due to the backend not being installed,
or the server not running, or the platform is wrong, the next backend is tried.

 0. JACK
 0. PulseAudio
 0. ALSA (Linux)
 0. CoreAudio (OSX)
 0. WASAPI (Windows)
 0. Dummy

If you don't like this order, you can use `soundio_connect_backend` to
explicitly choose a backend to connect to. You can use `soundio_backend_count`
and `soundio_get_backend` to get the list of available backends.

[API Documentation](http://libsound.io/doc)

### Building

Install the dependencies:

 * cmake
 * ALSA library (optional)
 * libjack2 (optional)
 * libpulseaudio (optional)

```
mkdir build
cd build
cmake ..
cmake --build .
```

Alternatively add using add_subdirectory

```
add_subdirectory(libsoundio)
target_link_libraries(<your-target> PRIVATE libsoundio)
```

### Building for Windows

Currently not supported

### Testing

Not present in this fork, visit [andrewrk/libsoundio](https://github.com/andrewrk/libsoundio) instead

### Building the Documentation

Not present in this fork, visit [andrewrk/libsoundio](https://github.com/andrewrk/libsoundio) instead