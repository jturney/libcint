cmake_policy(SET CMP0048 NEW)
project(cint VERSION 2.8.7 LANGUAGES C)

set(cint_AUTHORS "Qiming Sun")
set(cint_DESCRIPTION "An open source library for analytical Gaussian integrals.")
set(cint_EMAIL "osirpt.sun@gmail.com")
set(cint_URL "https://github.com/sunqm/libcint")
set(cint_LICENSE "BSD 2-clause \"Simplified\" License")

cmake_minimum_required (VERSION 3.0)

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)

################################### Options ####################################
include(psi4OptionsTools)
option_with_default(CMAKE_BUILD_TYPE "Build type" RelWithDebInfo)
option_with_default(BUILD_SHARED_LIBS "Build final library as shared, not static" OFF)
option_with_default(BUILD_FPIC "Libraries will be compiled with position independent code" ON)
if (${BUILD_SHARED_LIBS} AND NOT ${BUILD_FPIC})
	message(FATAL_ERROR "BUILD_SHARED_LIBS ON and BUILD_FPIC OFF are incompatible, as shared library requires position independent code")
endif()
option_with_print(ENABLE_GENERIC "Enable mostly static linking in shared library" OFF)
option_with_flags(ENABLE_XHOST "Enable processor-specific optimization" ON "-xHost" "-march=native")

######################### Process & Validate Options ###########################
include(autocmake_safeguards)
include(custom_static_library)

find_package(TargetLAPACK REQUIRED)

################################# Main Project #################################
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

set(PN ${PROJECT_NAME})

find_package(QUADMATH)
if(QUADMATH_FOUND)
  set(HAVE_QUADMATH_H 1 CACHE INTERNAL "Have QUADMATH")
endif()

#find_package(BLAS REQUIRED)

include(CheckFunctionExists)
set(CMAKE_REQUIRED_INCLUDES math.h)
set(CMAKE_REQUIRED_LIBRARIES m)
check_function_exists(expl HAVE_EXPL)
check_function_exists(sqrtl HAVE_SQRTL)
#set(CMAKE_REQUIRED_INCLUDES quadmath.h)
#set(CMAKE_REQUIRED_LIBRARIES quadmath)
#check_function_exists(fabsq HAVE_QUADMATH_H)

include_directories(
  ${PROJECT_BINARY_DIR}/include
  ${PROJECT_SOURCE_DIR}/src
  ${PROJECT_BINARY_DIR}/src)

configure_file(
  "${PROJECT_SOURCE_DIR}/src/config.h.in"
  "${PROJECT_BINARY_DIR}/src/config.h")

configure_file(
  "${PROJECT_SOURCE_DIR}/include/cint.h.in"
  "${PROJECT_BINARY_DIR}/include/cint.h")

set(cintSrc
  src/c2f.c src/cart2sph.c src/cint1e.c src/cint2e.c src/cint_bas.c
  src/fblas.c src/g1e.c src/g2e.c src/misc.c src/optimizer.c
  src/rys_roots.c
  src/cint2c2e.c src/g2c2e.c src/cint3c2e.c src/g3c2e.c src/optimizer_3c2e.c
  src/cint3c1e.c src/g3c1e.c src/breit.c src/cint4c1e.c src/g4c1e.c
  src/autocode/auto_intor1.c src/autocode/auto_intor2.c
  src/autocode/auto_intor3.c src/autocode/grad1.c
  src/autocode/grad2.c src/autocode/grad3.c src/autocode/auto_hess.c
  src/autocode/auto_3c2e.c src/autocode/auto_3c1e.c
  src/autocode/gaunt1.c src/autocode/breit1.c)

#option(BUILD_SHARED_LIBS "build shared libraries" 1)
option(ENABLE_EXAMPLE "build examples" 0)
option(ENABLE_TEST "build tests" 0)
if(QUICK_TEST)
  set(RUN_QUICK_TEST --quick)
endif()

#if(ENABLE_STATIC)
#  set(BUILD_SHARED_LIBS 0)
#endif()

add_library(cint ${cintSrc})
set_target_properties(cint PROPERTIES
  VERSION ${cint_VERSION}
  LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib
  POSITION_INDEPENDENT_CODE ON)
if (${BUILD_SHARED_LIBRARY})
	target_link_libraries(cint PRIVATE ${LIBC_INTERJECT})
	if (APPLE)
		set_target_properties(cint PROPERTIES LINK_FLAGS "-undefined dynamic_lookup")
	endif()
endif()
target_link_libraries(cint PRIVATE tgt::lapack)
if(QUADMATH_FOUND)
	target_link_libraries(cint PRIVATE quadmath)
endif()

install(TARGETS cint
	EXPORT "${PN}Targets"
	RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
	ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
	LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
	)
install(FILES ${PROJECT_BINARY_DIR}/include/cint.h DESTINATION ${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_INCLUDEDIR}/cint)

if(ENABLE_EXAMPLE)
  enable_language(Fortran)
  find_package(OpenMP)
  if(OPENMP_FOUND)
    set(HAVE_OPENMP 1)
  else ()
    set(OpenMP_C_FLAGS " ")
  endif()
  add_subdirectory(examples)
endif()

if(BUILD_SHARED_LIBS AND ENABLE_TEST)
  find_package(PythonInterp)
  message(STATUS "Found python  ${PYTHON_EXECUTABLE}")
  if(PYTHONINTERP_FOUND)
    enable_testing()
    add_test(NAME cinttest
      COMMAND ${PYTHON_EXECUTABLE} ${PROJECT_SOURCE_DIR}/testsuite/test_cint.py ${RUN_QUICK_TEST})
    add_test(NAME cint3c2etest
      COMMAND ${PYTHON_EXECUTABLE} ${PROJECT_SOURCE_DIR}/testsuite/test_3c2e.py ${RUN_QUICK_TEST})
  endif()
endif()

# <<<  Export Interface  >>>

target_compile_definitions(cint INTERFACE USING_${PN})
target_include_directories(cint INTERFACE
                           $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)

# <<<  Export Config  >>>

    # explicit "share" not "DATADIR" for CMake search path
set(CMAKECONFIG_INSTALL_DIR "share/cmake/${PN}")
configure_package_config_file(cmake/${PN}Config.cmake.in
                              "${CMAKE_CURRENT_BINARY_DIR}/${PN}Config.cmake"
                              INSTALL_DESTINATION ${CMAKECONFIG_INSTALL_DIR})
write_basic_package_version_file(${CMAKE_CURRENT_BINARY_DIR}/${PN}ConfigVersion.cmake
                                 VERSION ${${PN}_VERSION}
                                 COMPATIBILITY SameMajorVersion)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PN}Config.cmake
              ${CMAKE_CURRENT_BINARY_DIR}/${PN}ConfigVersion.cmake
        DESTINATION ${CMAKECONFIG_INSTALL_DIR})
install(EXPORT "${PN}Targets"
        NAMESPACE "${PN}::"
        DESTINATION ${CMAKECONFIG_INSTALL_DIR})

