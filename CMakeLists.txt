##
## This file is part of the PulseView project.
##
## Copyright (C) 2014 Marcus Comstedt <marcus@mc.pp.se>
## Copyright (C) 2018 Gerhard Sittig <gerhard.sittig@gmx.net>
## Copyright (C) 2020 Uwe Hermann <uwe@hermann-uwe.de>
##
## This program is free software: you can redistribute it and/or modify
## it under the terms of the GNU General Public License as published by
## the Free Software Foundation, either version 2 of the License, or
## (at your option) any later version.
##
## This program is distributed in the hope that it will be useful,
## but WITHOUT ANY WARRANTY; without even the implied warranty of
## MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
## GNU General Public License for more details.
##
## You should have received a copy of the GNU General Public License
## along with this program.  If not, see <http://www.gnu.org/licenses/>.
##

cmake_minimum_required(VERSION 3.12)

project(pulseview VERSION 0.5.0 LANGUAGES CXX)

include(GNUInstallDirs)

set(PV_MANUAL_VERSION ${PROJECT_VERSION})

find_package(Git QUIET)
if(Git_FOUND)
	execute_process(COMMAND ${GIT_EXECUTABLE} rev-parse --short HEAD
		WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} ERROR_QUIET
		OUTPUT_VARIABLE GITHASH OUTPUT_STRIP_TRAILING_WHITESPACE)
	execute_process(COMMAND
		${GIT_EXECUTABLE} name-rev --tags --name-only ${GITHASH}
		WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} ERROR_QUIET
		OUTPUT_VARIABLE GITTAG OUTPUT_STRIP_TRAILING_WHITESPACE)
	if((DEFINED GITTAG) AND ("${GITTAG}" STREQUAL "undefined"))
		string(APPEND PROJECT_VERSION "-git-${GITHASH}")
		set(PV_MANUAL_VERSION "unstable")
	endif()
endif()

option(BUILD_SHARED_LIBS "Shared (non-static) build" TRUE)

option(ENABLE_STACKTRACE "Enable stack trace when crashing" FALSE)
option(ENABLE_FLOW "Build with libsigrokflow" FALSE)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
	set_property(CACHE CMAKE_BUILD_TYPE PROPERTY VALUE RelWithDebInfo)
endif()

set_property(GLOBAL PROPERTY USE_FOLDERS TRUE)

set(ENABLE_SIGNALS TRUE)
if(WIN32)
	set(ENABLE_SIGNALS FALSE)
endif()

find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBGLIB REQUIRED IMPORTED_TARGET "glib-2.0>=2.28.0")
pkg_check_modules(LIBGLIBMM REQUIRED IMPORTED_TARGET "glibmm-2.4>=2.28.0")

find_package(sigrokcxx 6.0.0 REQUIRED)
find_package(sigrokdecode 6.0.0)
option(ENABLE_DECODE "Build with libsigrokdecode" ${sigrokdecode_FOUND})
message(STATUS "libsigrokcxx: ${sigrokcxx_VERSION}")
message(STATUS "libsigrokdecode: ${sigrokdecode_VERSION}")

find_package(Qt5 5.3 COMPONENTS Core Gui LinguistTools Widgets Svg REQUIRED)
message(STATUS "Qt: ${Qt5_VERSION}")

# Workaround for an MXE issue: https://github.com/mxe/mxe/issues/1642.
if(MINGW AND NOT DEFINED ENV{MSYSTEM})
	pkg_check_modules(QT5ALL QUIET REQUIRED Qt5Widgets Qt5Gui Qt5Svg)
endif()

set(MIN_BOOST_VER 1.55)
if(ENABLE_STACKTRACE)
	set(MIN_BOOST_VER 1.71.0)
	set(_Boost_STACKTRACE_BACKTRACE_HEADERS "boost/stacktrace.hpp")
	set(_Boost_STACKTRACE_BASIC_HEADERS     "boost/stacktrace.hpp")
	include(FindBacktrace)
	set(ST stacktrace_basic)
	if (Backtrace_FOUND)
		set(ST stacktrace_backtrace)
	endif()
endif()
find_package(Boost ${MIN_BOOST_VER} OPTIONAL_COMPONENTS unit_test_framework
	REQUIRED COMPONENTS filesystem serialization system ${ST})

