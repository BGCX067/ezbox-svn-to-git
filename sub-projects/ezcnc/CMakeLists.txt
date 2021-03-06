# ezcnc - Copyright (C) 2011 ezbox project

cmake_minimum_required(VERSION 2.6 FATAL_ERROR)

# if CMAKE_BUILD_TYPE is specified use it; otherwise set the default
# build type to "Debug" prior to calling project()
if(DEFINED CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE ${CMAKE_BUILD_TYPE} CACHE STRING "Choose build type")
else(DEFINED CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Debug CACHE STRING "Choose build type")
endif(DEFINED CMAKE_BUILD_TYPE)

project(ezcnc CXX C)

option(ENABLE_FLTK "Build FLTK GUI" ON)
option(ENABLE_FL_TREE "Enable FLTK tree browser widget" ON)
option(ENABLE_NATIVE_FILE_CHOOSER "Enable native file chooser in GUI" ON)

set(EZCNC_MAJOR_VERSION 0)
set(EZCNC_MINOR_VERSION 1)
set(EZCNC_PATCH_VERSION 1)
set(EZCNC_EXTRA_VERSION "" CACHE STRING "ezcnc extra version string")

set(EZCNC_VERSION "${EZCNC_MAJOR_VERSION}.${EZCNC_MINOR_VERSION}")
set(EZCNC_VERSION "${EZCNC_VERSION}.${EZCNC_PATCH_VERSION}${EZCNC_EXTRA_VERSION}")
set(EZCNC_SHORT_LICENSE "GNU General Public License")

if(APPLE)
  set(EZCNC_OS "MacOSX")
elseif(CYGWIN)
  set(EZCNC_OS "Windows")
else(APPLE)
  set(EZCNC_OS "${CMAKE_SYSTEM_NAME}")
endif(APPLE)

include(CheckTypeSize)
include(CheckFunctionExists)
include(CheckIncludeFile)

macro(append_ezcnc_src DIRNAME FILES)
  foreach(FILE ${FILES})
    list(APPEND LIST ${DIRNAME}/${FILE})
  endforeach(FILE)
  set(EZCNC_SRC ${EZCNC_SRC};${LIST} PARENT_SCOPE)
  set(EZCNC_DIRS ${EZCNC_DIRS};${DIRNAME} PARENT_SCOPE)
endmacro(append_ezcnc_src)

macro(set_config_option VARNAME STRING)
  set(${VARNAME} TRUE)
  list(APPEND CONFIG_OPTIONS ${STRING})
  message(STATUS "Found " ${STRING})
endmacro(set_config_option)

add_subdirectory(Common)

# process cmake environment variables so we can append them to the -I
# include commands. This is not recommended (we should only use the
# cache variables) but it is very convenient: otherwise we have to
# remember providing the -D... options to cmake for each new build.
set(ENV_CMAKE_PREFIX_PATH $ENV{CMAKE_PREFIX_PATH})
set(ENV_CMAKE_INCLUDE_PATH $ENV{CMAKE_INCLUDE_PATH})
if(UNIX)
  if(ENV_CMAKE_PREFIX_PATH)
    string(REPLACE ":" ";" ENV_CMAKE_PREFIX_PATH ${ENV_CMAKE_PREFIX_PATH})
  endif(ENV_CMAKE_PREFIX_PATH)
  if(ENV_CMAKE_INCLUDE_PATH)
    string(REPLACE ":" ";" ENV_CMAKE_INCLUDE_PATH ${ENV_CMAKE_INCLUDE_PATH})
  endif(ENV_CMAKE_INCLUDE_PATH)
endif(UNIX)
list(APPEND EXTERNAL_INCLUDES ${CMAKE_INCLUDE_PATH} ${ENV_CMAKE_INCLUDE_PATH})
list(APPEND EXTERNAL_INCLUDES ${CMAKE_PREFIX_PATH} ${ENV_CMAKE_PREFIX_PATH})
foreach(DIR ${CMAKE_PREFIX_PATH} ${ENV_CMAKE_PREFIX_PATH})
  list(APPEND EXTERNAL_INCLUDES ${DIR}/include)
endforeach(DIR)

if(EXTERNAL_INCLUDES)
  list(REMOVE_DUPLICATES EXTERNAL_INCLUDES)
endif(EXTERNAL_INCLUDES)

