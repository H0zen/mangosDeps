# BSD-3-Clause
# Copyright (c) 2022, Rochet2 <rochet2@post.com>
# All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are met:
#
# * Redistributions of source code must retain the above copyright notice, this
#   list of conditions and the following disclaimer.
#
# * Redistributions in binary form must reproduce the above copyright notice,
#   this list of conditions and the following disclaimer in the documentation
#   and/or other materials provided with the distribution.
#
# * Neither the name of the copyright holder nor the names of its
#   contributors may be used to endorse or promote products derived from
#   this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
# AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
# DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
# FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
# DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
# SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
# CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
# OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

project ( lua C )

include(GNUInstallDirs)

if(NOT WIN32)
  include(CheckLibraryExists)
  CHECK_LIBRARY_EXISTS(m sin "" HAVE_LIB_M)
  if (HAVE_LIB_M)
    set(EXTRA_LIBS ${EXTRA_LIBS} m)
  endif (HAVE_LIB_M)

  set(EXTRA_LIBS ${EXTRA_LIBS} ${READLINE_LIBRARY})
endif()

# LUA_VERSION must be one of lua51, lua52, lua53, lua54

include(FetchContent)
FetchContent_Declare(
 lua51
 URL   https://www.lua.org/ftp/lua-5.1.5.tar.gz
 URL_HASH SHA256=2640fc56a795f29d28ef15e13c34a47e223960b0240e8cb0a82d9b0738695333
)
FetchContent_Declare(
 lua52
 URL   https://www.lua.org/ftp/lua-5.2.4.tar.gz
 URL_HASH SHA256=b9e2e4aad6789b3b63a056d442f7b39f0ecfca3ae0f1fc0ae4e9614401b69f4b
)
FetchContent_Declare(
 lua53
 URL   https://www.lua.org/ftp/lua-5.3.6.tar.gz
 URL_HASH SHA256=fc5fd69bb8736323f026672b1b7235da613d7177e72558893a0bdcd320466d60
)
FetchContent_Declare(
 lua54
 URL   https://www.lua.org/ftp/lua-5.4.4.tar.gz
 URL_HASH SHA256=164c7849653b80ae67bec4b7473b884bf5cc8d2dca05653475ec2ed27b9ebf61
)
FetchContent_MakeAvailable(${LUA_VERSION})

# Easen warnings
string(REGEX REPLACE "( |^)/W[0-9]( |$)" "\\1/W2\\2" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
string(REGEX REPLACE "( |^)/W[0-9]( |$)" "\\1/W2\\2" CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")

set(LUA_SOURCE_FOLDER "${${LUA_VERSION}_SOURCE_DIR}/src")

file(GLOB LOCAL_SOURCES_H ${LUA_SOURCE_FOLDER}/*.h)
file(GLOB LOCAL_SOURCES_C ${LUA_SOURCE_FOLDER}/*.c)
list(REMOVE_ITEM LOCAL_SOURCES_C ${LUA_SOURCE_FOLDER}/lua.c)
list(REMOVE_ITEM LOCAL_SOURCES_C ${LUA_SOURCE_FOLDER}/luac.c)


if (LUA_STATIC)
  add_library(lualib STATIC ${LOCAL_SOURCES_H} ${LOCAL_SOURCES_C})
else()
  add_library(lualib STATIC ${LOCAL_SOURCES_H} ${LOCAL_SOURCES_C})
endif()

add_library(lualib_static STATIC ${LOCAL_SOURCES_H} ${LOCAL_SOURCES_C})
set_property(TARGET lualib PROPERTY POSITION_INDEPENDENT_CODE ON)
set_target_properties(lualib PROPERTIES LINKER_LANGUAGE C)
set_target_properties(lualib_static PROPERTIES LINKER_LANGUAGE C)

target_include_directories(lualib PUBLIC "${LUA_SOURCE_FOLDER}" "${CMAKE_CURRENT_SOURCE_DIR}")
target_include_directories(lualib_static PUBLIC "${LUA_SOURCE_FOLDER}" "${CMAKE_CURRENT_SOURCE_DIR}")

if (WIN32)
  set_target_properties(lualib PROPERTIES OUTPUT_NAME ${LUA_VERSION})
  target_compile_definitions(lualib PRIVATE _CRT_SECURE_NO_WARNINGS)
  target_compile_definitions(lualib_static PRIVATE _CRT_SECURE_NO_WARNINGS)
  if (NOT LUA_STATIC)
    target_compile_definitions(lualib PRIVATE LUA_BUILD_AS_DLL)
  endif()
else()
  target_include_directories(lualib PUBLIC "${READLINE_INCLUDE_DIR}")
  target_include_directories(lualib_static PUBLIC "${READLINE_INCLUDE_DIR}")
  set_target_properties(lualib PROPERTIES PUBLIC_HEADER "${LOCAL_SOURCES_H};${CMAKE_CURRENT_SOURCE_DIR}/lua.hpp")
  if (APPLE)
    target_compile_definitions(lualib PUBLIC LUA_USE_MACOSX)
    target_compile_definitions(lualib_static PUBLIC LUA_USE_MACOSX)
    target_compile_options(lualib PRIVATE -Wno-deprecated-declarations -Wno-empty-body)
    target_compile_options(lualib_static PRIVATE -Wno-deprecated-declarations -Wno-empty-body)
    target_link_libraries(lualib ${EXTRA_LIBS})
    target_link_libraries(lualib_static ${EXTRA_LIBS})
  elseif (UNIX)
    target_compile_definitions(lualib PUBLIC LUA_USE_LINUX)
    target_compile_definitions(lualib_static PUBLIC LUA_USE_LINUX)
    target_link_libraries(lualib ${CMAKE_DL_LIBS} ${EXTRA_LIBS})
    target_link_libraries(lualib_static ${CMAKE_DL_LIBS} ${EXTRA_LIBS})
    set_target_properties(lualib PROPERTIES OUTPUT_NAME ${LUA_VERSION})
    set_target_properties(lualib_static PROPERTIES OUTPUT_NAME ${LUA_VERSION})
  endif()
endif()

add_executable(lua_interpreter ${LUA_SOURCE_FOLDER}/lua.c)
target_link_libraries(lua_interpreter lualib_static)
target_compile_definitions(lua_interpreter PRIVATE _CRT_SECURE_NO_WARNINGS)
set_target_properties(lua_interpreter PROPERTIES OUTPUT_NAME ${LUA_VERSION}_interpreter)


add_executable(lua_compiler ${LUA_SOURCE_FOLDER}/luac.c)
target_link_libraries(lua_compiler lualib_static)
target_compile_definitions(lua_compiler PRIVATE _CRT_SECURE_NO_WARNINGS)
set_target_properties(lua_compiler PROPERTIES OUTPUT_NAME ${LUA_VERSION}_compiler)

install(TARGETS lualib lualib_static lua_interpreter lua_compiler
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

if (WIN32)
 if (NOT LUA_STATIC)
  install(FILES $<TARGET_PDB_FILE:lualib> DESTINATION "${CMAKE_INSTALL_BINDIR}" OPTIONAL)
 endif()
 install(FILES $<TARGET_PDB_FILE:lua_compiler>    DESTINATION "${CMAKE_INSTALL_PREFIX}" OPTIONAL)
 install(FILES $<TARGET_PDB_FILE:lua_interpreter> DESTINATION "${CMAKE_INSTALL_BINDIR}" OPTIONAL)
endif()
