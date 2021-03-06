# Copyright (c) 2017 Alain Martin
#
# This file is part of FRUT.
#
# FRUT is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.
#
# FRUT is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with FRUT.  If not, see <http://www.gnu.org/licenses/>.

cmake_minimum_required(VERSION 3.4)


if(NOT DEFINED JUCE_modules_DIRS)
  message(FATAL_ERROR "JUCE_modules_DIRS must be defined")
endif()
list(REMOVE_DUPLICATES JUCE_modules_DIRS)

foreach(modules_dir IN LISTS JUCE_modules_DIRS)
  if (EXISTS "${modules_dir}/juce_core/juce_core.h"
    AND EXISTS "${modules_dir}/juce_data_structures/juce_data_structures.h"
    AND EXISTS "${modules_dir}/juce_events/juce_events.h"
    AND EXISTS "${modules_dir}/juce_graphics/juce_graphics.h"
    AND EXISTS "${modules_dir}/juce_gui_basics/juce_gui_basics.h"
  )
    set(JUCE_modules_DIR ${modules_dir})
    break()
  endif()
endforeach()

if(NOT DEFINED JUCE_modules_DIR)
  message(FATAL_ERROR "Could not find juce_core, juce_data_structures, juce_events, "
    "juce_graphics and juce_gui_basics when searching in the following directories: "
    "${JUCE_modules_DIRS}"
  )
endif()

get_filename_component(JUCE_modules_DIR
  "${JUCE_modules_DIR}" ABSOLUTE
  BASE_DIR "${CMAKE_BINARY_DIR}"
)


project(IconBuilder VERSION 0.2.0)


if(NOT DEFINED CMAKE_CONFIGURATION_TYPES)
  if(NOT DEFINED CMAKE_BUILD_TYPE OR CMAKE_BUILD_TYPE STREQUAL "")
    message(STATUS "Setting CMAKE_BUILD_TYPE to \"Debug\" as it was not specified.")
    set(CMAKE_BUILD_TYPE Debug)
  endif()
endif()


add_executable(IconBuilder
  "${CMAKE_CURRENT_LIST_DIR}/main.cpp"
  "${CMAKE_CURRENT_LIST_DIR}/Source/Project Saving/jucer_ProjectExporter.cpp"
  "${CMAKE_CURRENT_LIST_DIR}/Source/Utility/jucer_FileHelpers.cpp"
)

if(APPLE)
  target_sources(IconBuilder PRIVATE
    "${JUCE_modules_DIR}/juce_core/juce_core.mm"
    "${JUCE_modules_DIR}/juce_data_structures/juce_data_structures.mm"
    "${JUCE_modules_DIR}/juce_events/juce_events.mm"
    "${JUCE_modules_DIR}/juce_graphics/juce_graphics.mm"
    "${JUCE_modules_DIR}/juce_gui_basics/juce_gui_basics.mm"
  )
else()
  target_sources(IconBuilder PRIVATE
    "${JUCE_modules_DIR}/juce_core/juce_core.cpp"
    "${JUCE_modules_DIR}/juce_data_structures/juce_data_structures.cpp"
    "${JUCE_modules_DIR}/juce_events/juce_events.cpp"
    "${JUCE_modules_DIR}/juce_graphics/juce_graphics.cpp"
    "${JUCE_modules_DIR}/juce_gui_basics/juce_gui_basics.cpp"
  )
endif()

target_include_directories(IconBuilder PRIVATE
  "${JUCE_modules_DIR}"
)

target_compile_definitions(IconBuilder PRIVATE
  $<$<CONFIG:Debug>:DEBUG=1>
  $<$<CONFIG:Debug>:_DEBUG=1>
  $<$<NOT:$<CONFIG:Debug>>:NDEBUG=1>
  JUCE_DISABLE_JUCE_VERSION_PRINTING=1
  JUCE_GLOBAL_MODULE_SETTINGS_INCLUDED=1
  JUCE_MODULE_AVAILABLE_juce_core=1
  JUCE_MODULE_AVAILABLE_juce_data_structures=1
  JUCE_MODULE_AVAILABLE_juce_events=1
  JUCE_MODULE_AVAILABLE_juce_graphics=1
  JUCE_MODULE_AVAILABLE_juce_gui_basics=1
  JUCE_STANDALONE_APPLICATION=1
  JUCE_USE_CURL=0
)

set_target_properties(IconBuilder PROPERTIES
  CXX_EXTENSIONS OFF
  CXX_STANDARD 11
  CXX_STANDARD_REQUIRED ON
  OUTPUT_NAME IconBuilder-${PROJECT_VERSION}
)

if(APPLE)
  target_compile_options(IconBuilder PRIVATE "-Wno-deprecated-declarations")

  find_library(Cocoa_framework "Cocoa")
  find_library(IOKit_framework "IOKit")
  find_library(QuartzCore_framework "QuartzCore")

  target_link_libraries(IconBuilder PRIVATE
    ${Cocoa_framework}
    ${IOKit_framework}
    ${QuartzCore_framework}
  )
endif()

if(CMAKE_HOST_SYSTEM_NAME STREQUAL "Linux")
  find_package(PkgConfig REQUIRED)

  function(use_package package)
    pkg_check_modules(${package} "${package}")
    if(NOT ${package}_FOUND)
      message(FATAL_ERROR "pkg-config could not find ${package}")
    endif()
    target_compile_options(IconBuilder PRIVATE ${${package}_CFLAGS})
    target_link_libraries(IconBuilder PRIVATE ${${package}_LIBRARIES})
  endfunction()

  use_package(freetype2)
  use_package(x11)
  use_package(xext)

  target_compile_options(IconBuilder PRIVATE -pthread)
  target_link_libraries(IconBuilder PRIVATE dl pthread)
endif()

if(WIN32 AND NOT MSVC)
  target_compile_options(IconBuilder PRIVATE "-Wa,-mbig-obj")
  target_link_libraries(IconBuilder PRIVATE
    -limm32
    -lshlwapi
    -lversion
    -lwininet
    -lwinmm
    -lws2_32
  )
endif()


if(built_by_Reprojucer)
  install(TARGETS IconBuilder DESTINATION ".")
else()
  install(TARGETS IconBuilder DESTINATION "FRUT/cmake/bin")
endif()