if(ENABLE_FLTK)
  # first, try to use fltk-config for fltk >= 1.3 (FindFLTK is buggy
  # on Unix, where e.g. xft and xinerama options are not dealt with)
  find_program(FLTK_CONFIG_SCRIPT fltk-config)
  if(FLTK_CONFIG_SCRIPT)
    execute_process(COMMAND ${FLTK_CONFIG_SCRIPT} --api-version
                    OUTPUT_VARIABLE FLTK_VERSION)
    string(STRIP ${FLTK_VERSION} FLTK_VERSION)
    if(FLTK_VERSION GREATER 1.1)
      add_subdirectory(Fltk)
      set_config_option(HAVE_FLTK "Fltk")
      message(STATUS "Using fltk-config script for Fltk " ${FLTK_VERSION})
      execute_process(COMMAND ${FLTK_CONFIG_SCRIPT} --use-gl --use-images --includedir
                      OUTPUT_VARIABLE FLTK_INCLUDE_DIR)
      string(STRIP ${FLTK_INCLUDE_DIR} FLTK_INCLUDE_DIR)
      list(APPEND EXTERNAL_INCLUDES ${FLTK_INCLUDE_DIR} ${FLTK_INCLUDE_DIR}/FL/images
           ${FLTK_INCLUDE_DIR}/jpeg ${FLTK_INCLUDE_DIR}/zlib ${FLTK_INCLUDE_DIR}/png)
      execute_process(COMMAND ${FLTK_CONFIG_SCRIPT} --use-gl --use-images --ldflags
                      OUTPUT_VARIABLE FLTK_LIBRARIES)
      string(STRIP ${FLTK_LIBRARIES} FLTK_LIBRARIES)
      string(REGEX MATCH "fltk[_ ]jpeg" FLTK_JPEG ${FLTK_LIBRARIES})
      string(REGEX MATCH "fltk[_ ]z" FLTK_Z ${FLTK_LIBRARIES})
      string(REGEX MATCH "fltk[_ ]png" FLTK_PNG ${FLTK_LIBRARIES})
    endif(FLTK_VERSION GREATER 1.1)
  endif(FLTK_CONFIG_SCRIPT)
  # then try the built-in FindFLTK module
  if(NOT HAVE_FLTK)
    set(FLTK_SKIP_FORMS TRUE)
    set(FLTK_SKIP_FLUID TRUE)
    find_package(FLTK)
    if(FLTK_FOUND)
      add_subdirectory(Fltk)
      set_config_option(HAVE_FLTK "Fltk")
      list(APPEND EXTERNAL_INCLUDES ${FLTK_INCLUDE_DIR})
      # find fltk jpeg
      find_library(FLTK_JPEG fltk_jpeg)
      if(NOT FLTK_JPEG)
        find_library(FLTK_JPEG fltkjpeg)
      endif(NOT FLTK_JPEG)
      if(FLTK_JPEG)
        list(APPEND EXTERNAL_LIBRARIES ${FLTK_JPEG})
        foreach(DIR ${FLTK_INCLUDE_DIR})
          list(APPEND EXTERNAL_INCLUDES ${DIR}/FL/images ${DIR}/jpeg)
        endforeach(DIR)
      endif(FLTK_JPEG)
      # find fltk zlib
      find_library(FLTK_Z fltk_z)
      if(NOT FLTK_Z)
        find_library(FLTK_Z fltkz)
      endif(NOT FLTK_Z)
      if(FLTK_Z)
        list(APPEND EXTERNAL_LIBRARIES ${FLTK_Z})
        foreach(DIR ${FLTK_INCLUDE_DIR})
          list(APPEND EXTERNAL_INCLUDES ${DIR}/FL/images ${DIR}/zlib)
        endforeach(DIR)
      endif(FLTK_Z)
      # find fltk png
      find_library(FLTK_PNG fltk_png)
      if(NOT FLTK_PNG)
        find_library(FLTK_PNG fltkpng)
      endif(NOT FLTK_PNG)
      if(FLTK_PNG)
        list(APPEND EXTERNAL_LIBRARIES ${FLTK_PNG})
        foreach(DIR ${FLTK_INCLUDE_DIR})
          list(APPEND EXTERNAL_INCLUDES ${DIR}/FL/images ${DIR}/png)
        endforeach(DIR)
      endif(FLTK_PNG)   
    endif(FLTK_FOUND)
  endif(NOT HAVE_FLTK)
  if(HAVE_FLTK)
    if(ENABLE_NATIVE_FILE_CHOOSER)
      if(NOT FLTK_VERSION OR FLTK_VERSION EQUAL 1.1)
        add_subdirectory(contrib/NativeFileChooser)
        include_directories(contrib/NativeFileChooser)
        add_definitions(-DFLTK1)
      endif(NOT FLTK_VERSION OR FLTK_VERSION EQUAL 1.1)
      set_config_option(HAVE_NATIVE_FILE_CHOOSER "NativeFileChooser")
    endif(ENABLE_NATIVE_FILE_CHOOSER)
    if(ENABLE_FL_TREE)
      if(NOT FLTK_VERSION OR FLTK_VERSION EQUAL 1.1)
        add_subdirectory(contrib/Fl_Tree)
        include_directories(contrib/Fl_Tree)
      endif(NOT FLTK_VERSION OR FLTK_VERSION EQUAL 1.1)
      set_config_option(HAVE_FL_TREE "FlTree")
    endif(ENABLE_FL_TREE)
  endif(HAVE_FLTK)
