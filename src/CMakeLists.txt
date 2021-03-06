# CMakeLists for AppStream database shared lib

pkg_check_modules(GLIB2 REQUIRED glib-2.0>=2.46)
pkg_check_modules(GIO2 REQUIRED gio-2.0)
find_package(GI REQUIRED)
find_package(LibXml2 REQUIRED)
find_library(M_LIB m)
pkg_check_modules(YAML REQUIRED yaml-0.1)

if (STEMMING)
	find_path(SNOWBALL_INCLUDE_DIR libstemmer.h HINTS "/usr/include" "/usr/include/libstemmer")
	find_library(SNOWBALL_LIB stemmer)
	if (NOT SNOWBALL_LIB OR NOT SNOWBALL_INCLUDE_DIR)
		message (FATAL_ERROR "Stemming support is enabled, but libstemmer was not found. Please install the Snowball development files.")
	endif ()
else ()
	set(SNOWBALL_LIB "")
	set(SNOWBALL_INCLUDE_DIR "")
endif ()

include(GObjectIntrospectionMacros)
set(INTROSPECTION_GIRS)
set(INTROSPECTION_SCANNER_ARGS "--identifier-prefix=As" "--symbol-prefix=as" "--warn-all" "--add-include-path=${CMAKE_CURRENT_SOURCE_DIR}")
set(INTROSPECTION_COMPILER_ARGS "--includedir=${CMAKE_CURRENT_SOURCE_DIR}")

# check if the git tools are really found.
if (NOT INTROSPECTION_SCANNER)
	message (FATAL_ERROR "GObject introspection scanner was not found! Please install it to continue.")
endif ()
if (NOT INTROSPECTION_COMPILER)
	message (FATAL_ERROR "GObject introspection compiler was not found! Please install it to continue.")
endif ()

set (srcdir ${CMAKE_CURRENT_SOURCE_DIR} CACHE INTERNAL "Source dir")
set(AS_VERSION_BASE "${AS_VERSION_MAJOR}.${AS_VERSION_MINOR}" CACHE INTERNAL "AppStream library combined version number")

set(APPSTREAM_LIB_SRC
		as-utils.c
		# internal
		as-xmldata.c
		as-yamldata.c
		as-desktop-entry.c
		as-cache-file.c
		as-distro-extras.c
		as-stemmer.c
		# (mostly) public
		as-spdx.c
		as-metadata.c
		as-component.c
		as-enums.c
		as-provided.c
		as-bundle.c
		as-pool.c
		as-category.c
		as-distro-details.c
		as-screenshot.c
		as-image.c
		as-release.c
		as-checksum.c
		as-validator.c
		as-validator-issue.c
		as-icon.c
		as-translation.c
		as-suggested.c
		as-content-rating.c
		as-launchable.c
)

set(APPSTREAM_LIB_PUBLIC_HEADERS
		appstream.h
		as-utils.h
		as-spdx.h
		as-metadata.h
		as-component.h
		as-pool.h
		as-enums.h
		as-provided.h
		as-bundle.h
		as-category.h
		as-distro-details.h
		as-screenshot.h
		as-image.h
		as-release.h
		as-checksum.h
		as-validator.h
		as-validator-issue.h
		as-icon.h
		as-translation.h
		as-suggested.h
		as-content-rating.h
		as-launchable.h
)

set(APPSTREAM_LIB_PRIVATE_HEADERS
		as-utils-private.h
		as-xmldata.h
		as-yamldata.h
		as-desktop-entry.h
		as-cache-file.h
		as-pool-private.h
		as-component-private.h
		as-screenshot-private.h
		as-release-private.h
		as-distro-extras.h
		as-stemmer.h
		as-content-rating-private.h
)

include_directories(${CMAKE_BINARY_DIR}
		    ${CMAKE_BINARY_DIR}/src
		    ${GLIB2_INCLUDE_DIRS}
		    ${GIO2_INCLUDE_DIRS}
		    ${LIBXML2_INCLUDE_DIR}
		    ${YAML_INCLUDE_DIR}
		    ${SNOWBALL_INCLUDE_DIR}
)

add_definitions("-DAS_COMPILATION")

