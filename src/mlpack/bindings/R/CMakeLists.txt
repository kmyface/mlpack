macro (not_found_return message)
  message(STATUS "${message}")
  macro (add_r_binding directory name)
    # Do nothing.
  endmacro ()

  macro (post_r_setup)
    # Do nothing.
  endmacro ()

  return()
endmacro ()

# If we are not supposed to make R bindings, define the macro so it does
# nothing and leave this file.
if (NOT BUILD_R_BINDINGS)
  not_found_return("Not building R bindings.")
endif ()

if (BUILD_R_BINDINGS)

  # If mlpack upgrade the version of dependencies, then we also have to update the version here.
  set(RcppArmadillo_Version "0.${ARMADILLO_VERSION}")
  set(RcppEnsmallen_Version "0.${ENSMALLEN_VERSION}")

  # Import find_r_module.
  include(${CMAKE_SOURCE_DIR}/CMake/FindRModule.cmake)
  find_package(R 4.0)
  if (NOT R_FOUND)
    set(R_NOT_FOUND_MSG "${R_NOT_FOUND_MSG}\n    - R")
  endif ()
  find_r_module(roxygen2)
  if (NOT R_ROXYGEN2)
    set(R_NOT_FOUND_MSG "${R_NOT_FOUND_MSG}\n    - roxygen2")
  endif ()
  find_r_module(Rcpp 0.12.12)
  if (NOT R_RCPP)
    set(R_NOT_FOUND_MSG "${R_NOT_FOUND_MSG}\n    - Rcpp")
  endif ()
  find_r_module(RcppArmadillo "${RcppArmadillo_Version}")
  if (NOT R_RCPPARMADILLO)
    set(R_NOT_FOUND_MSG "${R_NOT_FOUND_MSG}\n    - RcppArmadillo")
  endif ()
  find_r_module(RcppEnsmallen "${RcppEnsmallen_Version}")
  if (NOT R_RCPPENSMALLEN)
    set(R_NOT_FOUND_MSG "${R_NOT_FOUND_MSG}\n    - RcppEnsmallen")
  endif ()
  find_r_module(testthat)
  if (NOT R_TESTTHAT)
    set(R_NOT_FOUND_MSG "${R_NOT_FOUND_MSG}\n    - testthat")
  endif ()
  find_r_module(pkgbuild) # Required for roxygen2 with src/ directories.
  if (NOT R_PKGBUILD)
    set(R_NOT_FOUND_MSG "${R_NOT_FOUND_MSG}\n    - pkgbuild")
  endif ()

  ## We need to check here if R and other dependencies is even available, as
  ## it is require to build R-bindings.
  if (FORCE_BUILD_R_BINDINGS)
    if (NOT R_FOUND OR NOT R_RCPP OR NOT R_RCPPARMADILLO OR NOT R_RCPPENSMALLEN
        OR NOT R_ROXYGEN2 OR NOT R_TESTTHAT OR NOT R_PKGBUILD)
      unset(BUILD_R_BINDINGS CACHE)
      message(FATAL_ERROR "Could not Build R Bindings, Following modules are not available:${R_NOT_FOUND_MSG}")
    endif()
  else ()
    if (NOT R_FOUND OR NOT R_RCPP OR NOT R_RCPPARMADILLO OR NOT R_RCPPENSMALLEN
        OR NOT R_ROXYGEN2 OR NOT R_TESTTHAT OR NOT R_PKGBUILD)
      unset(BUILD_R_BINDINGS CACHE)
      not_found_return("Not building R bindings, Following modules are not available:${R_NOT_FOUND_MSG}")
    endif()
  endif ()

  add_custom_target(R ALL)

  # Now configure DESCRIPTION.
  file(READ "${CMAKE_SOURCE_DIR}/src/mlpack/core/util/version.hpp"
      VERSION_HPP_CONTENTS)
  string(REGEX REPLACE ".*#define MLPACK_VERSION_MAJOR ([0-9]+).*" "\\1"
      MLPACK_VERSION_MAJOR "${VERSION_HPP_CONTENTS}")
  string(REGEX REPLACE ".*#define MLPACK_VERSION_MINOR ([0-9]+).*" "\\1"
      MLPACK_VERSION_MINOR "${VERSION_HPP_CONTENTS}")
  string(REGEX REPLACE ".*#define MLPACK_VERSION_PATCH [\"]?([0-9x]+)[\"]?.*"
      "\\1" MLPACK_VERSION_PATCH "${VERSION_HPP_CONTENTS}")
  set(PACKAGE_VERSION
      "${MLPACK_VERSION_MAJOR}.${MLPACK_VERSION_MINOR}.${MLPACK_VERSION_PATCH}")
  if (USING_GIT)
    set(PACKAGE_DOC_VERSION "git")
  else ()
    set(PACKAGE_DOC_VERSION "${PACKAGE_VERSION}")
  endif ()

  string(TIMESTAMP PACKAGE_DATE "%Y-%m-%d")

  # We need to generate an Authors@R list using every single contributor in
  # COPYRIGHT.txt.  That takes a little bit of processing.
  file(READ "${CMAKE_SOURCE_DIR}/COPYRIGHT.txt" COPYRIGHT_TXT_CONTENTS)
  string(REGEX MATCHALL "  Copyright [0-9-]*, ([^\n]*)\n" CONTRIBUTORS_LIST
      "${COPYRIGHT_TXT_CONTENTS}")

  # These are the authors meant to be listed as 'authors' and not
  # 'contributors'.  If you contributed specifically to the R bindings, you
  # should probably be listed here, so if you're not, open a PR to fix it! :)
  set(SPECIAL_AUTHORS "Yashwant Singh Parihar" "Ryan Curtin" "Dirk Eddelbuettel"
      "James Balamuta")

  string(CONCAT AUTHORS_R "c(\n"
      "    person(\"Yashwant\", \"Singh Parihar\", "
              "email = \"yashwantsingh.sngh@gmail.com\", "
              "role = c(\"aut\", \"ctb\", \"cph\")),\n"
      "    person(\"Ryan\", \"Curtin\", email = \"ryan@ratml.org\", "
              "role = c(\"aut\", \"ctb\", \"cph\", \"cre\")),\n"
      "    person(\"Dirk\", \"Eddelbuettel\", email = \"edd@debian.org\", "
              "role = c(\"aut\", \"ctb\", \"cph\")),\n"
      "    person(\"James\", \"Balamuta\", "
              "email = \"james.balamuta@gmail.com\", "
              "role = c(\"aut\", \"ctb\", \"cph\")),")
  foreach (CONTRIBUTOR_LINE ${CONTRIBUTORS_LIST})
    # Strip 'Copyright XXXX-YYYY, '.
    string(REGEX REPLACE "^  Copyright [0-9-]*, (.*)\n$" "\\1"
        CONTRIBUTOR_FILTERED "${CONTRIBUTOR_LINE}")

    # Extract the email if it exists.
    string(REGEX MATCH "^[^<]*<(.*)>.*$" HAS_EMAIL "${CONTRIBUTOR_FILTERED}")

    # The first name is just the first space-delimited word.  (That may not
    # always be right, but we have no way to know what is a first name and last
    # name and therefore must assume.)
    string(REGEX REPLACE "^([^ ]*) .*$" "\\1" CONTRIBUTOR_FIRST_NAME
        "${CONTRIBUTOR_FILTERED}")

    # Extracting the last name is just the rest of the tokens, but the regex is
    # different depending on whether we managed to get an email.
    if (HAS_EMAIL)
      string(REGEX REPLACE "^[^<]*<(.*)>.*$" "\\1" CONTRIBUTOR_EMAIL
          "${CONTRIBUTOR_FILTERED}")
      string(REGEX MATCH "^[^ ]* (.*) <.*$" CONTRIBUTOR_LAST_NAME
          "${CONTRIBUTOR_FILTERED}")
      if (NOT CONTRIBUTOR_LAST_NAME)
        set (CONTRIBUTOR_LAST_NAME "")
      else ()
        string(REGEX REPLACE "^[^ ]* (.*) <.*$" "\\1" CONTRIBUTOR_LAST_NAME
            "${CONTRIBUTOR_FILTERED}")
      endif ()

      # Skip anyone already listed as an author.
      if ("${CONTRIBUTOR_FIRST_NAME} ${CONTRIBUTOR_LAST_NAME}" IN_LIST
          SPECIAL_AUTHORS)
        continue()
      endif ()

      string(CONCAT AUTHORS_R "${AUTHORS_R}\n    "
          "person(\"${CONTRIBUTOR_FIRST_NAME}\", \"${CONTRIBUTOR_LAST_NAME}\", "
          "email = \"${CONTRIBUTOR_EMAIL}\", role = c(\"ctb\", \"cph\")),")

    else ()
      # No email is available.  So just get the last name.
      string(REGEX MATCH "^[^ ]* (.*)$" CONTRIBUTOR_LAST_NAME
          "${CONTRIBUTOR_FILTERED}")
      if (NOT CONTRIBUTOR_LAST_NAME)
        set (CONTRIBUTOR_LAST_NAME "")
      else ()
        string(REGEX REPLACE "^[^ ]* (.*)$" "\\1" CONTRIBUTOR_LAST_NAME
            "${CONTRIBUTOR_FILTERED}")
      endif ()

      # Skip anyone already listed as an author.
      if ("${CONTRIBUTOR_FIRST_NAME} ${CONTRIBUTOR_LAST_NAME}" IN_LIST
          SPECIAL_AUTHORS)
        continue()
      endif ()

      string(CONCAT AUTHORS_R "${AUTHORS_R}\n    "
          "person(\"${CONTRIBUTOR_FIRST_NAME}\", \"${CONTRIBUTOR_LAST_NAME}\", "
          "role = c(\"ctb\", \"cph\")),")
    endif ()
  endforeach ()
  # We also have to remove the final comma...
  string(REGEX REPLACE ",$" "" AUTHORS_R_OUT "${AUTHORS_R}")
  set(AUTHORS_R "${AUTHORS_R_OUT})")

  configure_file(${CMAKE_SOURCE_DIR}/src/mlpack/bindings/R/mlpack/DESCRIPTION.in
                 ${CMAKE_CURRENT_BINARY_DIR}/mlpack/DESCRIPTION
                 @ONLY)

  # Create the empty NAMESPACE file that will include all export functions.
  file(WRITE
      "${CMAKE_CURRENT_BINARY_DIR}/mlpack/NAMESPACE"
      "# Generated by roxygen2: do not edit by hand"
      "\n\n")

  set(CPP_SOURCES
   "${CMAKE_CURRENT_SOURCE_DIR}/mlpack/src/r_util.cpp"
   "${CMAKE_CURRENT_SOURCE_DIR}/mlpack/src/rcpp_mlpack.h"
   "${CMAKE_CURRENT_SOURCE_DIR}/mlpack/src/Makevars"
   "${CMAKE_CURRENT_SOURCE_DIR}/mlpack/src/Makevars.win"
  )

  set(R_SOURCES
    "${CMAKE_CURRENT_SOURCE_DIR}/mlpack/R/matrix_utils.R"
    "${CMAKE_CURRENT_SOURCE_DIR}/mlpack/R/package.R"
  )

  set(BINDINGS_SOURCES
    "${CMAKE_CURRENT_SOURCE_DIR}/get_type.hpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/print_doc.hpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/print_doc_functions.hpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/print_doc_functions_impl.hpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/print_input_param.hpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/get_param.hpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/get_printable_param.hpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/get_r_type.hpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/print_input_processing.hpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/print_output_processing.hpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/print_serialize_util.hpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/mlpack_main.hpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/R_option.hpp"
  )

  set(TESTS_SOURCES
    "${CMAKE_CURRENT_SOURCE_DIR}/tests/test_r_binding_main.cpp"
  )

  set(R_TESTTHAT_SOURCES
    "${CMAKE_CURRENT_SOURCE_DIR}/mlpack/tests/testthat/test-R_binding.R"
  )

  set(R_TESTS_SOURCES
    "${CMAKE_CURRENT_SOURCE_DIR}/mlpack/tests/testthat.R"
  )

  # Configure the license file.
  string(TIMESTAMP LICENSE_YEAR "%Y")
  configure_file("${CMAKE_CURRENT_SOURCE_DIR}/mlpack/LICENSE.in"
                 "${CMAKE_CURRENT_BINARY_DIR}/mlpack/LICENSE")

  add_custom_target(r_copy ALL)

  # First we have to create all the required directories for copy.
  add_custom_command(TARGET r_copy PRE_BUILD
      COMMAND ${CMAKE_COMMAND} -E make_directory
          ${CMAKE_CURRENT_BINARY_DIR}/mlpack/R/
      COMMAND ${CMAKE_COMMAND} -E make_directory
          ${CMAKE_CURRENT_BINARY_DIR}/mlpack/tests/testthat
      COMMAND ${CMAKE_COMMAND} -E make_directory
          ${CMAKE_CURRENT_BINARY_DIR}/mlpack/src/mlpack/bindings/R/tests
   )

  if (BUILD_TESTS)
    add_custom_command(TARGET r_copy PRE_BUILD
        COMMAND ${CMAKE_COMMAND} ARGS -E copy_if_different
            ${TESTS_SOURCES}
            ${CMAKE_CURRENT_BINARY_DIR}/mlpack/src/mlpack/bindings/R/tests)
  endif()

  # Copy all necessary files for building package.
  foreach(cpp_file ${CPP_SOURCES})
    add_custom_command(TARGET r_copy PRE_BUILD
        COMMAND ${CMAKE_COMMAND} ARGS -E copy_if_different
            ${cpp_file}
            ${CMAKE_CURRENT_BINARY_DIR}/mlpack/src/)
  endforeach()
  foreach(r_file ${R_SOURCES})
    add_custom_command(TARGET r_copy PRE_BUILD
        COMMAND ${CMAKE_COMMAND} ARGS -E copy_if_different
            ${r_file}
            ${CMAKE_CURRENT_BINARY_DIR}/mlpack/R/)
  endforeach()
  foreach(bindings_file ${BINDINGS_SOURCES})
    add_custom_command(TARGET r_copy PRE_BUILD
        COMMAND ${CMAKE_COMMAND} ARGS -E copy_if_different
            ${bindings_file}
            ${CMAKE_CURRENT_BINARY_DIR}/mlpack/src/mlpack/bindings/R)
  endforeach()
  add_custom_command(TARGET r_copy PRE_BUILD
      COMMAND ${CMAKE_COMMAND} ARGS -E copy_if_different
          ${R_TESTTHAT_SOURCES}
          ${CMAKE_CURRENT_BINARY_DIR}/mlpack/tests/testthat)
  add_custom_command(TARGET r_copy PRE_BUILD
      COMMAND ${CMAKE_COMMAND} ARGS -E copy_if_different
          ${R_TESTS_SOURCES}
          ${CMAKE_CURRENT_BINARY_DIR}/mlpack/tests)
  # This file will take care of multiple definition of functions in .cpp files.
  add_custom_command(TARGET r_copy PRE_BUILD
      COMMAND ${CMAKE_COMMAND} ARGS -E touch
          "${CMAKE_CURRENT_BINARY_DIR}/mlpack/model.txt")

  file(COPY
       "${CMAKE_CURRENT_SOURCE_DIR}/mlpack/cleanup"
       DESTINATION
       "${CMAKE_CURRENT_BINARY_DIR}/mlpack/")

  file(COPY
       "${CMAKE_CURRENT_SOURCE_DIR}/mlpack/inst/CITATION"
       DESTINATION
       "${CMAKE_CURRENT_BINARY_DIR}/mlpack/inst")

  # Do the actual build.
  add_custom_target(r_build ALL)

  # "model.txt" is no longer useful, after generating src/.cpp files.
  # Remove this file.
  add_custom_command(TARGET r_build POST_BUILD
      COMMAND ${CMAKE_COMMAND} ARGS -E remove -f "model.txt"
      WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/mlpack"
  )

  # Build RcppExports.cpp/.R, NAMESPACE and man/ files.
  add_custom_command(TARGET r_build POST_BUILD
      COMMAND ${RSCRIPT_EXECUTABLE} ARGS "-e" "'Rcpp::compileAttributes()'"
      COMMAND ${RSCRIPT_EXECUTABLE} ARGS "-e" "'roxygen2::roxygenize(\".\")'"
      WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/mlpack"
  )

  # Build mlpack_${PACKAGE_VERSION}.tar.gz package.
  add_custom_command(TARGET r_build POST_BUILD
      COMMAND ${R_EXECUTABLE} CMD build mlpack
      WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
  )

  # Installation script for the packagae.
  install(CODE
      "execute_process(
         COMMAND ${R_EXECUTABLE} CMD INSTALL mlpack_${PACKAGE_VERSION}.tar.gz
         WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})"
  )

  add_dependencies(R r_build)