endif(ENABLE_FLTK)

check_include_file(pthread.h HAVE_PTHREAD)
if(HAVE_PTHREAD)
  add_definitions(-DHAVE_PTHREAD_H)
endif (HAVE_PTHREAD)

list(SORT CONFIG_OPTIONS)
set(EZCNC_CONFIG_OPTIONS "")
foreach(OPT ${CONFIG_OPTIONS})
  set(EZCNC_CONFIG_OPTIONS "${EZCNC_CONFIG_OPTIONS} ${OPT}")
endforeach(OPT)

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/Common/EzcncConfig.h.in 
               ${CMAKE_CURRENT_BINARY_DIR}/Common/EzcncConfig.h)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/Common/EzcncVersion.h.in
               ${CMAKE_CURRENT_BINARY_DIR}/Common/EzcncVersion.h)

install(FILES LICENSE README DESTINATION share/doc/ezcnc)
install(PROGRAMS ezcnc DESTINATION bin)
install(DIRECTORY doc/ DESTINATION share/doc/ezcnc)

if(HAVE_FLTK)
  set(LINK_LIBRARIES ${FLTK_LIBRARIES} ${EXTERNAL_LIBRARIES}) 
else(HAVE_FLTK)
  set(LINK_LIBRARIES ${EXTERNAL_LIBRARIES})
endif(HAVE_FLTK)

# we could specify include dirs more selectively, but this is simpler
include_directories(Common Fltk ${EXTERNAL_INCLUDES}
    ${CMAKE_CURRENT_BINARY_DIR}/Common)

# set this for external codes that might include this CMakeList file
set(EZCNC_EXTERNAL_INCLUDE_DIRS ${EXTERNAL_INCLUDES} CACHE 
    STRING "External include directories")
set(EZCNC_EXTERNAL_LIBRARIES ${EXTERNAL_LIBRARIES} CACHE 
    STRING "External libraries")

# group sources for easier navigation in IDEs
foreach(DIR ${EZCNC_DIRS})
  source_group(${DIR} REGULAR_EXPRESSION ${DIR}/.*)
endforeach(DIR)

# binary targets
if(HAVE_FLTK)
  add_executable(ezcnc Fltk/Main.cxx ${EZCNC_SRC})
endif(HAVE_FLTK)
target_link_libraries(ezcnc ${LINK_LIBRARIES})

message("")
message("ezcnc ${EZCNC_VERSION} has been configured for ${EZCNC_OS} with the "
        "following options:" ${EZCNC_CONFIG_OPTIONS})
message("")
message("C compiler: " ${CMAKE_C_COMPILER})
message("C++ compiler: " ${CMAKE_CXX_COMPILER})
if(NOT CMAKE_BUILD_TYPE)
  message("C compiler flags: " ${CMAKE_C_FLAGS})
  message("C++ compiler flags: " ${CMAKE_CXX_FLAGS})
else(NOT CMAKE_BUILD_TYPE)
  message("Build type: " ${CMAKE_BUILD_TYPE})
endif(NOT CMAKE_BUILD_TYPE)
message("Install prefix: " ${CMAKE_INSTALL_PREFIX})
message("")
message("Run 'ccmake ${CMAKE_CURRENT_SOURCE_DIR}' to fine-tune the configuration.")
message("")

