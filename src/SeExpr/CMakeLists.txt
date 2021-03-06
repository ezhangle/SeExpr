# Copyright Disney Enterprises, Inc.  All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License
# and the following modification to it: Section 6 Trademarks.
# deleted and replaced with:
#
# 6. Trademarks. This License does not grant permission to use the
# trade names, trademarks, service marks, or product names of the
# Licensor and its affiliates, except as required for reproducing
# the content of the NOTICE file.
#
# You may obtain a copy of the License at
# http://www.apache.org/licenses/LICENSE-2.0

# Source files for llvm supported library and interpreter library
file(GLOB io_cpp "*.cpp")
file(GLOB to_remove "ExprLLVMCodeGeneration.cpp")
list(REMOVE_ITEM io_cpp ${to_remove})

set_source_files_properties("ExprBuiltins.cpp" PROPERTIES COMPILE_DEFINITIONS "__STDC_LIMIT_MACROS")

# Uncomment below to print debug messages / performance stats
#add_definitions(-DSEEXPR_DEBUG)
#add_definitions(-DSEEXPR_PERFORMANCE)

## find our parser generators
find_program(BISON_EXE bison)
find_program(FLEX_EXE flex)
find_program(SED_EXE sed)

# TODO use recent cmake to use FindFlex FindBison
if((BISON_EXE STREQUAL "BISON_EXE-NOTFOUND") OR (FLEX_EXE STREQUAL "FLEX_EXE-NOTFOUND")  OR (SED_EXE STREQUAL "SED_EXE-NOTFOUND"))
    # don't have flex/bison/sed, use pregenerated versions
    set (parser_cpp generated/ExprParser.cpp generated/ExprParserLex.cpp )
else ((BISON_EXE STREQUAL "BISON_EXE-NOTFOUND") OR (FLEX_EXE STREQUAL "FLEX_EXE-NOTFOUND")  OR (SED_EXE STREQUAL "SED_EXE-NOTFOUND"))
     ## build the parser from the flex/yacc sources

    ADD_CUSTOM_COMMAND(
      SOURCE "ExprParserLex.l"
      COMMAND "flex"
      ARGS "-oExprParserLexIn.cpp" "${CMAKE_CURRENT_SOURCE_DIR}/ExprParserLex.l"
      OUTPUT ExprParserLexIn.cpp
      DEPENDS ExprParserLex.l
    )
    
    ADD_CUSTOM_COMMAND(
      SOURCE "ExprParserLexIn.cpp"
      COMMAND "sed"
#      ARGS -e "'s/SeExprwrap(n)/SeExprwrap()/g'" -e "'s/yy/SeExpr2/g'" -e "'s/YY/SeExprYY/g'"  ExprParserLexIn.cpp | tee ExprParserLex.cpp ${CMAKE_CURRENT_SOURCE_DIR}/generated/ExprParserLex.cpp > /dev/null
      ARGS -e "'s/SeExprwrap(n)/SeExprwrap()/g'" -e "'s/yy/SeExpr2/g'" -e "'s/YY/SeExprYY/g'"  ExprParserLexIn.cpp | tee ExprParserLex.cpp ${CMAKE_CURRENT_SOURCE_DIR}/generated/ExprParserLex.cpp > /dev/null
      OUTPUT ExprParserLex.cpp
      DEPENDS ExprParserLexIn.cpp
    )
    
    ADD_CUSTOM_COMMAND(
      SOURCE "ExprParser.y"
      COMMAND "bison"
      ARGS "--defines" "--verbose" "--fixed-output-files" "-p" "SeExpr2" "${CMAKE_CURRENT_SOURCE_DIR}/ExprParser.y"
      OUTPUT y.tab.c y.tab.h
      DEPENDS ExprParser.y
    )
    
    ADD_CUSTOM_COMMAND(
      SOURCE "y.tab.h"
      COMMAND "sed"
      ARGS -e "'s/yy/SeExpr2/g'" -e "'s/YY/SeExprYY/g'" y.tab.h | tee  ExprParser.tab.h ${CMAKE_CURRENT_SOURCE_DIR}/generated/ExprParser.tab.h > /dev/null
      OUTPUT ExprParser.tab.h
      DEPENDS y.tab.h
    )
    
    ADD_CUSTOM_COMMAND(
      SOURCE "y.tab.c"
      COMMAND "sed"
      ARGS -e "'s/yy/SeExpr2/g'" -e "'s/YY/SeExprYY/g'" y.tab.c | tee ExprParser.cpp  "${CMAKE_CURRENT_SOURCE_DIR}/generated/ExprParser.cpp" > /dev/null
      OUTPUT ExprParser.cpp
      DEPENDS y.tab.c ExprParser.tab.h
    )

    ## set build files
    set (parser_cpp ExprParser.cpp ExprParserLex.cpp )

endif( (BISON_EXE STREQUAL "BISON_EXE-NOTFOUND") OR (FLEX_EXE STREQUAL "FLEX_EXE-NOTFOUND")  OR (SED_EXE STREQUAL "SED_EXE-NOTFOUND"))


## Make the SeExpr library with and without LLVM support
file(GLOB llvm_cpp "*.cpp")
ADD_LIBRARY (SeExpr2 SHARED ${io_cpp} ${core_cpp} ${parser_cpp} ${llvm_cpp})
IF(NOT WIN32)
    TARGET_LINK_LIBRARIES(SeExpr2 "dl" "pthread")
ENDIF(NOT WIN32)

## Install binary and includes
INSTALL (TARGETS SeExpr2 DESTINATION ${CMAKE_INSTALL_LIBDIR})
INSTALL (DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/ DESTINATION include/SeExpr2
        FILES_MATCHING PATTERN "*.h")

# Setup header files
FILE(GLOB public_includes "*.h")
FILE(COPY ${public_includes} DESTINATION ${CMAKE_BINARY_DIR}/include/SeExpr2)

configure_file(ExprConfig.h.in ExprConfig.h)
FILE(COPY ${CMAKE_CURRENT_BINARY_DIR}/ExprConfig.h DESTINATION ${CMAKE_BINARY_DIR}/include/SeExpr2)
INSTALL (FILES ${CMAKE_CURRENT_BINARY_DIR}/ExprConfig.h DESTINATION include/SeExpr2)

if (ENABLE_LLVM_BACKEND)
       TARGET_LINK_LIBRARIES(SeExpr2 ${LLVM_LIB} "dl" "pthread")
endif()