# ensure that the GI compiler has been found (apparently, the automatic tests fail on some machines)
if (NOT INTROSPECTION_COMPILER)
  message(FATAL_ERROR "GObject-Introspection was not found! Please install it to continue.")
endif()

# GResources
find_program (GCR_TOOL glib-compile-resources)
if (NOT GCR_TOOL)
	message(FATAL_ERROR "Could not find glib-compile-resources! Please mek the tool available to continue.")
endif ()

add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/as-resources.h
	DEPENDS appstream.gresource.xml
		${CMAKE_SOURCE_DIR}/data/xdg-category-names.txt
	COMMAND ${GCR_TOOL}
		"--sourcedir=${CMAKE_SOURCE_DIR}/data"
		"--target=as-resources.h"
		"--generate-header"
		"--c-name=as"
		"${CMAKE_CURRENT_SOURCE_DIR}/appstream.gresource.xml"
	WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
	VERBATIM
)

add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/as-resources.c
	DEPENDS appstream.gresource.xml
		${CMAKE_SOURCE_DIR}/data/xdg-category-names.txt
	COMMAND ${GCR_TOOL}
		"--sourcedir=${CMAKE_SOURCE_DIR}/data"
		"--target=as-resources.c"
		"--generate-source"
		"--c-name=as"
		"${CMAKE_CURRENT_SOURCE_DIR}/appstream.gresource.xml"
	WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
	VERBATIM
)

set (APPSTREAM_LIB_GRESOURCE ${CMAKE_CURRENT_BINARY_DIR}/as-resources.c
			      ${CMAKE_CURRENT_BINARY_DIR}/as-resources.h
)

add_library(appstream SHARED
		${APPSTREAM_LIB_SRC}
		${APPSTREAM_LIB_PUBLIC_HEADERS}
		${APPSTREAM_LIB_PRIVATE_HEADERS}
		${APPSTREAM_LIB_GRESOURCE}
)

set_target_properties(appstream PROPERTIES VERSION ${AS_VERSION} SOVERSION ${APPSTREAM_LIB_API_LEVEL})

target_link_libraries(appstream
		${SANITIZER_LIBS}
		${GLIB2_LIBRARIES}
		${GIO2_LIBRARIES}
		${LIBXML2_LIBRARIES}
		${YAML_LIBRARIES}
		${SNOWBALL_LIB}
)

configure_file (${CMAKE_CURRENT_SOURCE_DIR}/appstream.pc.in ${CMAKE_CURRENT_BINARY_DIR}/appstream.pc @ONLY)

####
# GObject Introspection

if(NOT (MAINTAINER AND SANITIZERS))
# compiling GIR doesn't work well with asan, needs to be investigated.

macro(_list_prefix _outvar _listvar _prefix)
  set(${_outvar})
  foreach(_item IN LISTS ${_listvar})
    list(APPEND ${_outvar} ${_prefix}${_item})
  endforeach()
endmacro(_list_prefix)

set(introspection_files ${APPSTREAM_LIB_SRC} ${APPSTREAM_LIB_PUBLIC_HEADERS})
set(AppStream_1_0_gir "appstream")
set(AppStream_1_0_gir_INCLUDES GObject-2.0 Gio-2.0)
set(AppStream_1_0_gir_CFLAGS "-I${CMAKE_CURRENT_SOURCE_DIR}/" "-DAS_COMPILATION" ${MAINTAINER_CFLAGS})
set(AppStream_1_0_gir_LIBS appstream)

_list_prefix(AppStream_1_0_gir_FILES introspection_files "${CMAKE_CURRENT_SOURCE_DIR}/")
set(AppStream_1_0_gir_SCANNERFLAGS --c-include appstream.h)
set(AppStream_1_0_gir_EXPORT_PACKAGES appstream)

list(APPEND INTROSPECTION_GIRS AppStream-1.0.gir)
gir_add_introspections(INTROSPECTION_GIRS)

endif()
####

install(TARGETS appstream EXPORT AppStreamLibsTarget DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(FILES ${APPSTREAM_LIB_PUBLIC_HEADERS} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/AppStream)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/appstream.pc DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig/)
