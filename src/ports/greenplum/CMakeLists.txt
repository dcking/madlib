include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/../postgres
    ${CMAKE_CURRENT_SOURCE_DIR}
)

set(MAD_DBAL_GP_SOURCES
	../postgres/dbconnector/PGAbstractValue.cpp
    dbconnector/GPCompatibility.cpp
	../postgres/dbconnector/PGNewDelete.cpp
    ../postgres/dbconnector/PGAllocator.cpp
	../postgres/dbconnector/PGInterface.cpp
	../postgres/dbconnector/PGToDatumConverter.cpp
	../postgres/dbconnector/PGValue.cpp
)

if(LINUX)
	list(APPEND MAD_DBAL_GP_SOURCES
		linux/GPMain.cpp
	)
else(LINUX)
    list(APPEND MAD_DBAL_GP_SOURCES
    	../postgres/dbconnector/PGMain.cpp
    )
endif(LINUX)

file(GLOB_RECURSE PYTHON_FILES
    RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}/../postgres/modules"
    "../postgres/modules/*.py_in"
)

file(GLOB_RECURSE SQL_FILES
    RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}/../postgres/modules"
    "../postgres/modules/*.sql_in"
)

message(STATUS "Trying to find Greenplum installation")
find_package(Greenplum)

if(GREENPLUM_FOUND)
    message(STATUS "***")
    message(STATUS "*** Adding Greenplum to target list...")
    message(STATUS "***")
    
    include_directories(${GREENPLUM_INCLUDE_DIR})
    
    # Directory we will use for patches
    set(PATCH_DIR ${CMAKE_CURRENT_BINARY_DIR}/patch)
    
    # Version 4.1 of Greenplum has a bug in one of its header files that
    # needs to be fixed when building with C++.
    set(_GP_VERSION "${GREENPLUM_VERSION_MAJOR}.${GREENPLUM_VERSION_MINOR}.${GREENPLUM_VERSION_PATCH}")
    if(${_GP_VERSION} VERSION_EQUAL "4.1.0" OR ${_GP_VERSION} VERSION_EQUAL "4.1.1")
        
        message(STATUS "Installed Greenplum version has bug in header file. Applying patch...")
        
        # The first element in GREENPLUM_INCLUDE_DIR is include/server
        list(GET GREENPLUM_INCLUDE_DIR 0 GREENPLUM_SERVER_INCLUDE_DIR)
        
        file(COPY ${GREENPLUM_SERVER_INCLUDE_DIR}/utils/syncbitvector.h DESTINATION
            ${PATCH_DIR}/server/utils)
        execute_process(
            COMMAND ${MAD_BUILD_TOOLS}/greenplum_patch_4_1_x.sh
            WORKING_DIRECTORY ${PATCH_DIR}
        )
        
        # Append directory with patched header file at beginning
        include_directories(BEFORE ${PATCH_DIR}/server)
    endif(${_GP_VERSION} VERSION_EQUAL "4.1.0" OR ${_GP_VERSION} VERSION_EQUAL "4.1.1")

    # Create library
    add_library(
        madlib_greenplum
        MODULE
        ${MAD_DBAL_GP_SOURCES}
    )
    add_dependencies(madlib_greenplum madlib)
    install(TARGETS madlib_greenplum
        LIBRARY DESTINATION ports/greenplum/lib)
    set_target_properties(madlib_greenplum PROPERTIES
        LIBRARY_OUTPUT_DIRECTORY lib
        BUILD_WITH_INSTALL_RPATH YES)