find_package(Threads REQUIRED)

include(CheckCXXSourceRuns)
function(memaccess_check_unaligned_le _var)
	check_cxx_source_runs("#include <stdint.h>
	int main() { int i; union { uint64_t u64; uint8_t u8[16]; } d;
	uint64_t v; for (i = 0; i < 16; i++) d.u8[i] = i;
	v = *(uint64_t *)(d.u8 + 1); if (v != 0x0807060504030201ULL) return 1;
	return 0; }" ${_var})
endfunction()
if(NOT CMAKE_CROSSCOMPILING)
	memaccess_check_unaligned_le(HAVE_UNALIGNED_LITTLE_ENDIAN_ACCESS)
endif()

function(pv_files cond dirs filebases)
	if(NOT ${cond})
		return()
	endif()
	foreach(f ${filebases})
		target_sources(lib${PROJECT_NAME} PRIVATE ${dirs}${f}.cpp)
		target_sources(lib${PROJECT_NAME} PRIVATE ${dirs}${f}.hpp)
	endforeach()
endfunction()

add_library(lib${PROJECT_NAME} STATIC "")

pv_files(TRUE "pv/" "application;devicemanager;globalsettings;logging;mainwindow;metadata_obj;session;storesession;util")
pv_files(TRUE "pv/binding/" "binding;device;inputoutput")
pv_files(TRUE "pv/data/" "analog;analogsegment;logic;logicsegment;segment;signalbase;signaldata")
pv_files(TRUE "pv/devices/" "device;file;hardwaredevice;inputfile;sessionfile")
pv_files(TRUE "pv/dialogs/" "connect;inputoutputoptions;settings;storeprogress")
pv_files(TRUE "pv/popups/" "channels;deviceoptions")
pv_files(TRUE "pv/prop/" "bool;double;enum;int;property;string")
pv_files(TRUE "pv/subwindows/" "subwindowbase")
pv_files(TRUE "pv/toolbars/" "mainbar")
pv_files(TRUE "pv/views/" "viewbase")
pv_files(TRUE "pv/views/trace/" "analogsignal;cursor;cursorpair;flag;header;logicsignal;marginwidget;ruler;signal;standardbar;timeitem;timemarker;trace;tracegroup;tracepalette;tracetreeitem;tracetreeitemowner;triggermarker;view;viewitem;viewitemowner;viewitempaintparams;viewport;viewwidget")
pv_files(TRUE "pv/widgets/" "colorbutton;colorpopup;devicetoolbutton;exportmenu;flowlayout;importmenu;popup;popuptoolbutton;sweeptimingwidget;timestampspinbox;wellarray")

pv_files(ENABLE_SIGNALS "" "signalhandler")

pv_files(ENABLE_DECODE "pv/binding/" "decoder")
pv_files(ENABLE_DECODE "pv/data/" "decodesignal")
pv_files(ENABLE_DECODE "pv/data/decode/" "annotation;decoder;row;rowdata")
pv_files(ENABLE_DECODE "pv/subwindows/decoder_selector/" "subwindow")
pv_files(ENABLE_DECODE "pv/views/decoder_binary/" "view;QHexView")
pv_files(ENABLE_DECODE "pv/views/tabular_decoder/" "view")
pv_files(ENABLE_DECODE "pv/views/trace/" "decodetrace")
pv_files(ENABLE_DECODE "pv/widgets/" "decodergroupbox;decodermenu")

target_sources(lib${PROJECT_NAME} PRIVATE extdef.h pv/strnatcmp.hpp
	pv/views/trace/viewitemiterator.hpp)

if(ENABLE_DECODE)
	target_sources(lib${PROJECT_NAME} PRIVATE
		pv/views/tabular_decoder/model.cpp
		pv/subwindows/decoder_selector/item.cpp
		pv/subwindows/decoder_selector/model.cpp)
endif()

set_target_properties(lib${PROJECT_NAME} PROPERTIES AUTOMOC TRUE AUTORCC TRUE)

target_compile_features(lib${PROJECT_NAME} PUBLIC cxx_std_11)

target_include_directories(lib${PROJECT_NAME} PUBLIC
	$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
	$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)

target_link_libraries(lib${PROJECT_NAME} PUBLIC Qt5::Gui Qt5::Widgets Qt5::Svg
	Boost::system Boost::filesystem Boost::serialization Threads::Threads
	$<$<BOOL:${ENABLE_STACKTRACE}>:Boost::${ST}>
	$<$<BOOL:${ENABLE_STACKTRACE}>:${CMAKE_DL_LIBS}>
	$<$<BOOL:${Backtrace_FOUND}>:${Backtrace_LIBRARIES}>
	sigrok::sigrokcxx $<IF:$<BOOL:${sigrokdecode_FOUND}>,sigrok::sigrokdecode,>)

if(${CMAKE_SYSTEM_PROCESSOR} MATCHES "sparc")
	target_link_libraries(lib${PROJECT_NAME} PUBLIC atomic)
endif()

# if(WIN32) # TODO
if(WIN32 AND NOT BUILD_SHARED_LIBS)
	target_compile_definitions(lib${PROJECT_NAME} PUBLIC -DQT_STATICPLUGIN)
	target_link_libraries(lib${PROJECT_NAME} PUBLIC
		Qt5::QSvgPlugin Qt5::QWindowsIntegrationPlugin ${QT5ALL_LDFLAGS}
		$<$<VERSION_LESS:${Qt5Gui_VERSION},5.8.0>:-lQt5PlatformSupport>)
endif()

target_compile_options(lib${PROJECT_NAME} PUBLIC
	$<IF:$<C_COMPILER_ID:MSVC>,/W4,-Wall -Wextra>
	$<$<BOOL:${ENABLE_STACKTRACE}>:-no-pie -fno-pie>)

# TODO: Requires cmake >= 3.13.
target_link_options(lib${PROJECT_NAME} PUBLIC
	$<$<BOOL:${ENABLE_STACKTRACE}>:-no-pie -fno-pie>)

target_compile_definitions(lib${PROJECT_NAME} PUBLIC
	-DQT_NO_KEYWORDS -D__STDC_LIMIT_MACROS -DBOOST_MATH_DISABLE_FLOAT128
	$<$<BOOL:${ENABLE_DECODE}>:ENABLE_DECODE>
	$<$<BOOL:${ENABLE_SIGNALS}>:ENABLE_SIGNALS>
	$<$<BOOL:${ENABLE_STACKTRACE}>:ENABLE_STACKTRACE>
	$<$<BOOL:${Backtrace_FOUND}>:BOOST_STACKTRACE_USE_BACKTRACE>)

function(pkgconfig_lib p)
	if(${p}_FOUND AND BUILD_SHARED_LIBS)
		target_link_libraries(lib${PROJECT_NAME} PUBLIC PkgConfig::${p})
	elseif(${p}_FOUND)
		target_include_directories(lib${PROJECT_NAME} PUBLIC ${${p}_STATIC_INCLUDE_DIRS})
		target_link_libraries(lib${PROJECT_NAME} PUBLIC ${${p}_STATIC_LDFLAGS})
	endif()
endfunction()

pkgconfig_lib(LIBGLIB)
pkgconfig_lib(LIBGLIBMM)

set(TS_FILES l10n/de.ts)
set_source_files_properties(${TS_FILES} PROPERTIES
	OUTPUT_LOCATION "${CMAKE_CURRENT_BINARY_DIR}/l10n")
if(NOT CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR)
	configure_file(translations.qrc ${CMAKE_CURRENT_BINARY_DIR} COPYONLY)
endif()
qt5_add_translation(QM_FILES ${TS_FILES})
add_executable(${PROJECT_NAME} WIN32 main.cpp pulseview.qrc pulseviewico.rc
	${CMAKE_CURRENT_BINARY_DIR}/translations.qrc)
set_target_properties(${PROJECT_NAME} PROPERTIES AUTOMOC TRUE AUTORCC TRUE)
target_link_libraries(${PROJECT_NAME} PRIVATE lib${PROJECT_NAME})

install(TARGETS ${PROJECT_NAME} RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

install(FILES doc/pulseview.1 DESTINATION ${CMAKE_INSTALL_MANDIR}/man1)
install(FILES contrib/org.sigrok.PulseView.desktop DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/applications)
install(FILES contrib/org.sigrok.PulseView.appdata.xml DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/metainfo)
install(FILES icons/pulseview.png DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/48x48/apps)
install(FILES icons/pulseview.svg DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/scalable/apps)

message(STATUS "CMake: ${CMAKE_COMMAND} ${CMAKE_VERSION}")
message(STATUS "Building ${PROJECT_NAME} ${PROJECT_VERSION} for ${CMAKE_SYSTEM} ${CMAKE_SYSTEM_PROCESSOR}")

include(CTest)
if(Boost_UNIT_TEST_FRAMEWORK_FOUND AND BUILD_TESTING)
	enable_testing()
	add_executable(unittests test/test.cpp test/test.hpp test/util.cpp
		test/view/ruler.cpp test/data/analogsegment.cpp
		test/data/logicsegment.cpp test/data/segment.cpp
		$<$<BOOL:${ENABLE_DECODE}>:test/data/decoderstack.cpp>)
	target_compile_definitions(unittests PRIVATE
		$<$<BOOL:${BUILD_SHARED_LIBS}>:BOOST_TEST_DYN_LINK>)
	target_link_libraries(unittests PRIVATE
		lib${PROJECT_NAME} Boost::unit_test_framework)
	add_test(NAME unittests COMMAND unittests)
endif()

set(CPACK_SOURCE_PACKAGE_FILE_NAME "${PROJECT_NAME}-${PROJECT_VERSION}")
set(CPACK_SOURCE_IGNORE_FILES ${CMAKE_CURRENT_BINARY_DIR} ".git*")
set(CPACK_SOURCE_GENERATOR "TGZ")
include(CPack)

configure_file(config.h.in config.h @ONLY)
configure_file(contrib/${PROJECT_NAME}_cross.nsi.in contrib/${PROJECT_NAME}_cross.nsi @ONLY)

# # TODO: Manual:
# find_program(ASCIIDOCTOR_EXECUTABLE NAMES asciidoctor)
# find_program(ASCIIDOCTOR_PDF_EXECUTABLE NAMES asciidoctor-pdf)
# 
# set(STYLES_DIR "${CMAKE_CURRENT_SOURCE_DIR}/asciidoctor-stylesheet-factory/stylesheets")
# set(MANUAL_SRC "${CMAKE_CURRENT_SOURCE_DIR}/manual.txt")
# set(MANUAL_OUT_HTML "${CMAKE_CURRENT_BINARY_DIR}/manual.html")
# set(MANUAL_OUT_PDF "${CMAKE_CURRENT_BINARY_DIR}/manual.pdf")
# 
# add_custom_target(manual-html COMMAND ${ASCIIDOCTOR_EXECUTABLE}
# 	-a stylesheet=readthedocs.css -a stylesdir=${STYLES_DIR} -a toc=left
# 	--destination-dir=${CMAKE_CURRENT_BINARY_DIR} ${MANUAL_SRC}
# 	BYPRODUCTS ${MANUAL_OUT_HTML} DEPENDS ${MANUAL_SRC}
# 	WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
# 	COMMENT "Generating manual, HTML output")
# 
# if(ASCIIDOCTOR_PDF_EXECUTABLE)
# 	add_custom_target(manual-pdf COMMAND ${ASCIIDOCTOR_PDF_EXECUTABLE}
# 		-a stylesheet=readthedocs.css -a stylesdir=${STYLES_DIR}
# 		--destination-dir=${CMAKE_CURRENT_BINARY_DIR} ${MANUAL_SRC}
# 		BYPRODUCTS ${MANUAL_OUT_PDF} DEPENDS ${MANUAL_SRC}
# 		WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
# 		COMMENT "Generating manual, HTML output")
# else()
# 	add_custom_target(manual-pdf COMMAND ${CMAKE_COMMAND} -E echo
# 		 "asciidoctor-pdf missing, NOT generating PDF output"
# 		DEPENDS ${MANUAL_SRC}
# 		WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
# endif()
# 
# add_custom_target(manual)
# add_dependencies(manual manual-html manual-pdf)
# 
# install(FILES ${MANUAL_OUT_HTML} ${MANUAL_OUT_PDF}
# 	DESTINATION ${CMAKE_INSTALL_DOCDIR} OPTIONAL)
# if(ASCIIDOCTOR_EXECUTABLE)
# 	install(DIRECTORY manual/images DESTINATION ${CMAKE_INSTALL_DOCDIR}
# 		PATTERN "*.xcf" EXCLUDE)
# endif()
