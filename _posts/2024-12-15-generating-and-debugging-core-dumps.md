---
categories: [cpp]
date: '2024-12-15T01:05:13-05:00'
layout: post
permalink: /c/generating-and-debugging-core-dumps/
tags: [c, cpp, debugging]
title: Generating and Debugging Core Dumps
---

We’ve all been there: you build your application, run it, and see it crash: possibly with an error message like `Segmentation fault (core dumped)`.

The `(core dumped)` part is what we’ll discover today. Most of the information on this page is documented more concisely in the [core dump manual](https://www.man7.org/linux/man-pages/man5/core.5.html).

## What is a Core Dump?

When an application is launched, it becomes a process that is tracked by the operating system, and uses up memory (RAM) to store its instructions (compiled code) and data (the variables in your code). A core dump is a file containing the memory that a process was using at the moment it crashed.

Similar to how an application can be run through a debugger (like [gdb](https://sourceware.org/gdb/current/onlinedocs/gdb)) to add breakpoints and inspect values, a core dump can be analyzed through a debugger to see what was going on at the moment the process crashed.

# Generating Core Dumps

If you’re lucky, when you get a message saying something like `Aborted (core dumped)`, you’ll have a file called `core` in your current working directory.

Rarely are you so lucky.

When an application would like to generate a core dump, there are a few things that need to happen:

1. The process needs to abort or be killed by a signal that generates a core dump
2. The kernel needs to be configured to generate the core dumps
3. The shell that launched the process needs to have a non-zero max core file size
4. The process needs to be able to write to the corresponding core dump file

## Testing the Core Dump Generation

An easy way to check whether core dumps are generated is to run `sleep 60` and kill it by pressing Ctrl+\\. That key combination will send a `SIGQUIT` signal, which is similar to the `SIGINT` signal sent by pressing Ctrl+C, except that `SIGINT` doesn’t generate a core dump. If a core dump appears in your current directory (likely something called `core`), then you probably wouldn’t be reading this.

## Signals Generating Core Dumps

A process may manually generate a core dump by calling the `abort` function in the standard C library. If you wish to keep your own process running, you can call `fork` to create an identical child process and `abort` that one instead.

A process may also generate a core dump if it was killed by a signal that generates core dumps. Below are some common signals with my commentary:

```
SIGSEGV      Segmentation fault; likely some invalid pointer dereference
SIGABRT      Calling abort(), or not handling an exception in C++
SIGQUIT      Quit from keyboard (Ctrl+\)
```

For a full list of signals that generate core dumps, refer to the [signals man page](https://man7.org/linux/man-pages/man7/signal.7.html).

## Configuring kernel.core\_pattern

The Linux Kernel has some parameters that can be modified at runtime. You can view all of these parameters by running `sysctl -a`.

The parameter that configures where core dump files go is `kernel.core_pattern` ([documentation](https://docs.kernel.org/admin-guide/sysctl/kernel.html#core-pattern)). You can find where it goes by running `sysctl kernel.core_pattern` (or by reading the `/proc/sys/kernel/core_pattern` file). By default, it should be set to `core`, meaning that if an application produces a core dump it will appear in your current working directory as a file named `core`. However, the default value is usually modified by the Linux distro being used.

In any case, you can restore the default value (`core`) by modifying `kernel.core_pattern` in one of two ways:

1. You can run `sudo sysctl kernel.core_pattern=core`
2. You can overwrite the file `/proc/sys/kernel/core_pattern`. This requires `sudo`.

I personally recommend setting the pattern to `core-%e-%t`, which appends the application name (`%e`) and a timestamp (`%t`). This makes it easier to organize and identify a core dump when dealing with many of them.

Since this is a kernel parameter, the `sudo sysctl` command will need to be run outside of any docker containers. If the core file pattern refers to an absolute path, it will be the absolute path in the host machine, not in a container.

To set this pattern permanently, you can add the line `kernel.core_pattern = core-%e-%t` to `/etc/sysctl.conf`, or to another .conf file under `/etc/sysctl.d/`.

## Configuring ulimit

The `kernel.core_pattern` parameter was a **kernel-wide limit**, affecting everything. There are also some **per-user** or **per-process** limits. These limits can be found by running `ulimit -a` (or by running `cat /proc/$$/limits`). `ulimit` is not (and cannot be) an executable: it is a shell builtin command because it needs to modify the running shell, and is documented for bash [here](https://www.man7.org/linux/man-pages/man1/bash.1.html). There is more info on `ulimit` [here](https://ss64.com/bash/ulimit.html). The `csh` shell uses a builtin called `limit`, and the syntax is slightly different, see [here](https://linux.die.net/man/1/csh).

There is a concept of a **hard limit** and a **soft limit**. The soft limit is the effective limitation, and can be configured by the user to be increased up to a maximum of the hard limit. Privileged (root) processes can increase or decrease the hard limit. Non-privileged processes can decrease the hard limit, but usually cannot increase it back to the original hard limit. You can add a `-H` or `-S` to `ulimit` to print the hard or soft limit respectively.

The limit relevant to core dumps is the “core file size” limit, and can be printed by running `ulimit -c`. If this prints out 0, then core dumps will not be generated. You can increase the limit to unlimited by running `ulimit -c unlimited`. Do not use `sudo`. Fortunately, the hard limit is *usually* already set to `unlimited`.

`ulimit` settings are inherited by sub-shells, but they do not persist, nor do they affect parent processes. To set limits permanently, you can modify [`/etc/security/limits.conf`](https://www.man7.org/linux/man-pages/man5/limits.conf.5.html).

Possible causes and solutions of permission errors are:

1. You are trying to set the soft limit higher than the hard limit 
    - Solution: Increase the hard limit first
2. You are trying to increase the hard limit but are not root 
    - Solution: Increase the hard limits for your user by modifying /etc/security/limits.conf (and rebooting), or run your application as root
    - Reminder: `root` in a docker container does not mean that you have `root` permissions in the host.

## Other Possible Issues

Fixing the `kernel.core_pattern` and `ulimit -c` settings solve most problems. If core dumps are still not being generated, there may be issues during the writing of the core file itself. For example, the destination directory may not be writable. This can happen if the directory does not exist, if it has a missing write permission, if the filesystem ran out of space or if it was mounted without write permissions. Another reason could be if the destination file already exists and cannot be overwritten.

There are some other edge cases that are covered in the [core dump manual](https://www.man7.org/linux/man-pages/man5/core.5.html).

# Debugging Core Dumps

Lets use the following code sample to generate a core dump and debug it:

```c
// main.c
#include <stdio.h>

void foo() {
    int* ptr = NULL;
    printf("*ptr = %d\n", *ptr);
}

int main() {
    foo();
    return 0;
}
```

We’ll walk through different setups to get a full fledged debug experience.

## Missing Debug Symbols

If you build your application by running `gcc -o app main.c` and run it (`./app`), it should immediately print `Segmentation fault (core dumped)`. If it doesn’t, or if you cannot find where the core file was dumped, follow the instructions in the first section to generate them.

With a core file named `core`, you can use the gdb debugger to inspect the core dump by running `gdb app core`. You should get something like the following:

```
Reading symbols from app...
(No debugging symbols found in app)
...
Core was generated by `./app'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0x00005633ff9aa161 in foo ()
```

Notice that `gdb` is able to determine that the program crashed when running a function called `foo`. If you type in `backtrace` at the gdb prompt, you should see where this function got called:

```
(gdb) backtrace
#0  0x00005633ff9aa161 in foo ()
#1  0x00005633ff9aa18e in main ()
```

So `foo` was called from `main`. This is good information, but we can do better if we resolve gdb’s warning at the start: `No debugging symbols found in app`.

## Adding Debug Symbols

If you now compile your program by adding the [-g compile flag](https://gcc.gnu.org/onlinedocs/gcc-14.2.0/gcc/Debugging-Options.html) like so: `gcc -g -o app main.c`, you will now have debug symbols in your debugger. If you’re using CMake, you should use the `-DCMAKE_BUILD_TYPE=Debug` or `-DCMAKE_BUILD_TYPE=RelWithDebInfo` options to enable debug information.

With debug information in your application (you don’t need to regenerate another core dump), `gdb app core` will print:

```
Reading symbols from app...
warning: exec file is newer than core file.
...
Core was generated by `./app'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0x0000562332807161 in foo () at main.c:5
5           printf("*ptr = %d\n", *ptr);
```

Now, we can see exactly which line was running (`main.c:5`) when the program crashed. Getting the stack trace also provides more information:

```
(gdb) backtrace
#0  0x0000562332807161 in foo () at main.c:5
#1  0x000056233280718e in main () at main.c:9
```

This is the ideal setup.

## Copying Another Environment

Sometimes, you may receive a core dump generated in another environment. If the environment is too different (e.g. different hardware/platform), then you will need to use a cross-compiled gdb, which is beyond this scope. However, if the environment is only *slightly* different, like another distro running on the same platform, or a docker container running on the same host, then you should be able to debug those core dumps as well. We’ll call the other environment the **target** environment, and the debugging environment (where you’ll run gdb) the **host** environment.

Not only do you need to retrieve the core dump (lets call it `core-target`) from the target environment, you also need to retrieve the application (`app-target`) from that target if you’re not able to generate the **exact** same one locally. Sometimes, you may be able to run `gdb app-target core-target` and begin debugging immediately. Other times, you won’t be so lucky:

```
Reading symbols from app...
...
Core was generated by `./app'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0x0000557f95e9a149 in ?? ()
(gdb) backtrace
#0  0x0000557f95e9a149 in ?? ()
#1  0x0000557f95e9a180 in ?? ()
#2  0x0000000000000000 in ?? ()
```

gdb doesn’t know how to resolve any of those symbols. Quick note: another reason for gdb not being able to resolve any symbols is if the core dump does not match the application, i.e. you are trying to debug a core dump for app1 as if it came from app2, or even from a different version of app1. Be careful.

Applications run code from more than just the application that was built: they also dynamically load a number of shared libraries. You can copy those shared libraries to a local directory and inform gdb to look in that directory by running `set sysroot ${directory}` and `set solib-search-path ${directory}`.

For example, to debug an application that crashes in a docker container, you may need to run something like:

```bash
CONTAINER_NAME=MyContainer
LOCAL_SYSROOT=sysroot
LIB_PATHS="/lib/. /lib64/. /usr/lib/. /usr/lib64/. /usr/local/lib/."

mkdir -p ${LOCAL_SYSROOT}
for LIB_PATH in ${LIB_PATHS}; do
  mkdir -p ${LOCAL_SYSROOT}${LIB_PATH}
  docker cp ${CONTAINER_NAME}:${LIB_PATH} ${LOCAL_SYSROOT}${LIB_PATH}
done

# ...
gdb -ex "set sysroot ${LOCAL_SYSROOT}" -ex "set solib-search-path ${LOCAL_SYSROOT}" app core"
```

Adjust the LIB\_PATHS to the library paths on your specific system. Don’t forget to inform `gdb` of the new `sysroot` by running the `set sysroot` and `set solib-search-path` commands mentioned above.

## Conclusion

Core dumps are a handy way of conducting a post-mortem analysis to debug a crashing program. We’ve gone over how to generate core dumps, and how to begin analyzing them in gdb. We did not go over the various features supported by gdb: there are enough tutorials for that going around.
