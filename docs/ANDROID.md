# Building and using codec-alac on Android

This library is plain C and C++17 with no dependencies, so it builds with the
Android NDK's CMake toolchain file. These notes cover a command line build, use
from a Gradle module, a small JNI example, and the points that cause crashes if
they are missed.

For playback you usually only need the decoder. Build with
`-DALAC_BUILD_ENCODER=OFF`: the shared library for arm64-v8a is about 20 KB
after stripping.

## CMake options

| Option | Default | Meaning |
|---|---|---|
| `ALAC_BUILD_ENCODER` | ON | Build the encoder. OFF builds the decoder only. |
| `ALAC_BUILD_SHARED` | follows `BUILD_SHARED_LIBS` (OFF) | ON builds a shared library, OFF a static one. |

On Android, shared builds are linked with `-Wl,-z,max-page-size=16384`
automatically (see "16 KB pages" below).

## Building from the command line

You need the NDK and a CMake and Ninja. The Android SDK ships both under
`Sdk/cmake/<version>/bin`. Replace the paths with yours.

```sh
NDK=$HOME/Android/Sdk/ndk/29.0.14206865
CMAKE=$HOME/Android/Sdk/cmake/3.22.1/bin/cmake
NINJA=$HOME/Android/Sdk/cmake/3.22.1/bin/ninja

for ABI in arm64-v8a armeabi-v7a x86_64; do
  $CMAKE -S . -B build/$ABI -G Ninja \
    -DCMAKE_MAKE_PROGRAM=$NINJA \
    -DCMAKE_TOOLCHAIN_FILE=$NDK/build/cmake/android.toolchain.cmake \
    -DANDROID_ABI=$ABI \
    -DANDROID_PLATFORM=android-26 \
    -DCMAKE_BUILD_TYPE=Release \
    -DALAC_BUILD_SHARED=ON \
    -DALAC_BUILD_ENCODER=OFF
  $CMAKE --build build/$ABI
done
```

The result is `build/<abi>/libcodec-alac.so`. For a static library use
`-DALAC_BUILD_SHARED=OFF`, which gives `libcodec-alac.a`. Add `x86` to the
list if you still ship 32-bit x86.

Sizes measured with NDK 29.0.14206865, `android-26`, Release, decoder only,
before stripping (Gradle strips them when it packages the app):

| ABI | libcodec-alac.so | stripped | libcodec-alac.a |
|---|---|---|---|
| arm64-v8a | 88,880 bytes | 19,840 bytes | 141,268 bytes |
| armeabi-v7a | 65,048 bytes | 13,656 bytes | 92,588 bytes |
| x86_64 | 96,112 bytes | 23,664 bytes | 148,228 bytes |

## Using it from a Gradle module

Put this repository in your project, as a git submodule or a copy, or fetch it
with CMake. Pin it to a commit so the build does not change under you.

`app/src/main/cpp/CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.22)
project(alacjni LANGUAGES C CXX)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)  # static objects go into a shared library
set(ALAC_BUILD_ENCODER OFF CACHE BOOL "" FORCE)
set(ALAC_BUILD_SHARED OFF CACHE BOOL "" FORCE)  # link it into our own .so

include(FetchContent)
FetchContent_Declare(codec_alac
  GIT_REPOSITORY https://github.com/afkcodes/codec-alac.git
  GIT_TAG        ede1e87796c4eb4fc71522b18c3e79dee796c55c)
FetchContent_MakeAvailable(codec_alac)

add_library(alacjni SHARED alac_jni.cpp)
target_link_libraries(alacjni PRIVATE codec-alac log)
if (ANDROID)
  # needed for 16 KB page devices; only your own shared library needs it
  target_link_options(alacjni PRIVATE -Wl,-z,max-page-size=16384)
endif()
```

With a submodule at `third_party/codec-alac`, replace the `FetchContent`
lines with `add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/../../../../third_party/codec-alac codec-alac)`.

The library is built as a static library here and linked into `libalacjni.so`,
so only your own library needs the page-size flag.

`app/build.gradle.kts`:

```kotlin
android {
    ndkVersion = "29.0.14206865"
    defaultConfig {
        minSdk = 26
        ndk { abiFilters += listOf("arm64-v8a", "armeabi-v7a", "x86_64") }
        externalNativeBuild {
            cmake { arguments += listOf("-DANDROID_STL=c++_static") }
        }
    }
    externalNativeBuild {
        cmake {
            path = file("src/main/cpp/CMakeLists.txt")
            version = "3.22.1"
        }
    }
}
```

FetchContent needs network access at configure time. For offline or
reproducible builds use the submodule.

## JNI wrapper, in outline

This is a sketch, not a complete decoder. Error handling is cut down.

