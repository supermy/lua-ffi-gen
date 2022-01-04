Welcome to the lua-ffi-gen wiki!

This project contains an imlementation of a Clang plugin for generating C declarations that are needed by LuaJIT FFI library in order to use C declarations inside Lua code.

Basic usage information

Mark the declarations of functions, structures/unions or enums in C code that you want to use inside Lua code with __attribute__((ffibinding)). Build the C code with Clang and run the implemented plugin. The plugin will generate bindings for marked declarations in an output Lua file. It will find all declarations of types that are needed for using marked declarations in Lua code.

Building Clang/LLVM

In order to use the plugin, you have to acquire Clang/LLVM source code, apply the provided clang-patch.diff and build the compiler. Follow the steps:

Download Clang and LLVM source code, version 3.6

Extract lua-ffi-gen to Clang examples:

cd llvm/tools/clang/examples/
tar xf lua-ffi-gen.tgz

Rename the created folder to ffi-gen.

Apply the patch to clang:

cd ..
patch -p1 < examples/ffi-gen/clang-patch.diff

Create separate build folder and build Clang and LLVM with the examples.

cd ../../../
mkdir build
cd build
../llvm/configure --enable-optimized --disable-debug
make BUILD_EXAMPLES=1

After the build is finished, you can find the ffi-gen.so on the following path:

build/Release+Asserts/lib/ffi-gen.so

Plugin name

Name of the plugin and its directory is set to ffi-gen in the provided Clang patch. If you rename the plugin directory, you should make appropriate changes to the provided Clang patch (substitute "ffi-gen" with new directory name in Makefile and CMakeLists.txt).
If you wish to change the name of the plugin, you have to change the library name in the Makefile, so it will be found on the following path:
build/Release+Asserts/lib/my_new_plugin_name.so
The provided Clang patch contains custom options added to Clang which ease the usage of the implemented plugin (shorten the number of arguments passed on the command line). If you change the plugin name, and want to use these custom options, you should make appropriate changes to the lib/Driver/Tools.cpp file (substitute "/../lib/ffi-gen.so" with the appropriate path to the plugin, and replace all occurrences of "-plugin-arg-ffi-gen" with "-plugin-arg-my_new_plugin_name").

Using the plugin

Once you have built Clang/LLVM with the plugin, you can start using it! For example, if you wish to get bindings for function foo in the following C code:

test.h

struct S {
int x;
};

typedef struct {
struct S s;
} TS;

test.c

#include "test.h"
void foo(TS ts);

You should mark the function with the ffibinding attribute:

__attribute__((ffibinding))
void foo(TS ts);

Next, you should build the C source code with Clang, and add an option to your command line to run the plugin. For example:

clang -c test.c -ffi-gen-enable

This will generate the following Lua file:

main_gen_ffi.lua

ffi = require("ffi")
ffi.cdef[[

struct S {
int x;
};

typedef struct {
struct S s;
} TS;

void foo(TS ts);

]]

There are several command line options that have been added to Clang to ease the usage of the plugin by shortening the number of parameters passed to the command line. These are:

-ffi-gen-enable - this parameter loads the plugin (it's mandatory to pass this argument to Clang if you want to use the implemented plugin, other parameters are optional)

ffi-gen-output output_file_name - use this parameter to specify the name of the output Lua file. The default is source_file_name_gen_ffi.lua.

ffi-gen-header header_file_name - use this parameter to specify the name of the header file. A header file is a simple .txt file that will be copied to the top of the generated output Lua file. If it contains a token "<source-files>", that token will be replaced with a path to the source file that the generated Lua file is based on.

ffi-gen-destdir path_to_dest_dir - use this parameter to specify the destination directory for the output Lua file.

ffi-gen-blacklist blacklist_file_name - use this parameter to specify the blacklist file name. The blacklist file is a .txt file that contains a list of declarations that must not end up in the generated Lua file.

ffi-gen-test - use this parameter to run the plugin in test mode. That means that bindings will be generated for each function found in the source code.

ffibinding attribute

You can mark functions, structures, unions and enumerations with the ffibinding attribute.

Blacklist

Declarations that should not end up in the generated Lua file should be stated in the following way, each in a new line in the file:

MyStruct - typedef name
struct S - structure/union
enum MyEnum - enumeration

ffi-combine

ffi-combine.lua is a Lua script that combines multiple generated Lua files. It takes declarations stated in the ffi.cdef block from input Lua files and stores them in the output Lua file, while avoiding duplicate declarations (it copies a declaration only once, if it comes across it again, it will ignore the declaration).
Example usage:

lua ffi-combine.lua file1.lua file2.lua --output=output.lua

In the line above, file1.lua and file2.lua are the input Lua files, and the output file is specified with the --output option. Supported options by the ffi-combine.lua script:

--output filename - specifies the output file name. Default is out.lua.

--header filename - specifies the header file name. Contents of the header file are copied to the top of the output Lua file. If the header file contains a token "", and one or more input Lua files also contain a header with lines starting with ">> ", those lines will be copied to the output Lua file, starting from the line where the mentioned token is found.