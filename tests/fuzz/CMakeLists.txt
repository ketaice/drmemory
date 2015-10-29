# **********************************************************
# Copyright (c) 2012-2015 Google, Inc.  All rights reserved.
# **********************************************************

# Dr. Memory: the memory debugger
#
# This library is free software; you can redistribute it and/or
# modify it under the terms of the GNU Lesser General Public
# License as published by the Free Software Foundation;
# version 2.1 of the License, and no later version.
#
# This library is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
# Library General Public License for more details.
#
# You should have received a copy of the GNU Lesser General Public
# License along with this library; if not, write to the Free Software
# Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.

cmake_minimum_required(VERSION 2.6)

include(${PROJECT_SOURCE_DIR}/make/policies.cmake NO_POLICY_SCOPE)

# Always build Debug
set(CMAKE_BUILD_TYPE "Debug")

set_output_dirs("${PROJECT_BINARY_DIR}/tests")

# variables for platform-specific options and descriptor components
if (UNIX)
  set(app_extension "")
  if (X64)
    set(fuzz_buffer_cpp_symbol "_ZN13BufferPrinter8repeatmeEPjm")
  else (X64)
    set(fuzz_buffer_cpp_symbol "_ZN13BufferPrinter8repeatmeEPjj")
  endif (X64)
  set(enable_mangled_names "")
else (UNIX)
  set(app_extension ".exe")
  if (X64)
    set(fuzz_buffer_cpp_symbol "?repeatme-BufferPrinter--QEAAXPEAI_K-Z") # '-' for '@'
  else (X64)
    set(fuzz_buffer_cpp_symbol "?repeatme-BufferPrinter--QAEXPAII-Z") # '-' for '@'
  endif (X64)
  set(enable_mangled_names ";-fuzz_mangled_names")
endif (UNIX)

# fuzz the c target
set(fuzz_buffer_drmem_ops "-fuzz_target;<main>!repeatme|2|0|1|10")
newtest_ex(fuzz_buffer fuzz_buffer.c
  "initialize" "${fuzz_buffer_drmem_ops}" "" OFF "" 0)
set(mutator_drmem_ops "-fuzz_mutator;o|b|r|3")
newtest_nobuild_ex(fuzz_buffer.mutator.o-b-s-3 fuzz_buffer
  "initialize" "${fuzz_buffer_drmem_ops};${mutator_drmem_ops}" "" OFF "" 0 "")
set(mutator_drmem_ops "-fuzz_mutator;r|b|r|3")
newtest_nobuild_ex(fuzz_buffer.mutator.r-b-s-3 fuzz_buffer
  "initialize" "${fuzz_buffer_drmem_ops};${mutator_drmem_ops}" "" OFF "" 0 "")
set(mutator_drmem_ops "-fuzz_mutator;o|b||3")
newtest_nobuild_ex(fuzz_buffer.mutator.o-b-3 fuzz_buffer
  "initialize" "${fuzz_buffer_drmem_ops};${mutator_drmem_ops}" "" OFF "" 0 "")
set(mutator_drmem_ops "-fuzz_mutator;r|n||0")
newtest_nobuild_ex(fuzz_buffer.mutator.r-n fuzz_buffer
  "initialize" "${fuzz_buffer_drmem_ops};${mutator_drmem_ops}" "" OFF "" 0 "")
set(mutator_drmem_ops "-fuzz_mutator;r|n||0|0x123456789abcdef0")
newtest_nobuild_ex(fuzz_buffer.mutator.random_seed fuzz_buffer
  "initialize" "${fuzz_buffer_drmem_ops};${mutator_drmem_ops}" "" OFF "" 0 "")
set(fuzz_one_input_drmem_ops "-fuzz_one_input;0102030405060708090a0b0c0d0e0f10")
newtest_nobuild_ex(fuzz_buffer.one-input fuzz_buffer
  "initialize" "${fuzz_buffer_drmem_ops};${fuzz_one_input_drmem_ops}" "" OFF "" 0 "")
set(skip_initial_drmem_ops "-fuzz_skip_initial;2")
newtest_nobuild_ex(fuzz_buffer.skip_initial fuzz_buffer
  "initialize" "${fuzz_buffer_drmem_ops};${skip_initial_drmem_ops}" "" OFF "" 0 "")
if (NOT X64) # test detection of errors that requires shadow-memory
  newtest_nobuild_ex(fuzz_buffer.uninitialized fuzz_buffer
    "" "${fuzz_buffer_drmem_ops}" "" OFF "" 0 "")
  newtest_nobuild_ex(fuzz_buffer.overwrite fuzz_buffer
    "initialize;overwrite" "${fuzz_buffer_drmem_ops}" "" OFF "" 0 "")
  newtest_nobuild_ex(fuzz_buffer.overread fuzz_buffer
    "initialize;overread" "${fuzz_buffer_drmem_ops}" "" OFF "" 0 "")
  newtest_nobuild_ex(fuzz_buffer.underwrite fuzz_buffer
    "initialize;underwrite" "${fuzz_buffer_drmem_ops}" "" OFF "" 0 "")
  newtest_nobuild_ex(fuzz_buffer.underread fuzz_buffer
    "initialize;underread" "${fuzz_buffer_drmem_ops}" "" OFF "" 0 "")
  newtest_nobuild_ex(fuzz_buffer.leak fuzz_buffer
    "initialize;leak" "${fuzz_buffer_drmem_ops}" "" OFF "" 0 "")
