cmake_minimum_required(VERSION 3.1...4.0)

project(shockolate VERSION 0.7.8)

include(FeatureSummary)

#set_property(GLOBAL PROPERTY FIND_LIBRARY_USE_LIB64_PATHS OFF)
#set_property(GLOBAL PROPERTY FIND_LIBRARY_USE_LIB32_PATHS ON)
set_property(GLOBAL PROPERTY FIND_LIBRARY_USE_LIB32_PATHS OFF)
set_property(GLOBAL PROPERTY FIND_LIBRARY_USE_LIB64_PATHS ON)

# Required for stdbool.h
set(CMAKE_C_STANDARD 99)
# For nullptr in C++
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g ")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -D__STDC_LIMIT_MACROS")

option(ENABLE_EXAMPLES "Enable example applications" OFF)
add_feature_info(ENABLE_EXAMPLES ENABLE_EXAMPLES "Enable example application (can be broken!)")
option(ENABLE_DEBUG_BLIT "Enable debugging blitter" OFF)
add_feature_info(ENABLE_DEBUG_BLIT ENABLE_DEBUG_BLIT "Enable debugging blitter")

set(ENABLE_OPENGL "OFF" CACHE STRING "Enable OpenGL (ON/OFF, default OFF")
option(ENABLE_OPENGL "Enable OpenGL support" OFF)
add_feature_info(ENABLE_OPENGL ENABLE_OPENGL "Enable OpenGL support")

set(ENABLE_SDL2 "ON" CACHE STRING "Enable SDL2 (ON/BUNDLED, default ON)")
set_property(CACHE ENABLE_SDL2 PROPERTY STRINGS "BUNDLED" "ON" )
add_feature_info(ENABLE_SDL2 ENABLE_SDL2 "Enable SDL2 support")

set(ENABLE_SOUND "BUNDLED" CACHE STRING "Enable sound support (requires SDL2_mixer) (ON/BUNDLED/OFF, default BUNDLED)")
set_property(CACHE ENABLE_SOUND PROPERTY STRINGS "ON" "BUNDLED" "OFF")
add_feature_info(ENABLE_SOUND ENABLE_SOUND "Enable sound support (requires SDL2_mixer)")

set(ENABLE_FLUIDSYNTH "OFF" CACHE STRING "Enable FluidSynth MIDI support (ON/BUNDLED/OFF, default OFF)")
set_property(CACHE ENABLE_FLUIDSYNTH PROPERTY STRINGS "ON" "BUNDLED" "OFF")
add_feature_info(ENABLE_FLUIDSYNTH ENABLE_FLUIDSYNTH "Enable FluidSynth MIDI support")

# HAAAAX!!
add_definitions(-DSVGA_SUPPORT)

if(ENABLE_DEBUG_BLIT)
	add_definitions(-DDEBUGGING_BLIT)
endif()

add_compile_options(-fsigned-char -fno-strict-aliasing)

# Find OpenGL
if(ENABLE_OPENGL)
	find_package(OpenGL REQUIRED)
	add_definitions(-DUSE_OPENGL)
	if(WIN32)
		list(APPEND OPENGL_INCLUDE_DIRS ${CMAKE_SOURCE_DIR}/build_ext/built_glew/include)
		list(APPEND OPENGL_LIBRARIES ${CMAKE_SOURCE_DIR}/build_ext/built_glew/lib/libglew32.dll.a winmm)
	endif(WIN32)
endif(ENABLE_OPENGL)

if(ENABLE_SDL2 MATCHES "ON")
	find_package(SDL2 REQUIRED)
	if(SDL2_FOUND)
		message(STATUS "SDL2 found: ${SDL2_INCLUDE_DIRS} ${SDL2_LIBRARIES}")
	endif(SDL2_FOUND)
endif(ENABLE_SDL2 MATCHES "ON")
if(ENABLE_SDL2 MATCHES "BUNDLED")
	set(SDL2_DIR ${CMAKE_SOURCE_DIR}/build_ext/built_sdl)
	find_library(SDL2_LIBRARY SDL2 PATHS ${SDL2_DIR}/lib NO_DEFAULT_PATH)
	find_library(SDL2MAIN_LIBRARY SDL2main PATHS ${SDL2_DIR}/lib NO_DEFAULT_PATH)
	set(SDL2_INCLUDE_DIRS ${SDL2_DIR}/include/SDL2)
	set(SDL2_LIBRARIES "${SDL2MAIN_LIBRARY};${SDL2_LIBRARY}")
