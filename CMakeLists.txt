cmake_minimum_required(VERSION 3.0.0)
set(CMAKE_BUILD_TYPE Release CACHE STRING
  "Choose the type of build: None Debug Release RelWithDebInfo MinSizeRel")

project(JPEG VERSION 9.0 LANGUAGES C)
set(srcdir ${JPEG_SOURCE_DIR})

#-----------------------------------------------------------------------------
# Include all the necessary files for macros
#-----------------------------------------------------------------------------
include (CheckFunctionExists) # check_function_exists
include (CheckIncludeFile) # check_include_file
include (CheckSymbolExists) # check_symbol_exists
if(NOT IOS)
  include (CheckTypeSize) # check_type_size
endif()
include (CheckCSourceCompiles) # check_c_source_compiles
include (CheckCCompilerFlag) # check_c_compiler_flag

function(check_c flag result_var)
  set(CMAKE_REQUIRED_FLAGS ${flag}) # to test linking as well
  check_c_compiler_flag("" ${result_var})
endfunction()

# compiler flags should be set before any HAVE_ checks
if( CMAKE_COMPILER_IS_GNUCC )
  # check_c_compiler_flag(-Wall  gcc_Wall)
  check_c(-Wall  gcc_Wall)
  if( gcc_Wall )
    set(CWARNFLAGS "${CWARNFLAGS} -Wall")
  endif()
  check_c("${CWARNFLAGS} -Wno-obsolete" gcc_Wno_obsolete)
  if( gcc_Wno_obsolete )
    set(CWARNFLAGS "${CWARNFLAGS} -Wno-obsolete")
  endif()
  # linker flags
  check_c("-Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/empty_test.map"
    gcc_Wl_version_script)
  check_c("-Wl,-no-undefined" gcc_no_undefined)
elseif( MSVC )
  add_definitions("-D_CRT_SECURE_NO_WARNINGS") # disable non-secure API warnings
else()
  # per cmake 2.8.8 documentation the following is not portable
  message(STATUS "C Compiler guessed by cmake: ${CMAKE_C_COMPILER_ID}")
  if( CMAKE_C_COMPILER_ID MATCHES "Intel" )
    check_c(-Wall intel_Wall)
    if( intel_Wall )
      set(CWARNFLAGS "${CWARNFLAGS} -Wall")
    endif()
  elseif( CMAKE_C_COMPILER_ID MATCHES "Clang" )
    check_c(-Wall clang_Wall)
    if( clang_Wall )
      set(CWARNFLAGS "${CWARNFLAGS} -Wall")
    endif()
  else()
    message(STATUS "Not supported compiler. No extra flags set.")
  endif()
endif()

set( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${CWARNFLAGS}" )

check_include_file(stddef.h HAVE_STDDEF_H)
check_include_file(stdlib.h HAVE_STDLIB_H)
check_include_file(locale.h HAVE_LOCALE_H)
check_include_file(string.h HAVE_STRING_H)
check_include_file(strings.h HAVE_STRINGS_H)
if( HAVE_STRINGS_H AND NOT HAVE_STRING_H )
  set(NEED_BSD_STRINGS 1)
endif()
if(IOS)
  set(HAVE_UNSIGNED_CHAR 1)
  set(HAVE_UNSIGNED_SHORT 1)
  set(HAVE_SYS_TYPES_H 1)
else()
  check_type_size(size_t SIZE_T) # -> HAVE_SIZE_T
  if( NOT HAVE_SIZE_T )
    message(WARNING "Type size_t is not defined in any of the usual places.
    Try putting 'typedef unsigned int size_t;' in jconfig.h."
    )
  endif()
  check_type_size("unsigned char" UNSIGNED_CHAR) # -> HAVE_UNSIGNED_CHAR
  check_type_size("unsigned short" UNSIGNED_SHORT) # -> HAVE_UNSIGNED_SHORT
  check_type_size(char CHAR) # -> HAVE_CHAR
  if( NOT HAVE_CHAR )
    message(ERROR "Major BUG: no 'char' in C.")
  endif()
  if( NOT (CHAR EQUAL 1) )
    message(WARNING  "Hmm, it seems char is not eight bits wide on your machine. "
    "I fear the JPEG software will not work at all.")
  endif()
endif()

check_symbol_exists(tmpfile stdio.h HAVE_TMPFILE_SYMBOL)
check_function_exists(tmpfile HAVE_TMPFILE_FUNC)

check_symbol_exists(mktemp stdlib.h HAVE_MKTEMP_SYMBOL)
check_function_exists(mktemp HAVE_MKTEMP_FUNC)

if( NOT ( HAVE_MKTEMP_SYMBOL AND HAVE_MKTEMP_FUNC AND HAVE_MKTEMP_DEFINITION) )
  set(NO_MKTEMP 1)
endif()

# See if compiler supports prototypes.
check_c_source_compiles(
"int testfunction (int arg1, int * arg2); /* check prototypes */
struct methods_struct { /* check method-pointer declarations */
  int (*error_exit) (char *msgtext);
  int (*trace_message) (char *msgtext);
  int (*another_method) (void);
};
int testfunction (int arg1, int * arg2) /* check definitions */
{ return arg2[arg1]; }
int test2function (void) /* check void arg list */
{ return 0; }
int main(int argc, char **argv) {return 0;}
"
HAVE_PROTOTYPES
)
if( NOT HAVE_PROTOTYPES )
  message(WARNING "Your compiler does not seem to know about function prototypes.
    Perhaps it needs a special switch to enable ANSI C mode.")
endif()

# checking if C compiler has void type
check_c_source_compiles(
"/* Caution: a C++ compiler will insist on valid prototypes */
typedef void * void_ptr; /* check void * */
#ifdef HAVE_PROTOTYPES /* check ptr to function returning void */
typedef void (*void_func) (int a, int b);
#else
typedef void (*void_func) ();
#endif

#ifdef HAVE_PROTOTYPES /* check void function result */
void test3function (void_ptr arg1, void_func arg2)
#else
void test3function (arg1, arg2)
  void_ptr arg1;
  void_func arg2;
#endif
{
  char * locptr = (char *) arg1; /* check casting to and from void * */
  arg1 = (void *) locptr;
  (*arg2) (1, 2);  /* check call of fcn returning void */
}
int main() { return 0; }
"
HAVE_VOID
)
if( NOT HAVE_VOID )
  message(WARNING "Defining 'void' as 'char' because the compiler is archaic")
  set(VOID char)
endif()

# Check for non-broken inline under various spellings
check_c_source_compiles(
  "inline int foo() {return 0;} int bar() {return foo();} int main() {return bar();}"
  HAVE_INLINE
)
check_c_source_compiles(
  "__inline__ int foo() {return 0;} int bar() {return foo();} int main() {return bar();}"
  HAVE___INLINE__
)
check_c_source_compiles(
  "__inline int foo() {return 0;} int bar() {return foo();} int main() {return bar();}"
  HAVE___INLINE
)
if( HAVE_INLINE )
  set(INLINE inline)
elseif(HAVE___INLINE__)
  set(INLINE __inline__)
elseif(HAVE___INLINE)
  set(INLINE __inline)
endif()

# checking fo C has const
check_c_source_compiles(
  "const char *ch; const int x=17; int func(const int *i) {return x + (*i);} int main() {return func(&x);}"
  HAVE_C_CONST
)
if( NOT HAVE_C_CONST )
  message(WARNING "C compiler does not have 'const'. That is an archaic compiler, UPGRADE!")
  set(CONST )
endif()

# We cannot check for bogus warnings, but at least we can check for errors
check_c_source_compiles(
  "typedef struct undefined_structure * undef_struct_ptr; int main(){return 0;}"
  INCOMPLETE_TYPES_NOT_BROKEN
)
if( NOT INCOMPLETE_TYPES_NOT_BROKEN )
  set(INCOMPLETE_TYPES_BROKEN 1)
  message(WARNING "Compiler does not support pointers to unspecified structures.")
endif()

# Test whether global names are unique to at least 15 chars
check_c_source_compiles(
"int possibly_duplicate_function () { return 0; }
int possibly_dupli_function () { return 1; }
int main() {return 0;}
"
HAVE_LONG_EXTERNAL_NAMES
)
if( NOT HAVE_LONG_EXTERNAL_NAMES )
  message(WARNING "C compiler is archaic, it does not undestand long function names")
  set(NEED_SHORT_EXTERNAL_NAMES 1)
endif()

check_include_file(limits.h HAVE_LIMITS_H)
if(NOT HAVE_LIMITS_H)
  message(FATAL_ERROR "limits.h is missing")
endif()
# checking to see if char is signed
check_c_source_compiles(
"#include <limits.h>
#if CHAR_MIN >= 0
#  error \"char is unsigned\"
#endif
int main() {return 0;}
"
CHAR_IS_SIGNED
)
if( CHAR_IS_SIGNED )
  message(STATUS "Assuming that char is signed on target machine. "
      "If it is unsigned, this will be a little bit inefficient.")
else()
  message(WARNING "Assuming that char is UNsigned on target machine.")
  set(CHAR_IS_UNSIGNED 1)
endif()

# checking to see if right shift is signed
check_c_source_compiles(
"#define NTST1 -0x7F7E80CL
#define NVAL -0x7F7E80B1L
#if  ((NVAL)>>4) != NTST1
#error \"right shift is unsigned\"
#endif
int main() {return 0;}
"
RIGHT_SHIFT_IS_SIGNED
)
if( RIGHT_SHIFT_IS_SIGNED )
  message(STATUS "Assuming that right shift is signed on target machine.")
else()
  set(RIGHT_SHIFT_IS_UNSIGNED 1)
  message(WARNING "Broken compiler shifts signed values as an unsigned shift.")
endif()

# superficial check to see if fopen accepts b spec
# assuming that it does, it is not used by the library anyway
check_c_source_compiles(
"
#include <stdio.h>
int main() {
  if (fopen(\"conftestdata\", \"wb\") != NULL)
    exit(0);
  exit(1);
}
"
HAVE_B_MODE_IN_FOPEN
)
if( NOT HAVE_B_MODE_IN_FOPEN )
  set(DONT_USE_B_MODE 1)
  message(WARNING "Do not have 'b' mode in fopen")
endif()


########### next target ###############
## Sources to build library
set(LIBSOURCES
  jaricom.c jcapimin.c jcapistd.c jcarith.c jccoefct.c jccolor.c
  jcdctmgr.c jchuff.c jcinit.c jcmainct.c jcmarker.c jcmaster.c
  jcomapi.c jcparam.c jcprepct.c jcsample.c jctrans.c jdapimin.c
  jdapistd.c jdarith.c jdatadst.c jdatasrc.c jdcoefct.c jdcolor.c
  jddctmgr.c jdhuff.c jdinput.c jdmainct.c jdmarker.c jdmaster.c
  jdmerge.c jdpostct.c jdsample.c jdtrans.c jerror.c jfdctflt.c
  jfdctfst.c jfdctint.c jidctflt.c jidctfst.c jidctint.c jquant1.c
  jutils.c jmemmgr.c jmemnobs.c
)

# System dependent sources
set(SYSDEPSOURCES
  jmemansi.c jmemname.c jmemnobs.c jmemdos.c jmemmac.c
)

set(jconfig_h "${CMAKE_BINARY_DIR}/include/jconfig.h")
configure_file(jconfig-cmake.cfg ${jconfig_h})

# Headers which are installed to support the library
set(INSTINCLUDES  jerror.h jmorecfg.h jpeglib.h ${jconfig_h})
set_source_files_properties(${jconfig_h} PROPERTIES GENERATED TRUE)
source_group(Interface FILES ${INSTINCLUDES})

# Headers which are not installed
set(OTHERINCLUDES cderror.h cdjpeg.h jdct.h jinclude.h
  jmemsys.h jpegint.h
  jversion.h transupp.h
)

# Manual pages (Automake uses 'MANS' for itself)
set(DISTMANS cjpeg.1 djpeg.1 jpegtran.1 rdjpgcom.1 wrjpgcom.1)

# Other documentation files
set(DOCS README "install.txt" usage.txt wizard.txt example.c libjpeg.txt
  structure.txt coderules.txt filelist.txt change.log)

# Miscellaneous support files
set(OTHERFILES jconfig.txt libjpeg.map)

# Test support files
set(TESTFILES testorig.jpg testimg.ppm testimg.bmp testimg.jpg testprog.jpg
  testimgp.jpg)

set(jpeg_LIB_SRCS ${LIBSOURCES} jquant2.c ${INSTINCLUDES} ${OTHERINCLUDES})
add_library(jpeg ${jpeg_LIB_SRCS})

set_target_properties(jpeg PROPERTIES VERSION 9.0 SOVERSION 9
  PUBLIC_HEADER "${INSTINCLUDES}"
  DEFINE_SYMBOL "JPEG_DLL_EXPORTS"
  FOLDER Libraries
)
target_include_directories(jpeg PUBLIC
  $<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/include>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
  "$<INSTALL_INTERFACE:include>"
)

if(gcc_Wl_version_script)
  # TODO: this does not seem necessary as libjpeg.map is basically empty
  set_target_properties(jpeg PROPERTIES LINK_FLAGS
    "-Wl,--version-script=${srcdir}/libjpeg.map")
endif()
if(gcc_no_undefined)
  set_target_properties(jpeg PROPERTIES LINK_FLAGS "-Wl,-no-undefined")
endif()


########### Executables ###############

set(cjpeg_SRCS
    cjpeg.c
    rdppm.c
    rdgif.c
    rdtarga.c
    rdrle.c
    rdbmp.c
    rdswitch.c
    cdjpeg.c)
add_executable(cjpeg ${cjpeg_SRCS})
target_link_libraries(cjpeg jpeg)

set(djpeg_SRCS
    djpeg.c
    wrppm.c
    wrgif.c
    wrtarga.c
    wrrle.c
    wrbmp.c
    rdcolmap.c
    cdjpeg.c)
add_executable(djpeg ${djpeg_SRCS})
target_link_libraries(djpeg jpeg)

set(jpegtran_SRCS
    jpegtran.c
    rdswitch.c
    cdjpeg.c
    transupp.c)
add_executable(jpegtran ${jpegtran_SRCS})
target_link_libraries(jpegtran jpeg)

add_executable(rdjpgcom rdjpgcom.c)
target_include_directories(rdjpgcom PRIVATE "${CMAKE_BINARY_DIR}/include")


add_executable(wrjpgcom wrjpgcom.c)
target_include_directories(wrjpgcom PRIVATE "${CMAKE_BINARY_DIR}/include")

set_target_properties(cjpeg djpeg jpegtran rdjpgcom wrjpgcom  PROPERTIES FOLDER
  Executables/jpeg)

########## setup configuration headers ###########

set(targets_export_name "${PROJECT_NAME}Targets")

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  "${CMAKE_BINARY_DIR}/lib/cmake/JPEGConfigVersion.cmake"
  COMPATIBILITY AnyNewerVersion
)

set(INCLUDE_INSTALL_DIR "include")

# Use:
# * targets_export_name
configure_package_config_file("cmake/Config.cmake.in"
  "${CMAKE_BINARY_DIR}/lib/cmake/JPEGConfig.cmake"
  INSTALL_DESTINATION lib/cmake/JPEG
  PATH_VARS INCLUDE_INSTALL_DIR
)

########### install files ###############

if(IOS)
  set(libs_to_install jpeg)
else()
  set(libs_to_install jpeg cjpeg djpeg jpegtran rdjpgcom wrjpgcom)
endif()

install(TARGETS ${libs_to_install}
  EXPORT ${targets_export_name}
  LIBRARY DESTINATION lib COMPONENT Runtime
  ARCHIVE DESTINATION lib COMPONENT Development
  RUNTIME DESTINATION bin COMPONENT Runtime
  PUBLIC_HEADER DESTINATION include COMPONENT Development
  BUNDLE DESTINATION bin COMPONENT Runtime
)
install(EXPORT ${targets_export_name} NAMESPACE "JPEG::" DESTINATION lib/cmake/JPEG)
install(FILES "${CMAKE_BINARY_DIR}/lib/cmake/JPEGConfigVersion.cmake"
  "${CMAKE_BINARY_DIR}/lib/cmake/JPEGConfig.cmake"
  DESTINATION lib/cmake/JPEG)

################### TESTS
enable_testing()

if(ANDROID OR IOS)
  add_test(
      NAME DummyTest
      COMMAND "${CMAKE_COMMAND}" -E echo "Skipping tests (Android/iOS)"
  )
else()
  add_test(NAME djpeg-test1 COMMAND
    djpeg -dct int -ppm -outfile testout.ppm  ${srcdir}/testorig.jpg)
  add_test(djpeg-diff-test1 ${CMAKE_COMMAND} -E compare_files ${srcdir}/testimg.ppm testout.ppm)
  set_tests_properties(djpeg-diff-test1 PROPERTIES DEPENDS djpeg-test1)

  add_test(NAME djpeg-test2 COMMAND
    djpeg -dct int -bmp -colors 256 -outfile testout.bmp  ${srcdir}/testorig.jpg)

  set(
      test_opts
      NAME djpeg-diff-test2
      COMMAND ${CMAKE_COMMAND} -E compare_files ${srcdir}/testimg.bmp testout.bmp
  )
  if(NOT MSVC OR MSVC_VERSION VERSION_LESS 1600) # Visual Studio 10 2010
    add_test(${test_opts})
  else()
    # Since Visual Studio 10 2010 Release version produce slightly different
    # final image so it doesn't match etalon:
    # * https://ci.appveyor.com/project/ruslo/jpeg/build/1.0.2
    add_test(${test_opts} CONFIGURATIONS Debug)
  endif()

  set_tests_properties(djpeg-diff-test2 PROPERTIES DEPENDS djpeg-test2)

  add_test(NAME cjpeg-test1
    COMMAND cjpeg -dct int -outfile testout.jpg  ${srcdir}/testimg.ppm)
  add_test(cjpeg-diff-test1
    ${CMAKE_COMMAND} -E compare_files  ${srcdir}/testimg.jpg testout.jpg)
  set_tests_properties(cjpeg-diff-test1 PROPERTIES DEPENDS cjpeg-test1)

  add_test(NAME djpeg-test3 COMMAND
    djpeg -dct int -ppm -outfile testoutp.ppm ${srcdir}/testprog.jpg)
  add_test(djpeg-diff-test3
    ${CMAKE_COMMAND} -E compare_files ${srcdir}/testimg.ppm testoutp.ppm)
  set_tests_properties(djpeg-diff-test3 PROPERTIES DEPENDS djpeg-test3)

  add_test(NAME cjpeg-test2 COMMAND
    cjpeg -dct int -progressive -opt -outfile testoutp.jpg ${srcdir}/testimg.ppm)
  add_test(cjpeg-diff-test2
    ${CMAKE_COMMAND} -E compare_files ${srcdir}/testimgp.jpg testoutp.jpg)
  set_tests_properties(cjpeg-diff-test2 PROPERTIES DEPENDS cjpeg-test2)

  add_test(NAME jpegtran-test1 COMMAND
    jpegtran -outfile testoutt.jpg ${srcdir}/testprog.jpg)
  add_test(jpegtran-diff-test1
    ${CMAKE_COMMAND} -E compare_files ${srcdir}/testorig.jpg testoutt.jpg)
  set_tests_properties(jpegtran-diff-test1 PROPERTIES DEPENDS jpegtran-test1)

  set_tests_properties(jpegtran-diff-test1 jpegtran-test1 cjpeg-diff-test2
    cjpeg-test2 djpeg-diff-test3  djpeg-test3 cjpeg-diff-test1  cjpeg-test1
    djpeg-diff-test2 djpeg-test2 djpeg-diff-test1  djpeg-test1
    PROPERTIES COST 15 LABELS "15_sec;jpeg"
  )
endif()

include(InstallRequiredSystemLibraries)
include(CPack)
