cmake_minimum_required(VERSION 2.8.12)

if(NOT DEFINED BUILD_TESTING)
    set(BUILD_TESTING 1)
endif()

if(NOT CMAKE_CROSSCOMPILING)
    enable_testing()
endif()

option(RUN_WITH_WINE		"Use wine to run tests" OFF)


if(NOT LIBDIR)
	set(LIBDIR lib CACHE STRING "Library installation directory" FORCE)
endif()

if(CMAKE_COMPILER_IS_GNUCXX)
	set(CXX_FLAGS "-Wall -Wextra")
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
	set(CXX_FLAGS "-Wall -Wextra")
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Intel")
	set(CXX_FLAGS "-Wall")
elseif(MSVC)
	set(CXX_FLAGS "/EHsc /W3")
endif()

set(CMAKE_DEBUG_POSTFIX "-d")


set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CXX_FLAGS}")

set(NOWIDE_TESTS
    test_convert
    test_stdio
    test_fstream
    )

if(BUILD_TESTING)
    foreach(TEST ${NOWIDE_TESTS})
        add_executable(${TEST} test/${TEST}.cpp)
        target_link_libraries(${TEST} nowide-static)
        if(RUN_WITH_WINE)
           add_test(NAME ${TEST} WORKING_DIRECTORY ${CMAKE_BUILD_DIR} COMMAND wine ./${TEST}.exe)
        else()
           add_test(${TEST} ${TEST})
        endif()
    endforeach()
endif()

if(DEFINED BUILD_SHARED_LIBS)
    if(BUILD_SHARED_LIBS)
        set(BUILD_SHARED 1)
    else()
        set(BUILD_STATIC 1)
    endif()
else()
    set(BUILD_SHARED 1)
    set(BUILD_STATIC 1)
endif()

if(BUILD_SHARED OR BUILD_TESTING)
    add_library(nowide SHARED src/iostream.cpp)
    set_target_properties(nowide PROPERTIES
        VERSION 0.0.0 SOVERSION 0
		CLEAN_DIRECT_OUTPUT 1
	)
    target_compile_definitions(nowide PUBLIC DLL_EXPORT)
    target_include_directories(nowide PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<INSTALL_INTERFACE:include>
    )
    if(BUILD_SHARED)
        list(APPEND targets nowide)
    endif()
endif()

if(BUILD_STATIC OR BUILD_TESTING)
    add_library(nowide-static STATIC src/iostream.cpp)
    set_target_properties(nowide-static PROPERTIES
    	CLEAN_DIRECT_OUTPUT 1
    )
    target_include_directories(nowide-static PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<INSTALL_INTERFACE:include>
    )
    if(MSVC)
        set_target_properties(nowide-static PROPERTIES PREFIX "lib")
    endif()
    if(BUILD_STATIC)
        list(APPEND targets nowide-static)
    endif()
endif()

if(BUILD_TESTING)
    add_executable(test_iostream_shared test/test_iostream.cpp)
    target_link_libraries(test_iostream_shared nowide)

    add_executable(test_iostream_static test/test_iostream.cpp)

    add_executable(test_system test/test_system.cpp)


    add_executable(test_env_proto test/test_env.cpp)
    add_executable(test_env_win test/test_env.cpp)
    set_target_properties(test_env_win PROPERTIES COMPILE_DEFINITIONS NOWIDE_TEST_INCLUDE_WINDOWS)

    foreach(t test_iostream_static test_env_proto test_env_win test_system)
        target_link_libraries(${t} nowide-static)
    endforeach()

    set(OTHER_TESTS test_iostream_shared test_iostream_static test_env_win test_env_proto)

    if(RUN_WITH_WINE)
    	foreach(T ${OTHER_TESTS})
    		add_test(NAME ${T} WORKING_DIRECTORY ${CMAKE_BUILD_DIR} COMMAND wine ./${T}.exe)
    	endforeach()

    	add_test(NAME test_system_n WORKING_DIRECTORY ${CMAKE_BUILD_DIR} COMMAND wine ./test_system.exe "-n")
    	add_test(NAME test_system_w WORKING_DIRECTORY ${CMAKE_BUILD_DIR} COMMAND wine ./test_system.exe "-w")
    else()
    	foreach(T ${OTHER_TESTS})
    		add_test(${T} ${T})
    	endforeach()

    	add_test(test_system_n test_system "-n")
    	add_test(test_system_w test_system "-w")
    endif()
endif()

install(TARGETS ${targets} EXPORT nowide-targets
	RUNTIME DESTINATION bin
	LIBRARY DESTINATION ${LIBDIR}
	ARCHIVE DESTINATION ${LIBDIR})

install(DIRECTORY nowide DESTINATION include)
install(EXPORT nowide-targets FILE nowide-config.cmake
    DESTINATION lib/cmake/nowide
    NAMESPACE nowide::)
