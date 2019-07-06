cmake_minimum_required(VERSION 3.5)
project(cpp-mods.grappix)
set(CMAKE_CXX_STANDARD 17)

set(MAIN_FILES
    grappix/buffer.cpp
    grappix/color.cpp
    grappix/distancefield.cpp
    grappix/font.cpp
    grappix/frame.cpp
    grappix/gui/renderable.cpp
    grappix/gui/renderset.cpp
    grappix/render_target.cpp
    grappix/shader.cpp
    grappix/texture.cpp
    grappix/tiles.cpp
    grappix/transform.cpp
    grappix/window.cpp)

set(FREETYPE_FILES
    grappix/freetype-gl/edtaa3func.c
    grappix/freetype-gl/texture-atlas.c
    grappix/freetype-gl/texture-font.c
    grappix/freetype-gl/vector.c)

set(SOURCE_FILES ${MAIN_FILES} ${FREETYPE_FILES})

if(NOT TARGET image)
    add_subdirectory(../image image)
endif()

if(NOT TARGET tween)
    add_subdirectory(../tween tween)
endif()

if(NOT TARGET coreutils)
    add_subdirectory(../coreutils coreutils)
endif()

if(NOT TARGET freetype)
    add_subdirectory(../freetype freetype)
    include_directories(../freetype/include)
endif()

if(NOT TARGET glew)
    add_subdirectory(../glew glew)
endif()

if(NOT TARGET glfw)
    add_subdirectory(../glfw glfw)
endif()

if(APPLE)
  find_library(COCOA_LIBRARY Cocoa)
  find_library(CORE_VIDEO_LIBRARY CoreVideo)
  find_library(IOKIT_LIBRARY IOKit)
  set(EXTRA_LIBS ${EXTRA_LIBS} ${COCOA_LIBRARY} ${CORE_VIDEO_LIBRARY} ${IOKIT_LIBRARY})
endif()

set(OpenGL_GL_PREFERENCE GLVND)

if(EMSCRIPTEN)  

  set(SOURCE_FILES ${SOURCE_FILES} grappix/specific/window_pc.cpp)
  set(EXTRA_LIBS ${EXTRA_LIBS} -lGL)

elseif(RASPBERRY)

  set(VC ${PI_SYS_ROOT}/vc)
  set(SOURCE_FILES ${SOURCE_FILES} grappix/specific/window_pi.cpp grappix/specific/eglutil.cpp)
  set(EXTRA_LIBS ${EXTRA_LIBS} EGL GLESv2 vcos vchiq_arm bcm_host)
  include_directories(${VC}/include ${VC}/include/interface/vcos/pthreads ${VC}/include/interface/vmcs_host/linux)

elseif(WIN32)

  set(SOURCE_FILES ${SOURCE_FILES} grappix/specific/window_pc.cpp)

  find_package(OpenGL REQUIRED)
  set(EXTRA_LIBS ${EXTRA_LIBS} ${OPENGL_LIBRARIES})
  include_directories(${OPENGL_INCLUDE_DIRS})
  set(EXTRA_LIBS ${EXTRA_LIBS} glew glfw)

elseif(UNIX)
  set(SOURCE_FILES ${SOURCE_FILES} grappix/specific/window_pc.cpp)
  find_package(OpenGL REQUIRED)

  set(EXTRA_LIBS ${EXTRA_LIBS} ${OPENGL_LIBRARIES})
  include_directories(${OPENGL_INCLUDE_DIRS})
  set(EXTRA_LIBS ${EXTRA_LIBS} glew glfw)

  if(NOT APPLE)
    if(GLFW_LIBRARY MATCHES "\\.a")
      message("GLFW is static")
      set(EXTRA_LIBS ${EXTRA_LIBS} -lX11 -lXxf86vm -lXrandr -lpthread -lXi -lXinerama -lXcursor)
    else()
      message("GLFW is dynamic")
      set(EXTRA_LIBS ${EXTRA_LIBS} -lX11 -lXxf86vm -lpthread)
    endif()
  endif()

else()
endif()

file(GLOB SHADERS grappix/shaders/*.glsl)
add_custom_target(grappix.shaders SOURCES ${SHADERS})
if(NOT ANDROID)
    add_executable(bin2c bin2c.c)
    set(BIN2C bin2c)
else()
    set(BIN2C ${CMAKE_SOURCE_DIR}/bin/bin2c)
endif()

foreach(SHADER ${SHADERS})
  message(${SHADER})
  get_filename_component(SHADERNAME ${SHADER} NAME)
  string(REPLACE "." "_" NAME2 ${SHADERNAME})
    add_custom_command(OUTPUT ${SHADERNAME}.c
        COMMAND ${BIN2C} -o ${SHADERNAME}.c -n "_shader_${NAME2}" ${SHADER}
        DEPENDS ${SHADER})
    set(SOURCE_FILES ${SOURCE_FILES} ${SHADERNAME}.c)
endforeach()

add_library(grappix ${SOURCE_FILES})
target_link_libraries(grappix PRIVATE coreutils image tween freetype PUBLIC ${EXTRA_LIBS})
target_compile_definitions(grappix PUBLIC GLEW_STATIC WITH_FREETYPE)
target_include_directories(grappix PRIVATE .. grappix/freetype-gl)
