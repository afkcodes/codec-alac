cmake_minimum_required(VERSION 3.16)

if (DEFINED ESP_PLATFORM)
  # idf component
  idf_component_register(
      SRC_DIRS src src/alac/codec
      INCLUDE_DIRS src src/alac/codec
  )

  target_compile_options(${COMPONENT_LIB} PUBLIC -Wno-error -Wno-multichar)

else()

# set the project name
project(codec-alac)

# lots of warnings and all warnings as errors
## add_compile_options(-Wall -Wextra )
set(CMAKE_CXX_STANDARD 17)

option(ALAC_BUILD_ENCODER "Build the encoder (turn off for a decoder-only library)" ON)
option(ALAC_BUILD_SHARED "Build a shared library (default follows BUILD_SHARED_LIBS)" ${BUILD_SHARED_LIBS})

# define source code
file(GLOB_RECURSE SRC_LIST_C CONFIGURE_DEPENDS  "${PROJECT_SOURCE_DIR}/src/*.c" )
file(GLOB_RECURSE SRC_LIST_CPP CONFIGURE_DEPENDS  "${PROJECT_SOURCE_DIR}/src/*.cpp" )

if (NOT ALAC_BUILD_ENCODER)
  list(FILTER SRC_LIST_C EXCLUDE REGEX "/(ag|dp|matrix)_enc\\.c$")
  list(FILTER SRC_LIST_CPP EXCLUDE REGEX "/ALACEncoder\\.cpp$")
endif()

# define as library
if (ALAC_BUILD_SHARED)
  add_library (codec-alac SHARED ${SRC_LIST_C} ${SRC_LIST_CPP})
else()
  add_library (codec-alac STATIC ${SRC_LIST_C} ${SRC_LIST_CPP})
endif()

# define location for header files
target_include_directories(codec-alac PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/src ${CMAKE_CURRENT_SOURCE_DIR}/src/alac/codec )

# disable multi-character character constant warning
target_compile_options(codec-alac PRIVATE -Wno-multichar)

if (ANDROID)
  # Android 15 and later want shared libraries aligned to 16 KB pages
  if (ALAC_BUILD_SHARED)
    target_link_options(codec-alac PRIVATE -Wl,-z,max-page-size=16384)
  endif()
endif()

endif()