endif(ENABLE_SDL2 MATCHES "BUNDLED")

if(ENABLE_SOUND MATCHES "ON")
	# FIXME applies only for *nix systems
	find_package(PkgConfig)
	pkg_check_modules(SDL2_MIXER REQUIRED SDL2_mixer>=2.0.4)
	add_definitions(-DUSE_SDL_MIXER=1)
endif(ENABLE_SOUND MATCHES "ON")
if(ENABLE_SOUND MATCHES "BUNDLED")
	set(SDL2_MIXER_DIR ${CMAKE_SOURCE_DIR}/build_ext/built_sdl_mixer)
	set(SDL2_MIXER_INCLUDE_DIRS ${SDL2_MIXER_DIR}/include/SDL2)
	find_library(SDL2_MIXER_LIBRARY SDL2_mixer PATHS ${SDL2_MIXER_DIR}/lib)
	set(SDL2_MIXER_LIBRARIES ${SDL2_MIXER_LIBRARY})
	add_definitions(-DUSE_SDL_MIXER=1)
endif(ENABLE_SOUND MATCHES "BUNDLED")

if(ENABLE_FLUIDSYNTH MATCHES "ON")
	find_package(PkgConfig)
	pkg_check_modules(FLUIDSYNTH REQUIRED fluidsynth)
	add_definitions("-DUSE_FLUIDSYNTH=1")
endif(ENABLE_FLUIDSYNTH MATCHES "ON")
if(ENABLE_FLUIDSYNTH MATCHES "BUNDLED")
        find_library(FLUIDSYNTH_LIBRARY fluidsynth PATHS ${CMAKE_SOURCE_DIR}/build_ext/fluidsynth-lite/src)
	set(FLUIDSYNTH_INCLUDE_DIRS ${CMAKE_SOURCE_DIR}/build_ext/fluidsynth-lite/include)
	set(FLUIDSYNTH_LIBRARIES ${FLUIDSYNTH_LIBRARY})
	add_definitions("-DUSE_FLUIDSYNTH=1")
endif(ENABLE_FLUIDSYNTH MATCHES "BUNDLED")

include_directories(
	${SDL2_INCLUDE_DIRS}
	${SDL2_MIXER_INCLUDE_DIRS}
	${FLUIDSYNTH_INCLUDE_DIRS}
	${OPENGL_INCLUDE_DIRS}
)

if(NOT WIN32)
  # Find ALSA for Linux native MIDI
  # NOTE: this seems to require having 64-bit dev pacakges installed when building
  #  on 64-bit OS, even when building a 32-bit binary
  find_package(ALSA)
  if(ALSA_FOUND)
      message(STATUS "ALSA found")
      include_directories(${ALSA_INCLUDE_DIRS})
      add_definitions(-DUSE_ALSA=1)
  endif(ALSA_FOUND)
endif(NOT WIN32)

# Generate version based on project version
set(PROJECT_REVERSION_STRING "")
find_package(Git)
if(GIT_FOUND)
	execute_process(COMMAND ${GIT_EXECUTABLE} describe --dirty
		WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
		OUTPUT_VARIABLE git_describe_out
		ERROR_VARIABLE git_describe_error
		RESULT_VARIABLE git_describe_result
		)
	string(REGEX MATCH "[a-z|0-9|.]*-[0-9]*-g([a-z|0-9]*)([-|a-z]*)" git_commit "${git_describe_out}")
	set(git_commit ${CMAKE_MATCH_1})
	set(git_dirty ${CMAKE_MATCH_2})
	set(PROJECT_REVERSION_STRING "-g${git_commit}${git_dirty}")
endif(GIT_FOUND)

message(STATUS "Version is ${PROJECT_VERSION}${PROJECT_REVERSION_STRING}")
configure_file("${CMAKE_SOURCE_DIR}/src/GameSrc/Headers/shockolate_version.h.in"
	"${CMAKE_BINARY_DIR}/src/GameSrc/Headers/shockolate_version.h" )

# Configuration done. Print features
feature_summary(WHAT ENABLED_FEATURES DESCRIPTION "Enabled features:")
feature_summary(WHAT DISABLED_FEATURES DESCRIPTION "Disabled features:")

# Sources configuration
add_subdirectory(src/Libraries/)