#    set_target_properties(madlib_greenplum
#        OUTPUT_NAME madlib)

    if(APPLE)
        osx_archs(${GREENPLUM_EXECUTABLE} MAD_ARCHITECTURES)
        message(STATUS "Will build madlib Greenplum connector for same "
            "architectures as detected in ${GREENPLUM_EXECUTABLE}, which has "
            "architectures ${MAD_ARCHITECTURES}")
        set_target_properties(madlib_greenplum PROPERTIES
            OSX_ARCHITECTURES "${MAD_ARCHITECTURES}"
            LINK_FLAGS "-bundle_loader ${GREENPLUM_EXECUTABLE} -Wl,-rpath -Wl,@loader_path/../../../lib")
        target_link_libraries(madlib_greenplum madlib)

        # the RPATH to be used when installing. This is set to "$ORIGIN/../../lib"
        # because the core MADlib library will reside in $MADLIB_ROOT/lib
        # and the Greenplum connector library in $MADLIB_ROOT/ports/greenplum/lib"
        set_target_properties(madlib_greenplum
            PROPERTIES INSTALL_RPATH "@loader_path/../../../lib")
    else(APPLE)
        if(NOT LINUX)
            # On non-Linux platforms (where there is direct binding), we declare
            # madlib as a dependency. On Linux, madlib is laoded dynamically.
            target_link_libraries(madlib_greenplum madlib)
        endif(NOT LINUX)

        # the RPATH to be used when installing. This is set to "$ORIGIN/../../lib"
        # because the core MADlib library will reside in $MADLIB_ROOT/lib
        # and the Greenplum connector library in $MADLIB_ROOT/ports/greenplum/lib"
        set_target_properties(madlib_greenplum
            PROPERTIES INSTALL_RPATH "\$ORIGIN/../../../lib")
    endif(APPLE)

    # Directory we will use for python files
#    set(PYTHON_DIR ${CMAKE_CURRENT_BINARY_DIR}/modules)
#    execute_process(COMMAND ${CMAKE_COMMAND} -E make_directory ${PYTHON_DIR})
    list(APPEND M4_ARGUMENTS
        "-DGREENPLUM"
    )
    foreach(CURRENT_FILE ${PYTHON_FILES})
        get_filename_component(CURRENT_PATH "../postgres/modules/${CURRENT_FILE}" ABSOLUTE)
        
        string(REGEX REPLACE ".py_in\$" ".py" OUTFILE "${CMAKE_CURRENT_BINARY_DIR}/modules/${CURRENT_FILE}")
        get_dir_name(OUTDIR ${OUTFILE})
        add_custom_command(OUTPUT "${OUTFILE}"
            COMMAND ${CMAKE_COMMAND} -E make_directory ${OUTDIR}
            COMMAND ${M4_BINARY} ${M4_ARGUMENTS} "${CURRENT_PATH}" > "${OUTFILE}"
            DEPENDS "${CURRENT_PATH}"
            COMMENT "Preprocessing ${CURRENT_FILE} with m4.")
        
        list(APPEND PYTHON_TARGET_FILES
            ${OUTFILE})
    endforeach(CURRENT_FILE)
    add_custom_target(pythonFiles ALL DEPENDS ${PYTHON_TARGET_FILES})
    
    # Directory we will use for SQL files
#    set(SQL_DIR ${CMAKE_CURRENT_BINARY_DIR}/sql)
#    execute_process(COMMAND ${CMAKE_COMMAND} -E make_directory ${SQL_DIR})
    foreach(CURRENT_FILE ${SQL_FILES})
        get_filename_component(CURRENT_PATH "../postgres/modules/${CURRENT_FILE}" ABSOLUTE)

        set(OUTFILE "${CMAKE_CURRENT_BINARY_DIR}/modules/${CURRENT_FILE}")
        add_custom_command(OUTPUT "${OUTFILE}"
#            COMMAND ${CMAKE_COMMAND} -E touch ${OUTFILE}
            COMMAND ${CMAKE_COMMAND} -E copy "${CURRENT_PATH}" "${OUTFILE}"
            DEPENDS "${CURRENT_PATH}"
            COMMENT "Copying ${CURRENT_FILE}.")
        
        list(APPEND SQL_TARGET_FILES
            ${OUTFILE})
    endforeach(CURRENT_FILE)
    add_custom_target(sqlFiles ALL DEPENDS ${SQL_TARGET_FILES})

    install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/modules
        DESTINATION ports/greenplum
        REGEX ".*/\\.DS_Store\$" EXCLUDE
    )
    
#    install(DIRECTORY ${SQL_DIR}
#        DESTINATION ports/greenplum
#        REGEX ".*/\\.DS_Store\$" EXCLUDE
#    )

else(GREENPLUM_FOUND)
    message(STATUS "***")
    message(STATUS "*** No Greenplum installation found. Skipping.")
    message(STATUS "***")
endif(GREENPLUM_FOUND)