endif (NOT X64)
set(fuzz_buffer_drmem_ops "-fuzz_target;<main>!repeatme|2|0|1|20")
set(fixed_size_drmem_ops "-fuzz_buffer_fixed_size;11")
newtest_nobuild_ex(fuzz_buffer.fixed_size fuzz_buffer
  "initialize" "${fuzz_buffer_drmem_ops};${fixed_size_drmem_ops}" "" OFF "" 0 "")
set(offset_drmem_ops "-fuzz_buffer_offset;9")
newtest_nobuild_ex(fuzz_buffer.offset fuzz_buffer
  "initialize" "${fuzz_buffer_drmem_ops};${offset_drmem_ops}" "" OFF "" 0 "")
set(fixed_with_offset_drmem_ops "-fuzz_buffer_fixed_size;5;-fuzz_buffer_offset;5")
newtest_nobuild_ex(fuzz_buffer.offset fuzz_buffer
  "initialize" "${fuzz_buffer_drmem_ops};${fixed_with_offset_drmem_ops}" "" OFF "" 0 "")
set(fuzz_buffer_drmem_ops "-fuzz_target;fuzz_buffer${app_extension}!repeatme|2|0|1|10")
newtest_nobuild_ex(fuzz_buffer.module_name fuzz_buffer
  "initialize" "${fuzz_buffer_drmem_ops}" "" OFF "" 0 "")

# fuzz the cpp target
if (NOT UNIX AND NOT X64)
  set(cpp_callconv "|6") # on Windows x86 the target uses thiscall
endif (NOT UNIX AND NOT X64)
set(fuzz_buffer_symbol "fuzz_buffer.cpp${app_extension}!${fuzz_buffer_cpp_symbol}")
set(fuzz_buffer_drmem_ops
  "-fuzz_target;${fuzz_buffer_symbol}|3|1|2|10${cpp_callconv}${enable_mangled_names}")
newtest_ex(fuzz_buffer.cpp fuzz_buffer.cpp "initialize"
  "${fuzz_buffer_drmem_ops}" "" OFF "" 0)
if (NOT X64) # test detection of errors that requires shadow-memory
  newtest_nobuild_ex(fuzz_buffer.uninitialized.cpp fuzz_buffer.cpp
    "" "${fuzz_buffer_drmem_ops}" "" OFF "" 0 "")
  newtest_nobuild_ex(fuzz_buffer.overwrite.cpp fuzz_buffer.cpp
    "initialize;overwrite" "${fuzz_buffer_drmem_ops}" "" OFF "" 0 "")
  newtest_nobuild_ex(fuzz_buffer.underwrite.cpp fuzz_buffer.cpp
    "initialize;underwrite" "${fuzz_buffer_drmem_ops}" "" OFF "" 0 "")
  newtest_nobuild_ex(fuzz_buffer.overread.cpp fuzz_buffer.cpp
    "initialize;overread" "${fuzz_buffer_drmem_ops}" "" OFF "" 0 "")
  newtest_nobuild_ex(fuzz_buffer.underread.cpp fuzz_buffer.cpp
    "initialize;underread" "${fuzz_buffer_drmem_ops}" "" OFF "" 0 "")
  newtest_nobuild_ex(fuzz_buffer.leak.cpp fuzz_buffer.cpp
    "initialize;leak" "${fuzz_buffer_drmem_ops}" "" OFF "" 0 "")
endif (NOT X64)
if (NOT UNIX)
  # demangled symbols can't be found on unix (elf table only has mangled)
  set(fuzz_buffer_symbol "fuzz_buffer.cpp${app_extension}!BufferPrinter::repeatme")
  set(fuzz_buffer_drmem_ops
    "-fuzz_target;${fuzz_buffer_symbol}|3|1|2|10${cpp_callconv}")
  newtest_nobuild_ex(fuzz_buffer.cpp.demangled fuzz_buffer.cpp
    "initialize" "${fuzz_buffer_drmem_ops}" "" OFF "" 0 "")

  # disable optimizations to prevent the target from being inlined
  append_property_string(TARGET fuzz_buffer COMPILE_FLAGS "/Ob0")
  append_property_string(TARGET fuzz_buffer.cpp COMPILE_FLAGS "/Ob0")
endif (NOT UNIX)
# XXX i#1734: need a test for a fuzz target loaded after target registration

if (NOT X64)
  set(fuzz_threads_drmem_ops "-fuzz_target;<main>!print_buffer|2|0|1|10")
  if (UNIX)
    newtest_ex(fuzz_threads fuzz_pthreads.c "" "${fuzz_threads_drmem_ops}" "" OFF "" 0)
    target_link_libraries(fuzz_threads pthread)
  else (UNIX)
    newtest_ex(fuzz_threads fuzz_winthreads.c "" "${fuzz_threads_drmem_ops}" "" OFF "" 0)
  endif (UNIX)
endif (NOT X64)