set(MAC_SRC
	src/MacSrc/ShockBitmap.c
	src/MacSrc/InitMac.c
	src/MacSrc/Shock.c
	src/MacSrc/Prefs.c
	src/MacSrc/MacTune.c
	src/MacSrc/SDLSound.c
	src/MacSrc/Modding.c
	src/MacSrc/OpenGL.cc
	src/MacSrc/Xmi.c
	src/MusicSrc/MusicDevice.c
)

set(GAME_SRC
	src/GameSrc/ai.c
	src/GameSrc/airupt.c
	src/GameSrc/amap.c
	src/GameSrc/amaploop.c
	src/GameSrc/ammomfd.c
	src/GameSrc/anim.c
	src/GameSrc/archiveformat.c
	src/GameSrc/audiolog.c
	src/GameSrc/automap.c
	src/GameSrc/bark.c
	src/GameSrc/biohelp.c
	src/GameSrc/cardmfd.c
	src/GameSrc/citres.c
	src/GameSrc/combat.c
	src/GameSrc/cone.c
	src/GameSrc/criterr.c
	src/GameSrc/cyber.c
	src/GameSrc/cybermfd.c
	src/GameSrc/cybmem.c
	src/GameSrc/cybrnd.c
	src/GameSrc/cutsloop.c
	src/GameSrc/damage.c
	src/GameSrc/digifx.c
	src/GameSrc/drugs.c
	src/GameSrc/effect.c
	src/GameSrc/email.c
	src/GameSrc/faceobj.c
	src/GameSrc/fixtrmfd.c
	src/GameSrc/frcamera.c
	src/GameSrc/frclip.c
	src/GameSrc/frcompil.c
	src/GameSrc/frmain.c
	src/GameSrc/frobj.c
	src/GameSrc/froslew.c
	src/GameSrc/frpipe.c
	src/GameSrc/frpts.c
	src/GameSrc/frsetup.c
	src/GameSrc/frtables.c
	src/GameSrc/frterr.c
	src/GameSrc/frutil.c
	src/GameSrc/FrUtils.c
	src/GameSrc/fullamap.c
	src/GameSrc/fullscrn.c
	src/GameSrc/gameloop.c
	src/GameSrc/gameobj.c
	src/GameSrc/gamesort.c
	src/GameSrc/gamestrn.c
	src/GameSrc/gamesys.c
	src/GameSrc/gametime.c
	src/GameSrc/gamewrap.c
	src/GameSrc/gearmfd.c
	src/GameSrc/gr2ss.c
	src/GameSrc/grenades.c
	src/GameSrc/hand.c
	src/GameSrc/hflip.c
	src/GameSrc/hkeyfunc.c
	src/GameSrc/hud.c
	src/GameSrc/hudobj.c
	src/GameSrc/init.c
	src/GameSrc/input.c
	src/GameSrc/invent.c
	src/GameSrc/leanmetr.c
	src/GameSrc/mainloop.c
	src/GameSrc/map.c
	src/GameSrc/mfdfunc.c
	src/GameSrc/mfdgadg.c
	src/GameSrc/mfdgames.c
	src/GameSrc/mfdgump.c
	src/GameSrc/mfdpanel.c
	src/GameSrc/minimax.c
	src/GameSrc/mlimbs.c
	src/GameSrc/movekeys.c
	src/GameSrc/musicai.c
	src/GameSrc/newai.c
	src/GameSrc/newmfd.c
	src/GameSrc/objapp.c
	src/GameSrc/objects.c
	src/GameSrc/objload.c
	src/GameSrc/objprop.c
	src/GameSrc/objsim.c
	src/GameSrc/objuse.c
	src/GameSrc/olh.c
	src/GameSrc/olhscan.c
	src/GameSrc/palfx.c
	src/GameSrc/pathfind.c
	src/GameSrc/physics.c
	src/GameSrc/player.c
	src/GameSrc/plotware.c
	src/GameSrc/popups.c
	src/GameSrc/render.c
	src/GameSrc/rendtool.c
	src/GameSrc/saveload.c
	src/GameSrc/schedule.c
	src/GameSrc/screen.c
	src/GameSrc/setup.c
	src/GameSrc/shodan.c
	src/GameSrc/sideicon.c
	src/GameSrc/sndcall.c
	src/GameSrc/star.c
	src/GameSrc/statics.c
	src/GameSrc/status.c
	src/GameSrc/target.c
	src/GameSrc/textmaps.c
	src/GameSrc/tickcount.c
	src/GameSrc/tfdirect.c
	src/GameSrc/tfutil.c
	src/GameSrc/tools.c
	src/GameSrc/trigger.c
	src/GameSrc/view360.c
	src/GameSrc/viewhelp.c
	src/GameSrc/vitals.c
	src/GameSrc/vmail.c
	src/GameSrc/wares.c
	src/GameSrc/weapons.c
	src/GameSrc/wrapper.c
	src/GameSrc/gamerend.c
	src/GameSrc/mouselook.c
)

