cmake_minimum_required(VERSION 3.16)
project(class_public_tt LANGUAGES C CXX)

# -------------------- Build setup --------------------
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Default to Release if not set (Debug/Release/RelWithDebInfo/MinSizeRel)
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

# Match Makefile: -O3 in Release; -g and -fPIC always
add_compile_options($<$<CONFIG:Release>:-O3>)
add_compile_options(-g -fPIC)

# C++-only flags to mirror g++ --std=c++11 -fpermissive -Wno-write-strings
add_compile_options($<$<COMPILE_LANGUAGE:CXX>:-fpermissive> $<$<COMPILE_LANGUAGE:CXX>:-Wno-write-strings>)

# Define __CLASSDIR__ like the Makefile (pass the source dir path)
add_compile_definitions(__CLASSDIR__="${CMAKE_SOURCE_DIR}")

# Threads (Makefile had -pthread always)
find_package(Threads REQUIRED)

# Optional OpenMP (Makefile had it commented; default OFF here)
option(WITH_OPENMP "Enable OpenMP" OFF)
if(WITH_OPENMP)
    find_package(OpenMP REQUIRED)
endif()

# -------------------- Include dirs --------------------
# These mirror the Makefile INCLUDES
set(INC_DIRS
        ${CMAKE_SOURCE_DIR}/include
        ${CMAKE_SOURCE_DIR}/external/RecfastCLASS
        ${CMAKE_SOURCE_DIR}/external/heating
        ${CMAKE_SOURCE_DIR}/external/Halofit
        ${CMAKE_SOURCE_DIR}/external/HMcode
)

# HyRec toggle (Makefile: HYREC = external/HyRec2020 unless blank)
option(WITH_HYREC "Build with HyRec" ON)
set(HYREC_DIR ${CMAKE_SOURCE_DIR}/external/HyRec2020 CACHE PATH "HyRec path")
if(WITH_HYREC)
    add_compile_definitions(HYREC)
    list(APPEND INC_DIRS ${HYREC_DIR})
endif()

# -------------------- Utility: append first existing --------------------
function(append_first_existing OUT_VAR)
    foreach(cand IN LISTS ARGN)
        if(EXISTS "${CMAKE_SOURCE_DIR}/${cand}")
            list(APPEND ${OUT_VAR} "${CMAKE_SOURCE_DIR}/${cand}")
            set(${OUT_VAR} "${${OUT_VAR}}" PARENT_SCOPE)
            return()
        endif()
    endforeach()
    message(FATAL_ERROR "None of these files exist: ${ARGN}")
endfunction()

# -------------------- Sources --------------------
# Files the Makefile compiled as C++ (rule %.opp)
set(CXX_SOURCES)
append_first_existing(CXX_SOURCES tools/arrays.c)
append_first_existing(CXX_SOURCES tools/hyperspherical.c)
append_first_existing(CXX_SOURCES source/perturbations.c)
append_first_existing(CXX_SOURCES source/primordial.c)
append_first_existing(CXX_SOURCES source/transfer.c)
append_first_existing(CXX_SOURCES source/harmonic.c)
append_first_existing(CXX_SOURCES external/HMcode/hmcode.c)

# Pure C sources (rule %.o)
set(C_SOURCES)
# tools/
append_first_existing(C_SOURCES tools/growTable.c)
append_first_existing(C_SOURCES tools/dei_rkck.c)
append_first_existing(C_SOURCES tools/sparse.c)
append_first_existing(C_SOURCES tools/evolver_rkck.c)
append_first_existing(C_SOURCES tools/evolver_ndf15.c)
append_first_existing(C_SOURCES tools/parser.c)
append_first_existing(C_SOURCES tools/quadrature.c)
append_first_existing(C_SOURCES tools/common.c source/common.c) # either location depending on version
append_first_existing(C_SOURCES source/trigonometric_integrals.c tools/trigonometric_integrals.c)

# source/
append_first_existing(C_SOURCES source/input.c)
append_first_existing(C_SOURCES source/background.c)
append_first_existing(C_SOURCES source/thermodynamics.c)
append_first_existing(C_SOURCES source/fourier.c)
append_first_existing(C_SOURCES source/lensing.c)
append_first_existing(C_SOURCES source/distortions.c)
append_first_existing(C_SOURCES source/output.c)

# main
set(MAIN_SRC ${CMAKE_SOURCE_DIR}/main/class.c)

# externals (always C in Makefile)
append_first_existing(C_SOURCES external/RecfastCLASS/wrap_recfast.c)
append_first_existing(C_SOURCES external/heating/injection.c)
append_first_existing(C_SOURCES external/heating/noninjection.c)
append_first_existing(C_SOURCES external/Halofit/halofit.c)