endif ()

# Define a global list of models, use for building serialization.R file.
define_property(GLOBAL PROPERTY R_MODELS
    BRIEF_DOCS "Global list of models"
    FULL_DOCS "Global list of models"
)

# Initialize list of models.
set_property(GLOBAL PROPERTY R_MODELS "")

macro (add_r_binding directory name)
if (BUILD_R_BINDINGS)

  # Append content to the list of models.
  set_property(GLOBAL APPEND PROPERTY R_MODELS
      ${CMAKE_CURRENT_SOURCE_DIR}/${directory}/${name}_main.cpp)

  # 1. Generate ${name}.cpp.
  add_custom_command(OUTPUT
      ${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/src/${name}.cpp
      COMMAND ${CMAKE_COMMAND}
          -DMODEL_FILE=${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/model.txt
          -DPROGRAM_NAME=${name}
          -DPROGRAM_MAIN_FILE=${CMAKE_CURRENT_SOURCE_DIR}/${directory}/${name}_main.cpp
          -DSOURCE_DIR=${CMAKE_SOURCE_DIR}
          -DR_CPP_IN=${CMAKE_SOURCE_DIR}/src/mlpack/bindings/R/r_method.cpp.in
          -DR_CPP_OUT=${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/src/${name}.cpp
          -P ${CMAKE_SOURCE_DIR}/CMake/R/ConfigureRCPP.cmake
      DEPENDS ${CMAKE_SOURCE_DIR}/src/mlpack/bindings/R/r_method.cpp.in
              ${CMAKE_SOURCE_DIR}/CMake/R/ConfigureRCPP.cmake)

  # 2. Generate ${name}.R.
  add_custom_command(OUTPUT ${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/build/generate_r_${name}.cpp
      COMMAND ${CMAKE_COMMAND}
          -DNAME=${name}
          -DGENERATE_CPP_IN=${CMAKE_SOURCE_DIR}/src/mlpack/bindings/R/generate_R.cpp.in
          -DGENERATE_CPP_OUT=${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/build/generate_r_${name}.cpp
          -DPROGRAM_MAIN_FILE=${CMAKE_CURRENT_SOURCE_DIR}/${directory}/${name}_main.cpp
          -P ${CMAKE_SOURCE_DIR}/CMake/ConfigureFile.cmake
      DEPENDS ${CMAKE_SOURCE_DIR}/src/mlpack/bindings/R/generate_R.cpp.in
              ${CMAKE_SOURCE_DIR}/CMake/ConfigureFile.cmake
              ${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/src/${name}.cpp)

  add_executable(generate_r_${name}
      ${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/build/generate_r_${name}.cpp
      ${CMAKE_SOURCE_DIR}/src/mlpack/bindings/R/print_R.hpp
      ${CMAKE_SOURCE_DIR}/src/mlpack/bindings/R/print_R.cpp
      ${CMAKE_SOURCE_DIR}/src/mlpack/bindings/R/get_type.hpp
      ${CMAKE_SOURCE_DIR}/src/mlpack/bindings/R/R_option.hpp
      ${CMAKE_SOURCE_DIR}/src/mlpack/bindings/R/print_input_param.hpp
      ${CMAKE_SOURCE_DIR}/src/mlpack/bindings/R/print_input_processing.hpp
      ${CMAKE_SOURCE_DIR}/src/mlpack/bindings/R/print_serialize_util.hpp
      ${CMAKE_SOURCE_DIR}/src/mlpack/bindings/R/print_output_processing.hpp)
  if (BUILD_SHARED_LIBS)
    target_link_libraries(generate_r_${name} ${MLPACK_LIBRARIES})
  else ()
    target_link_libraries(generate_r_${name} -static ${MLPACK_LIBRARIES})
  endif ()
  set_target_properties(generate_r_${name} PROPERTIES
      COMPILE_FLAGS "-DBINDING_TYPE=BINDING_TYPE_R"
      RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/build/bin/")
  add_custom_command(TARGET generate_r_${name} POST_BUILD
      COMMAND ${CMAKE_COMMAND}
          -DGENERATE_BINDING_PROGRAM=${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/build/bin/generate_r_${name}
          -DBINDING_OUTPUT_FILE=${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/R/${name}.R
          -P ${CMAKE_SOURCE_DIR}/CMake/GenerateBinding.cmake)
  add_dependencies(generate_r_${name} r_copy)
  add_dependencies(r_build generate_r_${name})
endif()
endmacro()

macro (post_r_setup)
  # In Case of R_Bindings Move all of these header and source files to
  # <package>/src/mlpack/.
  # Collect all header and source files in the library.
  file(GLOB_RECURSE R_SRC_HPP_FILES RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}" *.hpp)
  file(GLOB_RECURSE R_SRC_CPP_FILES RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}" *.cpp)
  file(GLOB_RECURSE R_SRC_H_FILES RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}" *.h)
  set(R_SRC_FILES ${R_SRC_HPP_FILES} ${R_SRC_CPP_FILES} ${R_SRC_H_FILES})

  # In case of R_Bindings we are copying all of these header to
  # <package>/src/cereal/ because the currently-packaged version
  # of Rcereal throws warnings on CRAN.
  # Collect all header files in the cereal folder.
  file(GLOB_RECURSE RCEREAL_SRC_H_FILES RELATIVE "${CEREAL_INCLUDE_DIR}"
       "${CEREAL_INCLUDE_DIR}/cereal/*.h")
  file(GLOB_RECURSE RCEREAL_SRC_HPP_FILES RELATIVE "${CEREAL_INCLUDE_DIR}"
       "${CEREAL_INCLUDE_DIR}/cereal/*.hpp")
  set(RCEREAL_SRC_FILES ${RCEREAL_SRC_HPP_FILES} ${RCEREAL_SRC_H_FILES})

  # First we have to create that directory though.
  execute_process(
    COMMAND ${CMAKE_COMMAND} -E
      make_directory ${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/inst/include/mlpack
  )

  # Then copy each of the header and source files over to that directory.
  set(MLPACK_SOURCES
    "${CMAKE_CURRENT_SOURCE_DIR}/base.hpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/prereqs.hpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/core.hpp"
  )

  foreach(mlpack_sources ${MLPACK_SOURCES})
    execute_process(
      COMMAND ${CMAKE_COMMAND} -E copy_if_different
        ${mlpack_sources}
        ${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/inst/include/mlpack
    )
  endforeach()

  # Copy configured config.hpp.
  execute_process(
    COMMAND ${CMAKE_COMMAND} -E copy
      "${CMAKE_BINARY_DIR}/include/mlpack/config-local.hpp"
      "${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/inst/include/mlpack/config.hpp")

  foreach(r_src_file ${R_SRC_FILES})
    if ("${r_src_file}" MATCHES "methods/" OR
        "${r_src_file}" MATCHES "core/" OR
        "${r_src_file}" MATCHES "bindings/util")
      execute_process(
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
          ${CMAKE_CURRENT_SOURCE_DIR}/${r_src_file}
          ${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/inst/include/mlpack/${r_src_file}
      )

      # Collect path of all source files and append to ${R_SRC}.
      if ("${r_src_file}" MATCHES ".cpp" AND
          NOT "${r_src_file}" MATCHES "main.cpp")
        string(APPEND R_SRC "#include <mlpack/${r_src_file}>\n")
      endif()

    endif()
  endforeach()

  # Copy cereal headers for R-bindings.
  foreach(rcereal_src_file ${RCEREAL_SRC_FILES})
    execute_process(
      COMMAND ${CMAKE_COMMAND} -E copy_if_different
        ${CEREAL_INCLUDE_DIR}/${rcereal_src_file}
        ${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/inst/include/${rcereal_src_file}
    )
  endforeach()

  # Resolve gcc warnings.
  file(READ ${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/inst/include/cereal/external/base64.hpp CEREAL_BASE64)
  string(REGEX REPLACE
      "\n#ifdef __GNUC__\n#pragma GCC diagnostic push\n#pragma GCC diagnostic ignored \"-Wconversion\"\n#endif\n"
      "" CEREAL_BASE64 "${CEREAL_BASE64}")
  string(REGEX REPLACE
      "#ifdef __GNUC__\n#pragma GCC diagnostic pop\n#endif"
      "" CEREAL_BASE64 "${CEREAL_BASE64}")
  file(WRITE ${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/inst/include/cereal/external/base64.hpp "${CEREAL_BASE64}")

  # NOTE: for CRAN, `sprintf` cannot be used; but it is used in
  # `stb_image_write.h`, so we replace it with `snprintf` below.
  file(READ "${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/inst/include/mlpack/core/stb/bundled/stb_image_write.h" STB_IMAGE_WRITE_H)
  string(REGEX REPLACE "sprintf\\(buffer," "snprintf(buffer, 128,"
      STB_IMAGE_WRITE_H_MOD "${STB_IMAGE_WRITE_H}")
  file(WRITE "${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/inst/include/mlpack/core/stb/bundled/stb_image_write.h" "${STB_IMAGE_WRITE_H_MOD}")

  # Then configure 'mlpack.h.in' using ${R_SRC}.
  configure_file(
      ${CMAKE_SOURCE_DIR}/src/mlpack/bindings/R/mlpack/inst/include/mlpack.h.in
      ${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/inst/include/mlpack.h
      @ONLY)

  file(WRITE
       "${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/R/serialization.R"
      "#' Serialize/Unserialize an mlpack model.\n"
      "#'\n"
      "#' @param model Input model pointer.\n"
      "#' @param filename Input filename.\n"
      "#' @export\n"
      "#' @rdname mlpack-serialization\n"
      "Serialize <- function(model, filename) {\n"
      "  model_serialization_function <-\n"
      "    switch(attributes(model)$type,\n")

  include("${CMAKE_SOURCE_DIR}/CMake/R/AppendSerialization.cmake")
  # Read list content.
  get_property(MODELS GLOBAL PROPERTY R_MODELS)
  foreach (models IN LISTS MODELS)
    append_serialization(
        "${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/R/serialization.R"
         ${models}
         TRUE)
  endforeach()

  file(APPEND
       "${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/R/serialization.R"
       "      stop(\"Requested model type is not currently supported.\")\n    )\n\n"
       "  # Read in model\n"
       "  con <- file(as.character(filename), \"wb\")\n"
       "  serialize(model_serialization_function(model), con)\n"
       "  close(con)\n}\n\n\n"
       "#' @return For Unserialize, Output model_ptr.\n"
       "#' @export\n"
       "#' @rdname mlpack-serialization\n"
       "Unserialize <- function(filename) {\n"
       "  con <- file(as.character(filename), \"rb\")\n"
       "  model <- unserialize(con)\n\n"
       "  model_unserialization_function <-\n"
       "    switch(attributes(model)$type,\n")
  foreach (models IN LISTS MODELS)
    append_serialization(
        "${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/R/serialization.R"
         ${models}
         FALSE)
  endforeach()

  file(APPEND
       "${CMAKE_BINARY_DIR}/src/mlpack/bindings/R/mlpack/R/serialization.R"
       "      stop(\"Requested model type is not currently supported.\")\n    )\n\n"
       "  model_ptr <- model_unserialization_function(model)\n"
       "  close(con)\n"
       "  return(model_ptr)\n}\n")
endmacro()

if (BUILD_TESTS AND BUILD_R_BINDINGS)
  add_subdirectory(tests)
endif ()
