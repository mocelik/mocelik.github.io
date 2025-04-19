---
categories: [cpp]
date: '2024-11-19T00:51:42-05:00'
layout: post
permalink: /c/debugging-loading-issues/
tags: [build, c, cpp, debugging]
title: 'From undefined references to ldd: Debugging Linking in C/C++'
---

One of the most misunderstood stages of building or deploying an application are the linking and loading stages. The error messages can be cryptic if you don’t understand exactly what they mean. Further causing confusion is that the C and C++ standards rely on linking, and indirectly refer to it, but they do not explicitly refer to static or dynamic linkers or loaders. Some of this post references information that is explained fairly well in [this tutorial from IBM](https://developer.ibm.com/tutorials/l-dynamic-libraries/). In this article, I assume the reader has some familiarity with how to compile and execute a program, and knows what a shared library is. I also assume they know what environment variables are.

Before diving deeper into the types of linker or loader issues you may come across, lets define the basic terms we’re using.

**Source files** are the code you write, typically with a .c, .cc, or .cpp extension. These files can be parsed as C, C++, or preprocessor statements. Once the preprocessing stage has completed, the source file is referred to as a translation unit.

A **translation unit** is compiled by the compiler (like gcc or g++) into an **object file**, which typically has a .o extension. Sometimes, for very simple programs, the object file can be omitted, and the translation unit can be converted directly to an executable file. In addition to object files, the translation unit can be compiled and archived (packaged) into a **static library** (typically **.a** extension, for archive) or a **shared library** (typically **.so** extension, for shared object).

**Linking** refers to the stage where multiple compiled object files are combined to create a single executable file. If object files or static libraries (archives) are linked, then the linking happens immediately, and the final executable file contains the code from those object files or static libraries within it. This is referred to as **static linking**.

Alternatively, linking can be done dynamically: the final executable contains a reference to a shared library that it needs, but it does not contain the shared library itself. In that case, when the executable is executed, the shared library will also be loaded into memory (if not already there) and combined with the executable as it starts up. This is referred to as **dynamic linking**.

There is a third type of linking called **dynamic loading**, which is where the executable file does not explicitly state the library it depends on. Instead, the developer will manually write code to find, open, parse, and load a library that contains the functions that it wants.

Yes, the terminology is nuanced.

Each of these stages can fail for different reasons. The following sections dive into the various types of failures that can occur, and how to resolve them. The sections refer to the following two files as main.c and foo.c respectively. While these files are in C, the same exact concepts apply directly to C++ as well (except for name mangling, which will be mentioned later).

```c
// main.c
int foo(); // function declaration

int main() {
    return foo();
}
```

```c
// foo.c
int foo() { // function definition
    return 0;
}
```

# Static Linking (Object Files)

As mentioned, static linking is when an object file (.o) or archive file (.a) is combined to create an executable file. While most build systems may execute the compiler (e.g. gcc or clang) to link objects, the real linker is usually an application called `<a href="https://man7.org/linux/man-pages/man1/ld.1.html">ld</a>`, or perhaps [gold](https://manpages.ubuntu.com/manpages/trusty/man1/x86_64-linux-gnu-ld.gold.1.html), or [lld](https://lld.llvm.org/), which is indirectly invoked by the compiler.

If the above main.c is compiled into an object file, `gcc -c main.c`, then it successfully produces an object file, `main.o`. However, if there is an attempt to compile and link it as an executable, `gcc main.c`, then we get an error:

```console
$ gcc -c main.c  # Succeeds
$ gcc main.c     # Error
/usr/bin/ld: /tmp/ccFEwSvr.o: in function `main':
main.c:(.text+0xe): undefined reference to `foo'
collect2: error: ld returned 1 exit status
```

The main thing to notice here is the phrase **`ld returned 1 exit status`**, indicating that the linker (`ld`) got an error. The error is that the `main` function referenced `foo`, but `foo` was not defined anywhere. Strictly speaking, this is not a compiler error, it is a linker error.

When linking object files, if you see a linker error like the above then you either are missing the function/symbol definition anywhere, or the file with the definition has not been passed to the linker.

In this case, the function is defined in `foo.c.` In that case, this file also needs to be compiled (`gcc -c foo.c`), and both object files need to be mentioned to the compiler (`gcc main.o foo.o`) in order to successfully link all of the symbols.

# Static Linking (Archives)

Linking static libraries, also known as archives, is almost the exact same as linking objects, but there are some important differences.

Firstly, to generate a static library from `foo.c`, the code has to be compiled into a static library. First, it is compiled into an object file: `gcc -c foo.c`. Then, the object file is archived: `ar -r libfoo.a foo.o`.

The library can now be used to link with `main.o`, with the important rule that the library with the symbol must be placed on the command line **after** the object files (or other libraries) that attempt to use that symbol. Therefore, the following `gcc libfoo.a main.o` does not work (it gives the same undefined reference error seen before), but `gcc main.o libfoo.a` works as intended. This is one of the important differences between linking static libraries and object files.

While it is possible to link directly to the archive, libraries are usually linked via a `-l` flag with the name of the library, excluding the `lib` prefix and `.a` suffix, as such: `gcc main.o -lfoo`. However, if you try running this command, you will see that it fails with a linker error:

```console
$ gcc main.o -lfoo
/usr/bin/ld: cannot find -lfoo: No such file or directory
collect2: error: ld returned 1 exit status
```

If the library (`libfoo.a`) is not in one of the default locations, like /lib, /usr/lib, or others depending on your system configuration, then the linker has to be explicitly told where to look.

You can tell the linker where to look by either setting the `LIBRARY_PATH` environment variable (not recommended), or by using a `-L` flag with the directory: `gcc main.o -lfoo -L .` (notice the `.` indicating the current directory). Now, the program compiles, links, and runs as expected.

Quick note: another environment variable, `LD_LIBRARY_PATH`, is more well-known, and will be mentioned later. Although it is similar to `LIBRARY_PATH`, it is used in different situations. Do not confuse the two.

Multiple libraries can be linked at the same time, and each library can be listed multiple times. This is required to handle circular dependencies between libraries – though there is also a less error-prone way by using `--start-group` and `--end-group` to handle a group of libraries with circular dependencies.

# Dynamic Linking (Shared Objects)

As previously mentioned, shared objects (\*.so files) are not built into the executable; rather, the executable contains a reference to the shared objects that it will need to link to when it runs. Even so, there are some verification steps that occur when the executable is first created – otherwise, the linker may have had to assume that a function that you wrote would have to have a definition in a library provided to you by a third party.

To create a shared library, you must compile the source file with the `-fpic` (or similar) flag: I won’t get into the details of fpic, fPIC, or fPIE here. As an aside, if you wish to link a static library into a shared library, you must also compile the static library with `-fpic`. To create a shared library from foo.c, run `gcc -c -fpic foo.c` and `gcc -shared foo.o -o libfoo.so`.

When linking, as before, you must inform the linker of where to find the shared library. Once again, this can be done by setting either the `LIBRARY_PATH` environment variable, or, preferably, by using the `-L` flag: `gcc main.o -lfoo -L .`, otherwise we will run into the same issue:

```console
$ gcc -c -fpic foo.c
$ gcc -shared foo.o -o libfoo.so
$ gcc main.o -lfoo
/usr/bin/ld: cannot find -lfoo: No such file or directory
collect2: error: ld returned 1 exit status
$ gcc main.o -lfoo -L . # Succeeds
```

Even you resolve this error by providing the path to the library via `-L`, the executable will still report an error at runtime. You will likely see an error like this:

```console
$ ./a.out
./a.out: error while loading shared libraries: libfoo.so: cannot open shared object file: No such file or directory
```

The linker that was able to connect the dots and create the reference to the library was the static linker, `ld` (see `<a href="https://man7.org/linux/man-pages/man1/ld.1.html">man ld</a>`), whereas the linker that executes the executable and loads requisite libraries at runtime is the dynamic linker, `ld.so` (see `<a href="https://man7.org/linux/man-pages/man8/ld.so.8.html">man ld.so</a>`). Yes, the names are confusing. To confirm that your executable is loaded via `ld.so`, you may run `file ./a.out` and look for the `interpreter` field:

```console
$ file ./a.out
./a.out: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, <strong>interpreter /lib64/ld-linux-x86-64.so.2</strong>, BuildID[sha1]=6a54363596cf0f19bda44d43ed6adc4e9daf7603, for GNU/Linux 3.2.0, not stripped
```

In this case, on my machine, the interpreter is not named exactly `ld.so`, but rather `ld-linux-x86-64.so.2`. Close enough. This interpreter, the dynamic linker (also called a loader), is responsible for loading the shared objects, resolving the symbols, calling constructors for static objects, and calling your programs `int main()` function.

## Finding Shared Libraries

Once again, the linker error from before:

```console
$ ./a.out
./a.out: error while loading shared libraries: libfoo.so: cannot open shared object file: No such file or directory
```

This error occurs because `ld.so` was not able to find the shared object. `ld.so` goes (roughly) through the following summarized steps to try to find shared objects:

1. *\[\[deprecated\]\]* Check the applications hardcoded `DT_RPATH` if set
2. Check directories from the `LD_LIBRARY_PATH` environment variable
3. Check applications hardcoded `DT_RUNPATH`, if set (via passing `-rpath` to the linker)
4. Check directories listed in the `/etc/ld.so.cache` cache file 
    - The cache can be updated via running `ldconfig` (see [man ldconfig](https://man7.org/linux/man-pages/man8/ldconfig.8.html)), optionally with the first argument or `LD_LIBRARY_PATH` environment variable set to an additional directory to cache
    - The cache contains lists of libraries found in directories specified by `/etc/ld.so.conf` and `/etc/ld.so.conf.d/*.conf`
5. Check */lib*, or */lib64*
6. Check */usr/lib*, or */usr/lib64*

This is not an exhaustive or completely accurate list; the exact search algorithm is on the man page for `ld.so`.

To inform your linker of how to load your custom shared libraries, you have a few choices. Some build systems, like CMake, set the `RUNPATH` value of the executable to point to the build directory to find the libraries. Others might prefer temporary overrides using the `LD_LIBRARY_PATH` environment variable, or permanently copy the libraries to /usr/lib. Lets use the `LD_LIBRARY_PATH` method for simplicity, by setting the environment variable and running the application in one line: `LD_LIBRARY_PATH=. ./a.out`. Now, the program runs successfully.

## Linking to the Wrong Shared Library

Since the linker at build time and the dynamic linker at runtime may search for the library in different ways, there are times when you might still get a runtime linker issue even if the build succeeds. To debug these issues, we need to confirm if we linked to the right library both at build time and at runtime.

At build time, the libraries that were linked to can be found by simply looking at the `-L` arguments passed to the linker, or, less commonly, the `LIBRARY_PATH` environment variable.

At runtime, as previously mentioned, the libraries that are dynamically linked are linked by `ld.so`. The `ldd` tool (ldd standing for List Dynamic Dependencies) can be used to figure out which libraries were linked. Running `ldd ./a.out` gives output like this:

```console
$ ldd ./a.out
linux-vdso.so.1 (0x00007ffeab351000)
libfoo.so => ./libfoo.so (0x00007fa5745d9000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fa5743a5000)
/lib64/ld-linux-x86-64.so.2 (0x00007fa5745e5000)
```

Unfortunately, `ldd` does not explain *how* those libraries were found or *why* those specific libraries were chosen over others. To figure that out, a more verbose approach is required: the `LD_DEBUG` environment variable can be set to `libs`, and it will print out the paths that `ld.so` searched for, and why it made that specific decision. The full output of `LD_DEBUG=libs ./a.out` is a too long to share here, but a summarized version looks like this:

```console
$ LD_DEBUG=libs ./a.out 
     13551:     find library=libfoo.so [0]; searching
     13551:      search path=./x86_64:.                (LD_LIBRARY_PATH)
     13551:       trying file=./x86_64/libfoo.so
     13551:       trying file=./libfoo.so
     13551:
     13551:     find library=libc.so.6 [0]; searching
     13551:      search path=./x86_64:.                (LD_LIBRARY_PATH)
     13551:       trying file=./libc.so.6
     13551:      search cache=/etc/ld.so.cache
     13551:       trying file=/lib/x86_64-linux-gnu/libc.so.6
```

This works because `ld.so` reads the `LD_DEBUG` environment variable and prints this information if it is set to `libs`. `LD_DEBUG` can also be set to other values, like `all` or `symbols`. More information is available on the `ld.so` man page.

## Missing Symbols

Sometimes, there may be multiple versions of a library in your system. If the different versions expose different symbols (functions), you may end up in a situation where your build succeeds, but your executable fails to load with an error message like this:

```console
$ ./a.out
./a.out: symbol lookup error: ./a.out: undefined symbol: foo
```

Ideally, this would have been an error message that occurred during the linking step as part of your build process instead of happening at runtime. What is happening here is that the definition for `foo` is missing. In this example, it should have been a part of `libfoo.so`, but perhaps `libfoo.so` had changed over time, and the definition for `foo` disappeared.

Since this issue did not occur during the build step, there likely *is* a `libfoo.so` somewhere that contains the symbol you need, but that library is not used by the dynamic linker. Refer to the previous sections to figure out how to debug the dynamic linker search paths and point it to the right library to use.

## Explicitly Link to Dependencies

Sometimes, your application may depend on a library, libfoo.so, and the library may depend on another library, libbar.so. In those cases, when your application is loaded and executed, libbar.so will also be loaded into memory. However, if your application uses a symbol from libbar.so, you must explicitly link your application to it. Otherwise, you will come across an error message as follows:

```console
$ gcc main.o -lfoo
error adding symbols: DSO missing from command line
```

This is because the libbar.so dependency of libfoo.so is an implementation detail; another implementation of libfoo.so may not have that dependency. Your application must not depend on the implementation details of its dependencies. So, if you use symbols from both libfoo.so and libbar.so, you must add both `-lfoo` and `-lbar` to your command line.

```console
$ gcc main.o -lfoo -lbar
```

# Modifying Binaries

If you have compiled and linked an application or binary but would like to modify some of its parameters, you may use the patchelf utility. It allows you to:

1. Set an r-path (that would have been set by the `ld` linker using the `-rpath` option)
2. Remove/add/replace needed libraries from a binary
3. Set the name of a library

More information and capabilities can be found on its github page: <https://github.com/NixOS/patchelf>

# Concluding Remarks

Linking is not something most developers have to think about, but it is an important step in the build and release process. Knowing the stages of linking and the roles of the static and dynamic linker are key to understanding error messages and how to resolve them.

It’s also important to know the tools we have available: we touched on how to inspect the command line arguments (`-l` and `-L`), environment variables (`LD_LIBRARY_PATH` and `LD_DEBUG`), and tools, like `ldd` or `patchelf`.

Each stage of building and releasing has different quirks to it, but you now know how to start your debugging journey.
