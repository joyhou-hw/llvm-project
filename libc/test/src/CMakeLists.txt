function(add_fp_unittest name)
  cmake_parse_arguments(
    "MATH_UNITTEST"
    "NEED_MPFR;UNIT_TEST_ONLY;HERMETIC_TEST_ONLY" # Optional arguments
    "" # Single value arguments
    "LINK_LIBRARIES" # Multi-value arguments
    ${ARGN}
  )

  if(MATH_UNITTEST_NEED_MPFR)
    if(NOT LIBC_TESTS_CAN_USE_MPFR)
      message(VERBOSE "Math test ${name} will be skipped as MPFR library is not available.")
      return()
    endif()
  endif()

  if(MATH_UNITTEST_HERMETIC_TEST_ONLY)
    set(test_type HERMETIC_TEST_ONLY)
  elseif(MATH_UNITTEST_UNIT_TEST_ONLY)
    set(test_type UNIT_TEST_ONLY)
  endif()
  if(MATH_UNITTEST_NEED_MPFR)
    if(MATH_UNITTEST_HERMETIC_TEST_ONLY)
      message(FATAL "Hermetic math test cannot require MPFR.")
    endif()
    set(test_type UNIT_TEST_ONLY)
    list(APPEND MATH_UNITTEST_LINK_LIBRARIES libcMPFRWrapper libc_math_test_utils -lmpfr -lgmp)
  endif()
  list(APPEND MATH_UNITTEST_LINK_LIBRARIES LibcFPTestHelpers)

  add_libc_test(
    ${name}
    ${test_type}
    LINK_LIBRARIES "${MATH_UNITTEST_LINK_LIBRARIES}"
    "${MATH_UNITTEST_UNPARSED_ARGUMENTS}"
  )
endfunction(add_fp_unittest)

add_subdirectory(__support)
add_subdirectory(ctype)
add_subdirectory(errno)
add_subdirectory(fenv)
add_subdirectory(inttypes)
add_subdirectory(math)
add_subdirectory(search)
add_subdirectory(stdbit)
add_subdirectory(stdio)
add_subdirectory(stdlib)
add_subdirectory(string)
add_subdirectory(wchar)

if(${LIBC_TARGET_OS} STREQUAL "linux")
  add_subdirectory(fcntl)
  add_subdirectory(sched)
  add_subdirectory(sys)
  add_subdirectory(termios)
  add_subdirectory(unistd)
endif()

if(NOT LLVM_LIBC_FULL_BUILD)
  return()
endif()

add_subdirectory(assert)
add_subdirectory(compiler)
add_subdirectory(dirent)
add_subdirectory(network)
add_subdirectory(setjmp)
add_subdirectory(signal)
add_subdirectory(spawn)
add_subdirectory(time)

if(${LIBC_TARGET_OS} STREQUAL "linux")
  add_subdirectory(pthread)
endif()

if(LLVM_RUNTIMES_BUILD OR LIBC_HDRGEN_EXE)
  # The public API test below uses tablegen to generate the test
  # source file. Since tablegen is not available during a runtimes
  # build, we will skip the test.
  # If a different libc-hdrgen binary is being used, then also we
  # skip the api-test as we cannot generate the test source file.
  return()
endif()

set(public_test ${CMAKE_CURRENT_BINARY_DIR}/public_api_test.cpp)

set(entrypoints_name_list "")
foreach(entry IN LISTS TARGET_LLVMLIBC_ENTRYPOINTS)
  get_target_property(entry_name ${entry} "ENTRYPOINT_NAME")
  list(APPEND entrypoints_name_list ${entry_name})
endforeach()

# TODO: Remove these when they are added to the TableGen.
list(REMOVE_ITEM entrypoints_name_list "__assert_fail" "__errno_location")
list(TRANSFORM entrypoints_name_list PREPEND "-e=")

file(GLOB spec_files ${LIBC_SOURCE_DIR}/spec/*.td)

# Generate api test souce code.
add_custom_command(
  OUTPUT ${public_test}
  COMMAND $<TARGET_FILE:libc-prototype-testgen> -o ${public_test}
          ${entrypoints_name_list}
          -I ${LIBC_SOURCE_DIR}
          ${LIBC_SOURCE_DIR}/config/${LIBC_TARGET_OS}/api.td

  DEPENDS ${LIBC_SOURCE_DIR}/config/${LIBC_TARGET_OS}/api.td ${spec_files}
          libc-prototype-testgen ${TARGET_PUBLIC_HEADERS}
          ${LIBC_TARGET}
)

add_custom_target(libc-api-test)
add_dependencies(check-libc libc-api-test)

set(
  allocator_entrypoints
    libc.src.stdlib.malloc
    libc.src.stdlib.calloc
    libc.src.stdlib.realloc
    libc.src.stdlib.aligned_alloc
    libc.src.stdlib.free
)
set(api-test-entrypoints ${TARGET_LLVMLIBC_ENTRYPOINTS})
list(REMOVE_ITEM api-test-entrypoints ${allocator_entrypoints})
add_integration_test(
  api-test
  SUITE
    libc-api-test
  SRCS
    ${public_test}
  DEPENDS
    ${api-test-entrypoints}
)

if(COMPILER_RESOURCE_DIR AND LLVM_LIBC_ENABLE_LINTING)
  add_custom_target(
    libc-api-test-tidy
    VERBATIM
    COMMAND ${LLVM_LIBC_CLANG_TIDY} --system-headers
      --checks=-*,llvmlibc-restrict-system-libc-headers
      "--extra-arg=-resource-dir=${COMPILER_RESOURCE_DIR}"
      --header-filter=.*
      --warnings-as-errors=llvmlibc-*
      "-config={CheckOptions: [{key: llvmlibc-restrict-system-libc-headers.Includes, value: '-*, linux/*, asm/*.h, asm-generic/*.h'}]}"
      --quiet
      -p ${PROJECT_BINARY_DIR}
      ${public_test}
    DEPENDS
      clang-tidy ${public_test}
  )
  add_dependencies(libc-api-test libc-api-test-tidy)
endif()
