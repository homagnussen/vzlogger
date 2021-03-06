# -*- mode: cmake; -*-
project(vzlogger)
cmake_minimum_required(VERSION 2.6.2 FATAL_ERROR)
if(POLICY CMP0011)
  cmake_policy(SET CMP0011 NEW)
endif(POLICY CMP0011)

set(VZLOGGER_MAJOR_VERSION 0)
set(VZLOGGER_MINOR_VERSION 6)
set(VZLOGGER_SUB_VERSION   1)
set(VERSION_SEP "-")
set(VZLOGGER_SHORT_VERSION "${VZLOGGER_MAJOR_VERSION}${VERSION_SEP}${VZLOGGER_MINOR_VERSION}")
set(VZLOGGER_VERSION "${VZLOGGER_SHORT_VERSION}${VERSION_SEP}${VZLOGGER_SUB_VERSION}")
set(VZLOGGER_RPM_VERSION "${VZLOGGER_MAJOR_VERSION}.${VZLOGGER_MINOR_VERSION}.${VZLOGGER_SUB_VERSION}")
set(VERSION_DATE "2012-02-20")
set(PACKAGE_STRING "${PROJECT_NAME} ${VZLOGGER_RPM_VERSION}")
set(VERSION "${VZLOGGER_RPM_VERSION}")
set(CMAKE_VERBOSE_MAKEFILE 1)

# debug build, on by default
ADD_DEFINITIONS(-g3)

###########################################################
# Where are the additional libraries installed? Note: provide includes
# path here, subsequent checks will resolve everything else
set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/modules")

# get git sha1 as additional version:
include(GetGitRevisionDescription)
get_git_head_revision(GIT_REFSPEC GIT_SHA1)
git_describe(GIT_SHALONG --always --abbrev=10 --long --all --dirty )
git_get_last_commit_date(GIT_LAST_COMMIT_DATE )

message("Compiling for target '${TARGET}'")
if( TARGET )
  if( ${TARGET} STREQUAL "ar71xx")
    include ( OpenWRT )
    message("Compiling for ar71xx")
    set(CMAKE_FIND_LIBRARY_SUFFIXES ".a")
    set(LINK_FLAGS_RELEASE "-static")
  endif( ${TARGET} STREQUAL "ar71xx")
else( TARGET )
  include ( CompilerFlags )
endif( TARGET )

include ( ConfigureMacro )

# options
OPTION(ENABLE_SML
  "enable support for smart messaging language (def=yes)])"
  On)
OPTION(ENABLE_OMS
  "enable support for OMS (M-Bus) based devices (def=yes)"
  On)
OPTION(ENABLE_LOCAL
  "enable support for local HTTPd (def=yes)]"
  On)
OPTION(WITH_READER
  "compile reader to for testing your meters (def=yes)])"
  On)
OPTION(ENABLE_OCR
	"enable support for OCR image/processing meter (def=off)"
	OFF)

OPTION(ENABLE_OCR_TESSERACT
	"enable support for OCR text recognition meter (def=off)"
	OFF)

OPTION(METEREXEC_ROOTACCESS
  "compile MeterExec protocol to use root privileges for script calls (def=no)])"
  Off)

# find dependencies
# libsml
if( ENABLE_SML )
  include(FindSML)
  if(SML_FOUND)
    set(SML_SUPPORT 1)
    include_directories(${SML_INCLUDE_DIR})
  endif(SML_FOUND)
endif( ENABLE_SML )

if( ENABLE_OMS )
  # find dependencies todo libmbus
  include(FindMBus)
  include(FindOpenSSL)
  if(MBUS_FOUND)
    set(OMS_SUPPORT 1)
    include_directories(${MBUS_INCLUDE_DIR})
    set(OMS_LIBRARIES ${MBUS_LIBRARY})
  elseif(MBUS_FOUND)
    set(OMS_SUPPORT 0)
    message( STATUS "libmbus not found. Disabling ENABLE_OMS" )
  endif(MBUS_FOUND)

endif( ENABLE_OMS )

if(ENABLE_LOCAL)
  include(FindMicroHttpd)
  if(MICROHTTPD_FOUND)
    set(MICROHTTPD_SUPPORT 1)
    set(LOCAL_SUPPORT 1)
  endif(MICROHTTPD_FOUND)
endif(ENABLE_LOCAL)

