set(CBE_TEST_DIRECTORIES <to-be-add>)

set(CBE_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/..")
set(CBE_BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}/..")
set(LLVM_SHLIBEXT "${CMAKE_SHARED_MODULE_SUFFIX}")

if( CBE_SOURCE_DIR STREQUAL "${LLVM_MAIN_SRC_DIR}/tools/cbe" )

  set(LLVM_LIT ${LLVM_TOOLS_BINARY_DIR}/llvm-lit)
  set(CBE_TEST_DEPS llvm-config opt colander FileCheck not)

  set(LLVM_BINARY_DIR "${LLVM_BINARY_DIR}")
  set(LLVM_TOOLS_DIR "${LLVM_TOOLS_BINARY_DIR}")
  set(LLVM_LIBS_DIR "${LLVM_BINARY_DIR}/lib${LLVM_LIBDIR_SUFFIX}")
  set(CBE_LIB_DIR "${LLVM_LIBS_DIR}")
  set(CBE_TOOLS_DIR "${LLVM_TOOLS_DIR}")

  add_lit_testsuite(check-cbe "Running cbe regression tests"
    ${CMAKE_CURRENT_BINARY_DIR}
    PARAMS cbe_site_config=${CMAKE_CURRENT_BINARY_DIR}/lit.site.cfg
    DEPENDS ${CBE_TEST_DEPS}
    )

  set_target_properties(check-cbe PROPERTIES FOLDER "CBE")

  # Run cbe-check-format as part of cbe-check only if we are compiling with
  # clang, so clang-format is availbale.
  # if (TARGET clang-format) would be preferable, but this target is only added
  # after CBE, i.e. there is no such target yet at this point. The CMake cache
  # entry LLVM_TOOL_CLANG_BUILD will only be defined after clang has been
  # configured at least once, i.e. format will be checked only after a rerun of
  # CMake's configure.
  if (LLVM_TOOL_CLANG_BUILD)
    add_dependencies(check-cbe cbe-check-format)
  endif ()

else ( CBE_SOURCE_DIR STREQUAL "${LLVM_MAIN_SRC_DIR}/tools/cbe" )

  # We are building cbe out of tree, adjust the settings.
  # FIXME: FileCheck is not available in llvm install directory at the moment.
  set(LLVM_LIT ${LLVM_BINARY_DIR}/bin/llvm-lit)
  set(CBE_TEST_DEPS colander)

  set(LLVM_TOOLS_DIR "${LLVM_BINARY_DIR}/bin")
  set(LLVM_LIBS_DIR "${LLVM_BINARY_DIR}/lib")
  set(CBE_LIB_DIR "${CBE_BINARY_DIR}/lib")
  set(CBE_TOOLS_DIR "${CBE_BINARY_DIR}/bin")

  include(FindPythonInterp)
  if(PYTHONINTERP_FOUND)
    option(CBE_TEST_DISABLE_BAR "Run CBE tests with --no-progress-bar" OFF)
    set(CBE_TEST_EXTRA_ARGS)
    if (MSVC OR XCODE OR CBE_TEST_DISABLE_BAR)
      set(CBE_TEST_EXTRA_ARGS "--no-progress-bar")
    endif()

    option(CBE_TEST_USE_VG "Run CBE tests under Valgrind" OFF)
    if(CBE_TEST_USE_VG)
      set(CBE_TEST_EXTRA_ARGS ${CBE_TEST_EXTRA_ARGS} "--vg")
    endif ()

    foreach(testdir ${CBE_TEST_DIRECTORIES})
      add_custom_target(cbe-test-${testdir}
        COMMAND ${LLVM_LIT}
                    --param cbe_site_config=${CMAKE_CURRENT_BINARY_DIR}/lit.site.cfg
                    --param build_config=${CMAKE_CFG_INTDIR}
                    -sv ${CBE_TEST_EXTRA_ARGS}
                    ${CMAKE_CURRENT_BINARY_DIR}/${testdir}
                    DEPENDS ${CBE_TEST_DEPS}
                    COMMENT "Running CBE regression tests in ${testdir}")
      set_target_properties(cbe-test-${testdir} PROPERTIES FOLDER "CBE")
    endforeach()

    add_custom_target(check-cbe
      COMMAND ${LLVM_LIT}
                  --param cbe_site_config=${CMAKE_CURRENT_BINARY_DIR}/lit.site.cfg
                  --param build_config=${CMAKE_CFG_INTDIR}
                  -sv ${CBE_TEST_EXTRA_ARGS}
                  ${CMAKE_CURRENT_BINARY_DIR}
                  DEPENDS ${CBE_TEST_DEPS}
                  COMMENT "Running CBE regression tests")
    set_target_properties(check-cbe PROPERTIES FOLDER "CBE")
  endif()

endif ( CBE_SOURCE_DIR STREQUAL "${LLVM_MAIN_SRC_DIR}/tools/cbe" )

configure_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/lit.site.cfg.in
  ${CMAKE_CURRENT_BINARY_DIR}/lit.site.cfg)

configure_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/update_check.py
  ${CMAKE_CURRENT_BINARY_DIR}/update_check.py)
file(COPY ${CMAKE_CURRENT_BINARY_DIR}/update_check.py
  DESTINATION ${LLVM_TOOLS_DIR}
  FILE_PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ
  GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)

# Add a legacy target spelling: cbe-test
add_custom_target(cbe-test)
set_target_properties(cbe-test PROPERTIES FOLDER "CBE")
add_dependencies(cbe-test check-cbe)
