cmake_minimum_required(VERSION 3.16)

project(TextRoom VERSION 0.5.5 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_AUTOUIC_SEARCH_PATHS "${CMAKE_CURRENT_SOURCE_DIR}/ui")

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
	message(STATUS "No build type specified, defaulting to Release")
	set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

include(GNUInstallDirs)

# Unpackaged files live in the user's non-packaged tree on Haiku, not in
# /usr/local (the filesystem below /boot/system is read-only).
if(CMAKE_SYSTEM_NAME STREQUAL "Haiku" AND CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
	set(CMAKE_INSTALL_PREFIX "$ENV{HOME}/config/non-packaged"
		CACHE PATH "Install prefix" FORCE)
endif()

find_package(Qt5 5.12 REQUIRED COMPONENTS Widgets)

set(TEXTROOM_SOURCES
	src/main.cpp
	src/textroom.cpp
	src/optionsdialog.cpp
	src/helpdialog.cpp
	src/searchdialog.cpp
	src/font.cpp
)

set(TEXTROOM_HEADERS
	src/textroom.h
	src/optionsdialog.h
	src/helpdialog.h
	src/searchdialog.h
	src/font.h
)

set(TEXTROOM_FORMS
	ui/textroom.ui
	ui/optionsdialog.ui
	ui/helpdialog.ui
	ui/font.ui
)

add_executable(textroom
	${TEXTROOM_SOURCES}
	${TEXTROOM_HEADERS}
	${TEXTROOM_FORMS}
	resource/textroom.qrc
)

target_link_libraries(textroom PRIVATE Qt5::Widgets)

# Typing sounds are an optional feature. SDL 1.2 and SDL2 are both supported;
# set -DTEXTROOM_SOUND=OFF to build without any SDL dependency.
option(TEXTROOM_SOUND "Enable typing sound effects (requires SDL and SDL_mixer)" ON)

if(TEXTROOM_SOUND)
	# SDL 2 and SDL 1.2 are both usable, but the headers and the libraries must
	# come from the same version, so each pair is probed as a whole.
	find_path(SDL2_INCLUDE_DIR NAMES SDL.h PATH_SUFFIXES SDL2)
	find_library(SDL2_LIBRARY NAMES SDL2)
	find_library(SDL2_MIXER_LIBRARY NAMES SDL2_mixer)

	find_path(SDL1_INCLUDE_DIR NAMES SDL.h PATH_SUFFIXES SDL)
	find_library(SDL1_LIBRARY NAMES SDL)
	find_library(SDL1_MIXER_LIBRARY NAMES SDL_mixer)

	if(SDL2_INCLUDE_DIR AND SDL2_LIBRARY AND SDL2_MIXER_LIBRARY)
		set(TEXTROOM_SDL_INCLUDE_DIR ${SDL2_INCLUDE_DIR})
		set(TEXTROOM_SDL_LIBRARIES ${SDL2_LIBRARY} ${SDL2_MIXER_LIBRARY})
	elseif(SDL1_INCLUDE_DIR AND SDL1_LIBRARY AND SDL1_MIXER_LIBRARY)
		set(TEXTROOM_SDL_INCLUDE_DIR ${SDL1_INCLUDE_DIR})
		set(TEXTROOM_SDL_LIBRARIES ${SDL1_LIBRARY} ${SDL1_MIXER_LIBRARY})
	endif()

	if(TEXTROOM_SDL_LIBRARIES)
		message(STATUS "Typing sounds enabled (${TEXTROOM_SDL_LIBRARIES})")
		target_compile_definitions(textroom PRIVATE HAVE_SDL)
		target_include_directories(textroom PRIVATE ${TEXTROOM_SDL_INCLUDE_DIR})
		target_link_libraries(textroom PRIVATE ${TEXTROOM_SDL_LIBRARIES})
	else()
		message(WARNING
			"SDL/SDL_mixer not found - building without typing sounds. "
			"Install sdl2_mixer_devel (SDL 2) or sdl_mixer_devel (SDL 1.2), "
			"or configure with -DTEXTROOM_SOUND=OFF.")
	endif()
endif()

install(TARGETS textroom RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
install(DIRECTORY resource/sounds/ DESTINATION ${CMAKE_INSTALL_DATADIR}/textroom)
install(DIRECTORY docs/ DESTINATION ${CMAKE_INSTALL_DOCDIR})