include_directories(
	BEFORE
	src/Libraries/2D/Source
	src/Libraries/3D/Source
	src/Libraries/AFILE/Source
	src/Libraries/DSTRUCT/Source
	src/Libraries/EDMS/Source
	src/Libraries/FIXPP/Source
	src/Libraries/INPUT/Source
	src/Libraries/PALETTE/Source
	src/Libraries/RES/Source
	src/Libraries/UI/Source
	src/Libraries/RND/Source
	src/Libraries/VOX/Source
	src/Libraries/FIX/Source
	src/Libraries/SND/Source
	src/Libraries/H
	src/Libraries/LG/Source
	src/Libraries/LG/Source/LOG/src
	src/Libraries/adlmidi/include
	src/GameSrc/Headers
	src/MacSrc
	src/MusicSrc
	${CMAKE_BINARY_DIR}/src/GameSrc/Headers
)

if(ENABLE_EXAMPLES)

add_executable(playmov
	src/Libraries/AFILE/Tests/playmov.c
)
add_executable(movinfo
	src/Libraries/AFILE/Tests/movinfo.c
)

target_link_libraries(playmov
	AFILE_LIB
	FIX_LIB
	${SDL2_LIBRARIES}
	${SDL2_MIXER_LIBRARIES}
)

target_link_libraries(movinfo
	AFILE_LIB
	FIX_LIB
)

add_executable(TestSimpleMain
	src/Libraries/2D/TestSource/SimpleMain.c
)

target_link_libraries(TestSimpleMain
	2D_LIB
	GR_LIB
	3D_LIB
	RES_LIB
	FIX_LIB
	LG_LIB
	${SDL2_LIBRARIES}
)

add_executable(BoxTest
	src/Libraries/3D/Tests/BoxTest.c
)

target_link_libraries(BoxTest
	2D_LIB
	GR_LIB
	3D_LIB
	RES_LIB
	FIX_LIB
	LG_LIB
	${SDL2_LIBRARIES}
)

add_executable(BitmapTest
	src/Libraries/3D/Tests/BitmapTest.c
)

target_link_libraries(BitmapTest
	2D_LIB
	GR_LIB
	3D_LIB
	RES_LIB
	FIX_LIB
	LG_LIB
	${SDL2_LIBRARIES}
)

add_executable(FixTest
	src/Libraries/FIX/Tests/FixTest/fixtest.c
)

target_link_libraries(FixTest
	FIX_LIB
	LG_LIB
)

endif()

# Include magic header file, set struct packing size
add_definitions(
	-include precompiled.h
)

add_executable(systemshock
	${MAC_SRC}
)

add_library(GAME_LIB ${GAME_SRC})

# MINGW additional linker options
if(MINGW)
	set(WINDOWS_LIBRARIES "mingw32 -mwindows")
endif(MINGW)

target_link_libraries(systemshock
	${WINDOWS_LIBRARIES} # Set it before any linker options! Beware WinMain@16 error!!
	GAME_LIB
	UI_LIB
	2D_LIB
	LG_LIB
	GR_LIB
	3D_LIB
	RND_LIB
	AFILE_LIB
	DSTRUCT_LIB
	FIX_LIB
	INPUT_LIB
	PALETTE_LIB
	RES_LIB
#	SND_LIB
	VOX_LIB
	EDMS_LIB
	FIXPP_LIB
	ADLMIDI_LIB
	${SDL2_LIBRARIES}
	${SDL2_MIXER_LIBRARIES}
	${FLUIDSYNTH_LIBRARIES}
	${OPENGL_LIBRARIES}
	${ALSA_LIBRARIES}
)

# Turn on address sanitizing if wanted
set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/externals/sanitizers/cmake" ${CMAKE_MODULE_PATH})
find_package(Sanitizers)
add_sanitizers(systemshock)