if( ENABLE_OCR OR ENABLE_OCR_TESSERACT )
	include(FindLeptonica)
	if (NOT LEPTONICA_FOUND)
		message( STATUS "Leptonica not found. Disabling ENABLE_OCR" )
		set(ENABLE_OCR false)
		set(ENABLE_OCR_TESSERACT false)
	else ()
		message("OCR Meter enabled")
		# TODO install tesseract
		# and set include_directories appropriate
		if (ENABLE_OCR_TESSERACT)
			set(OCR_LIBRARIES ${LEPTONICA_LIBRARIES} tesseract)
			set(OCR_TESSERACT_SUPPORT 1)
		else ()
			set( OCR_LIBRARIES ${LEPTONICA_LIBRARIES})
		endif( ENABLE_OCR_TESSERACT)
		include_directories(${LEPTONICA_INCLUDE_DIRS})
		set(OCR_SUPPORT 1)
	endif( NOT LEPTONICA_FOUND )
endif( ENABLE_OCR OR ENABLE_OCR_TESSERACT )

IF(NOT WIN32)
  include(FindThreads)
ENDIF(NOT WIN32)

# libjson
include(FindJson)
if( NOT JSON_FOUND)
  message(FATAL_ERROR "libjson-c is required.
Install json-c or call cmake -DJSON_HOME=path_to_json_install")
endif( NOT JSON_FOUND)
include_directories(${JSON_INCLUDE_DIR})

# check for curl
IF(WIN32)
  add_definitions(-DCURL_STATICLIB)
  include(FindCURL_WIN32)
else(WIN32)
  #  add_definitions(-DCURL_STATICLIB)
  include(FindCURL)
  include(FindGnutls)
  include(FindOpenSSL) # needed by MySmartGrid API...
endif(WIN32)

find_library(LIBUUID uuid)
find_library(LIBGCRYPT gcrypt)

include_directories(${CMAKE_BINARY_DIR})
include_directories(${CMAKE_SOURCE_DIR}/include)
include_directories(${CMAKE_SOURCE_DIR})

add_definitions(-DHAVE_CONFIG_HPP)
myconfigure_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/config.hpp.in
  ${CMAKE_CURRENT_BINARY_DIR}/config.hpp
  )

add_subdirectory(src)

find_package(Git)

set(ENABLE_GOOGLEMOCK TRUE)
#disable unit tests on gcc4.6 (problems with googlemock 1.7)
if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
    # require at least gcc 4.8
    execute_process(COMMAND ${CMAKE_CXX_COMPILER} -dumpversion
       OUTPUT_VARIABLE GOOGLETEST_GCC_COMPILER_VERSION)
# on travis-ci the CMAKE_CXX_COMPILER_VERSION is empty!    message(WARNING "Using compiler: ${CMAKE_CXX_COMPILER_VERSION} ${GOOGLETEST_GCC_COMPILER_VERSION}") 
    if (GOOGLETEST_GCC_COMPILER_VERSION VERSION_LESS 4.8)
        message(WARNING "Disabled googlemock/-test tests due to GCC version < 4.8!")
	set(ENABLE_GOOGLEMOCK FALSE)
    endif()
endif()

# enable unit testing
include(CTest)
enable_testing()

if(GIT_FOUND AND ENABLE_GOOGLEMOCK)
  add_subdirectory(gmock)
  add_subdirectory(tests)
else()
 if (ENABLE_GOOGLEMOCK)
    message(WARNING "googletest based unit tests disabled due to missing git. Please install git.")
 endif(ENABLE_GOOGLEMOCK)
endif()
add_test(vzlogger_unit_tests tests/vzlogger_unit_tests)

### print some output for the user
message("")
message("        ***** Configuration parameters *****")
message("             prefix: ${CMAKE_INSTALL_PREFIX}")
message("             json: -L${JSON_LIBRARY} -I${JSON_INCLUDE_DIR}")
if(SML_FOUND)
  message("             sml:  -L${SML_LIBRARY} -I${SML_INCLUDE_DIR}")
else(SML_FOUND)
  message("             sml: NOT FOUND")
endif(SML_FOUND)
if(MICROHTTPD_FOUND)
message("             microhttpd: -L${MICROHTTPD_LIBRARY} -I${MICROHTTPD_INCLUDE_DIR}")
endif(MICROHTTPD_FOUND)
if(METEREXEC_ROOTACCESS)
  message("             MeterExec: root privileges")
endif(METEREXEC_ROOTACCESS)
if(MBUS_FOUND)
message("             libmbus: -L${MBUS_LIBRARY} -I${MBUS_INCLUDE_DIR}")
endif(MBUS_FOUND)