```cpp
#include <jni.h>
#include <vector>
#include "ALAC.h"   // pulls in ALACDecoder.h and ALACBitUtilities.h

struct Dec {
  ALACDecoder d;
  uint32_t frames = 0, channels = 0, bytesPerSample = 0;
  std::vector<uint8_t> in;   // packet plus padding
};

extern "C" JNIEXPORT jlong JNICALL
Java_com_example_AlacDecoder_nativeInit(JNIEnv* env, jobject, jbyteArray cookie) {
  jsize n = env->GetArrayLength(cookie);
  std::vector<uint8_t> c(n);
  env->GetByteArrayRegion(cookie, 0, n, (jbyte*)c.data());
  auto* h = new Dec;
  if (h->d.Init(c.data(), n) != 0) { delete h; return 0; }   // 0 is ALAC_noErr
  h->frames = h->d.mConfig.frameLength;
  h->channels = h->d.mConfig.numChannels;
  h->bytesPerSample = h->d.mConfig.bitDepth == 16 ? 2
                    : h->d.mConfig.bitDepth == 32 ? 4 : 3;   // 20 and 24 are packed
  return (jlong)h;
}

// Returns the number of sample frames written to out, or a negative error.
extern "C" JNIEXPORT jint JNICALL
Java_com_example_AlacDecoder_nativeDecode(JNIEnv* env, jobject, jlong handle,
    jobject packet, jint size, jobject out) {
  auto* h = (Dec*)handle;
  size_t pad = 2 * h->frames * h->channels * h->bytesPerSample + 64;
  h->in.assign((size_t)size + pad, 0);                    // zero padding after the packet
  memcpy(h->in.data(), env->GetDirectBufferAddress(packet), size);
  if ((size_t)env->GetDirectBufferCapacity(out) < h->frames * h->channels * h->bytesPerSample)
    return -1;
  BitBuffer bits;
  BitBufferInit(&bits, h->in.data(), size);
  uint32_t got = 0;
  int32_t st = h->d.Decode(&bits, (uint8_t*)env->GetDirectBufferAddress(out),
                           h->frames, h->channels, &got);
  return st == 0 ? (jint)got : -2;
}
```

`Decode` takes the frame length from the cookie as the number of samples to
expect. A packet that holds fewer (usually the last one) says so in its own
header, and `got` returns the real count. The output size in bytes is
`got * channels * bytesPerSample`.

Call `ALACDecoder` from one thread at a time; it keeps scratch buffers. Make a
new decoder after a seek only if the format changed. A seek does not need one,
because each ALAC packet decodes on its own.

## Safety notes

**The cookie.** `Init` takes the 24-byte `ALACSpecificConfig` from the
`alac` box of the MP4 sample entry (big-endian fields: frame length, bit depth,
tuning parameters, channel count, max frame bytes, bit rate, sample rate). In
Media3 this is `Format.initializationData[0]`. `Init` also accepts the same
data wrapped in the 12-byte `alac` atom header, with or without a leading
`frma` atom. It refuses a cookie shorter than 24 bytes, a frame length of 0 or
over 16384, and a channel count of 0 or over 8, and returns a non-zero status.
Always check that status. Do not call `Decode` if `Init` failed.

**Pad every input packet.** Apple's bit reader does not check for the end of
the buffer. A damaged or truncated packet can make it read past the end of the
data. Copy each packet into a buffer that has zeros after it, at least
`2 * frameLength * channels * bytesPerSample + 64` bytes of them, as in the
example. With the padding, a bad packet decodes to noise or returns an error
instead of reading beyond the buffer. Pass the real packet size to `BitBufferInit`.

**Partial frames.** `Decode` returns an error for a packet that claims more
samples than the frame length. Treat any non-zero status as a failed packet:
output silence for that frame and carry on, or stop, but do not use the output
buffer.

## Output formats

`Decode` writes interleaved little-endian samples (on Android's CPUs) in the
file's own bit depth:

| Bit depth in the file | Bytes per sample in the output | Android PCM encoding |
|---|---|---|
| 16 | 2 | `ENCODING_PCM_16BIT` |
| 20 | 3, packed | no direct match; expand to a 32-bit container |
| 24 | 3, packed | `ENCODING_PCM_24BIT` (packed), or expand to a 32-bit container |
| 32 | 4 | `ENCODING_PCM_32BIT` |

Media3's `C.ENCODING_PCM_24BIT` is packed 3-byte samples, so decoder output
can be passed on directly. If the audio sink needs 32-bit containers, move each
24-bit sample into the top three bytes of a 32-bit word (shift left by 8) and
use `ENCODING_PCM_32BIT`; do not use float unless you scale it yourself.

## 16 KB pages

Android 15 and later can run on devices with 16 KB memory pages, and Google Play
requires apps that target Android 15 or later to support them. A shared library
must have its ELF load segments aligned to 16 KB (0x4000). This library adds
`-Wl,-z,max-page-size=16384` itself when it builds a shared library for
Android. For your own `.so` add the same flag, as shown above. NDK r28 and
later align by default, but older NDKs do not, and the flag does no harm.

To check a built library:

```sh
llvm-readelf -lW libcodec-alac.so | grep LOAD
```

The last column (`Align`) of every `LOAD` line should read `0x4000` or
larger. `llvm-readelf` is in the NDK under
`toolchains/llvm/prebuilt/<host>/bin`. All three decoder-only libraries from the
commands above show `0x4000`.
