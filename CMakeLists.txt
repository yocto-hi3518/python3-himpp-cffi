cmake_minimum_required(VERSION 3.8 FATAL_ERROR)

project(himpp-cffi)

option(PYTHON "Python binary.")
option(PYTHON_DISTUTILS_INSTALL_ARGS "Python distutils installtion arguments.")

#option(HIMPP_INCLUDE_PATH "HiMPP include path")


find_path(HIMPP_INCLUDE_PATH
          PATHS /usr/include/hi3518v100mpp
          NAMES hi_common.h)

include_directories(SYSTEM ${HIMPP_INCLUDE_PATH})


if (NOT DEFINED ${PYTHON})
    message("Auto detcting python from environment...")
    include(FindPythonInterp)
    set(PYTHON ${PYTHON_EXECUTABLE})
endif(NOT DEFINED ${PYTHON})


message("Python bin: ${PYTHON}")

execute_process(COMMAND "${PYTHON}" -c
                        "import cffi
"                       "test = cffi.FFI()"
                RESULT_VARIABLE CFFI_TEST_RESULT)

if(NOT "${CFFI_TEST_RESULT}" STREQUAL "0")
    message(FATAL_ERROR "Unable to import python CFFI.
"                       "Please install with pip install cffi")
    set(CFFI_FOUND TRUE)
endif()


execute_process(
   COMMAND "${PYTHON}" -c "if True:
            from distutils import sysconfig as sc
            print(sc.get_python_lib(prefix='', plat_specific=True))"
   OUTPUT_VARIABLE PYTHON_SITE
   OUTPUT_STRIP_TRAILING_WHITESPACE)

function(CFFIPackageFromHeader HEADER PACKAGE_NAME VERSION)

set(CFFI_PYTHON_GEN_${PACKAGE_NAME}  
)


    add_custom_target(cffi_header_${PACKAGE_NAME}
                      COMMAND ${CMAKE_C_COMPILER} ${CFFI_CFLAGS} -E ${CMAKE_CURRENT_SOURCE_DIR}/${HEADER} | grep -v '^\#' > ${CMAKE_CURRENT_BINARY_DIR}/cffi_gen.h)

    add_custom_target(cffi_gen_${PACKAGE_NAME} ALL 
                      DEPENDS cffi_header_${PACKAGE_NAME}
                      COMMAND "${PYTHON}" -c "\"import" "cffi;"
                      "ffibuilder" "=" "cffi.FFI();"
                      "ffibuilder.set_source('${PACKAGE_NAME}',None);"
                      "f" "=" "open('${CMAKE_CURRENT_BINARY_DIR}/cffi_gen.h','r');"
                      "header_file" "=" "f.read();"
                      "ffibuilder.cdef(header_file,override=True);"
                      "ffibuilder.compile(verbose=True)\"")

    configure_file(setup.template setup.py
                    NEWLINE_STYLE UNIX)

endfunction(CFFIPackageFromHeader)


set(CFFI_CFLAGS "-DCFFI=1" "-I${HIMPP_INCLUDE_PATH}")

CFFIPackageFromHeader("himpp.h" "himpp" "0.1")

get_cmake_property(_variableNames VARIABLES)
list (SORT _variableNames)
foreach (_variableName ${_variableNames})
    message(STATUS "${_variableName}=${${_variableName}}")
endforeach()