if( ENABLE_SML AND NOT SML_FOUND)
  message(WARNING "libsml was not found.
Install libsml or call cmake -DSML_HOME=path_to_sml_install")
endif( ENABLE_SML AND NOT SML_FOUND)
if( ENABLE_LOCAL AND NOT MICROHTTPD_FOUND )
  message(WARNING "microhttpd was not found. Ignored ENABLE_LOCAL!
Install microhttpd or call cmake -DMICROHTTPD_HOME=path_to_microhttpd_install")
endif( ENABLE_LOCAL AND NOT MICROHTTPD_FOUND )

# add some files to the installation target
INSTALL(FILES README.md COPYING DESTINATION
  share/doc/vzlogger-${VZLOGGER_MAJOR_VERSION}-${VZLOGGER_MINOR_VERSION}
)

# add clean-all target that removes the cached files from cmake as well (see e.g. issue #186)
add_custom_target(clean-all
    COMMAND ${CMAKE_BUILD_TOOL} clean
    COMMAND ${CMAKE_COMMAND} -P clean-all-cmake.txt
)

# CPACK packaging
INCLUDE(InstallRequiredSystemLibraries)
set(CPACK_DEBIAN_PACKAGE_SHLIBDEPS On)
#set(CPACK_SOURCE_GENERATOR "TGZ;TBZ2;PKG")
set(CPACK_SOURCE_GENERATOR "TGZ;TBZ2")

set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "volkszaehler.org ist ein freies Smart Meter (hier: intelligenter Stromzähler) im Selbstbau.")
set(CPACK_PACKAGE_VENDOR              "Steffen Vogel")
set(CPACK_PACKAGE_CONTACT             "Steffen Vogel <info@steffenvogel.de>")
set(CPACK_DEBIAN_PACKAGE_MAINTAINER   "Steffen Vogel <info@steffenvogel.de>")
set(CPACK_PACKAGE_DESCRIPTION_FILE    "${CMAKE_CURRENT_SOURCE_DIR}/README.md")
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/COPYING")
set(CPACK_PACKAGE_VERSION_MAJOR ${VZLOGGER_MAJOR_VERSION})
set(CPACK_PACKAGE_VERSION_MINOR ${VZLOGGER_MINOR_VERSION})
set(CPACK_PACKAGE_VERSION_PATCH ${VZLOGGER_SUB_VERSION})
set(CPACK_PACKAGE_INSTALL_DIRECTORY "${PROJECT_NAME}-${VZLOGGER_MAJOR_VERSION}.${VZLOGGER_MINOR_VERSION}")

set(CPACK_SOURCE_IGNORE_FILES "/CVS/;/.svn/;.swp$;.#;/#;.*~;cscope.*;build.*")

IF(WIN32 AND NOT UNIX)
  set(CPACK_GENERATOR "NSIS")
  set(CPACK_NSIS_PACKAGE_NAME "Gridlicense Demo Tools")
  # set(CPACK_PACKAGE_EXECUTABLES "gui4" "Grid License Demo Frontend")
  set(CPACK_PACKAGE_DEFAULT_LOCATION ${INSTALL_DIR})

  # sure there is at least one set of four (4) backlasshes.
  set(CPACK_NSIS_INSTALLED_ICON_NAME "bin\\vzlogger.exe")
  set(CPACK_NSIS_DISPLAY_NAME "${CPACK_PACKAGE_INSTALL_DIRECTORY} VZlogger")
  set(CPACK_NSIS_HELP_LINK    "http://volkszaehler.org/")
  set(CPACK_NSIS_URL_INFO_ABOUT "http://wiki.volkszaehler.org/software/controller/vzlogger")
  set(CPACK_NSIS_CONTACT "Steffen Vogel <info@steffenvogel.de>")
  set(CPACK_NSIS_MODIFY_PATH ON)
  # set(CPACK_PACKAGE_INSTALL_REGISTRY_KEY "${CMAKE_PROJECT_NAME}-${EWS_VERSION}")
  # set(CPACK_NSIS_MENU_LINKS "${EWS_WEBSITE}" "Homepage for ${EWS_APP_NAME}")
  set(CPACK_NSIS_DEFINES "  !define MUI_WELCOMEPAGE_TITLE_3LINES 1
    !define MUI_FINISHPAGE_TITLE_3LINES 1")
else(WIN32 AND NOT UNIX)
  #set(CPACK_SET_DESTDIR On)
  set(CPACK_SOURCE_STRIP_FILES "")
  set(CPACK_GENERATOR "TGZ;TBZ2;TZ;DEB;RPM")
  set(CPACK_RPM_PACKAGE_RELOCATABLE 1)
  set(CPACK_SYSTEM_NAME ${CMAKE_HOST_SYSTEM_PROCESSOR})
endif(WIN32 AND NOT UNIX)
include(CPack)