# HyRec (optional)
if(WITH_HYREC)
    append_first_existing(C_SOURCES external/HyRec2020/hyrectools.c)
    append_first_existing(C_SOURCES external/HyRec2020/helium.c)
    append_first_existing(C_SOURCES external/HyRec2020/hydrogen.c)
    append_first_existing(C_SOURCES external/HyRec2020/history.c)
    append_first_existing(C_SOURCES external/HyRec2020/wrap_hyrec.c)
    append_first_existing(C_SOURCES external/HyRec2020/energy_injection.c)
endif()

# Treat .c files listed in CXX_SOURCES as C++
foreach(src IN LISTS CXX_SOURCES)
    set_source_files_properties(${src} PROPERTIES LANGUAGE CXX)
endforeach()

# -------------------- Targets --------------------
# Static library libclass.a (exclude main/class.c from the lib)
set(LIB_SOURCES ${C_SOURCES} ${CXX_SOURCES})
list(REMOVE_ITEM LIB_SOURCES ${MAIN_SRC})

add_library(class STATIC ${LIB_SOURCES})
target_include_directories(class PRIVATE ${INC_DIRS})
target_compile_options(class PRIVATE -pthread)  # Makefile had OMPFLAG = -pthread even without OpenMP
target_link_libraries(class PRIVATE Threads::Threads)
if(UNIX AND NOT APPLE)
    target_link_libraries(class PRIVATE m)
endif()
if(WITH_OPENMP)
    target_link_libraries(class PRIVATE OpenMP::OpenMP_C OpenMP::OpenMP_CXX)
endif()

# Executable "class" linking the library
add_executable(class_cli ${MAIN_SRC})
target_include_directories(class_cli PRIVATE ${INC_DIRS})
target_compile_options(class_cli PRIVATE -pthread)
target_link_libraries(class_cli PRIVATE class Threads::Threads)
set_target_properties(class_cli PROPERTIES OUTPUT_NAME "class")

# -------------------- Optional tests (match Makefile targets) --------------------
option(BUILD_TESTS "Build CLASS test programs" OFF)
if(BUILD_TESTS)
    add_executable(test_loops test/test_loops.c)
    target_link_libraries(test_loops PRIVATE class)

    add_executable(test_loops_omp test/test_loops_omp.c)
    target_link_libraries(test_loops_omp PRIVATE class)

    add_executable(test_harmonic test/test_harmonic.c)
    target_link_libraries(test_harmonic PRIVATE class)

    add_executable(test_transfer test/test_transfer.c)
    target_link_libraries(test_transfer PRIVATE class)

    add_executable(test_fourier test/test_fourier.c)
    target_link_libraries(test_fourier PRIVATE class)

    add_executable(test_perturbations test/test_perturbations.c)
    target_link_libraries(test_perturbations PRIVATE class)

    add_executable(test_thermodynamics test/test_thermodynamics.c)
    target_link_libraries(test_thermodynamics PRIVATE class)

    add_executable(test_background test/test_background.c)
    target_link_libraries(test_background PRIVATE class)

    # The Makefile builds hyperspherical as C for this target
    add_executable(test_hyperspherical test/test_hyperspherical.c tools/hyperspherical.c)
    set_source_files_properties(${CMAKE_SOURCE_DIR}/tools/hyperspherical.c PROPERTIES LANGUAGE C)
    if(UNIX AND NOT APPLE)
        target_link_libraries(test_hyperspherical PRIVATE m)
    endif()
endif()

# -------------------- Python 'classy' (pip install) --------------------
# Mirrors the 'classy' Makefile target (without uninstall cleanup logic)
option(BUILD_CLASSY "Add 'classy' pip-install helper target" ON)
if(BUILD_CLASSY)
    find_package(Python3 COMPONENTS Interpreter QUIET)
    if(Python3_Interpreter_FOUND)
        add_custom_target(classy
                COMMAND ${CMAKE_COMMAND} -E env CC=${CMAKE_C_COMPILER} ${Python3_EXECUTABLE} -m pip install .
                WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/python
                USES_TERMINAL
                COMMENT "Installing Python package 'classy' with pip"
        )
    endif()
endif()

# -------------------- Small convenience: svnversion.h stub if missing --------------------
# Some versions expect include/svnversion.h to exist. Create a tiny stub if absent.
if(NOT EXISTS "${CMAKE_SOURCE_DIR}/include/svnversion.h")
    file(WRITE "${CMAKE_SOURCE_DIR}/include/svnversion.h" "#define SVN_REVISION \"local\"\n")
endif()
