---
categories: [cpp]
date: '2024-12-15T01:05:13-05:00'
layout: post
permalink: /c/generating-and-debugging-core-dumps/
tags: [c, cpp, debugging]
title: Generating and Debugging Core Dumps
---

## Intro

We’ve all been there: you build your application, run it, and see it crash: possibly with an error message like `Segmentation fault (core dumped)`.

The `(core dumped)` part is what we’ll discover today. Most of the information on this page is documented more concisely in the [core dump manual](https://www.man7.org/linux/man-pages/man5/core.5.html).

## What is a Core Dump?

When an application is launched, it becomes a process that is tracked by the operating system, and uses up memory (RAM) to store its instructions (compiled code) and data (the variables in your code). A core dump is a file containing the memory that a process was using at the moment it crashed.

Similar to how an application can be run through a debugger (like [gdb](https://sourceware.org/gdb/current/onlinedocs/gdb)) to add breakpoints and inspect values, a core dump can be analyzed through a debugger to see what was going on at the moment the process crashed.

## Testing the Core Dump Generation

An easy way to check whether core dumps are generated is to run `sleep 60` and kill it by pressing `Ctrl+\`. That key combination is similar to the more well-known `Ctrl+C` combination, except that `Ctrl+C` sends a `SIGINT` signal, whereas `Ctrl+\` sends a `SIGQUIT` signal to the process. `SIGINT` doesn’t generate a core dump, whereas `SIGQUIT` does - more on that later. If a core dump appears in your current directory (likely something called `core`), then congratulations! You can skip to the [Debugging Core Dumps](#debugging-core-dumps) section. Otherwise, keep reading to figure out how core dumps are generated and how to troubleshoot missing core dump issues.

## Generating Core Dumps

If you’re lucky, when you get a message saying something like `Aborted (core dumped)`, you’ll have a file called `core` in your current working directory.

Rarely are you so lucky.

When an application would like to generate a core dump, there are a few things that need to happen:

1. The process needs to abort or be killed by a signal that generates a core dump
2. The kernel needs to be configured to generate the core dumps
3. The shell that launched the process needs to have a non-zero max core file size
4. The process needs to be able to write to the corresponding core dump file


### Signals Generating Core Dumps

A process may manually generate a core dump by calling the `abort` function in the standard C library. If you wish to keep your own process running, you can call `fork` to create an identical child process and `abort` that one instead.

A process may also generate a core dump if it was killed by a signal that generates core dumps. Below are some common signals with my commentary:

```
SIGSEGV      Segmentation fault; likely some invalid pointer dereference
SIGABRT      Calling abort(), or not handling an exception in C++
SIGQUIT      Quit from keyboard (Ctrl+\)
```

For a full list of signals that generate core dumps, refer to the [signals man page](https://man7.org/linux/man-pages/man7/signal.7.html).

### Configuring kernel.core\_pattern

The Linux Kernel has some parameters that can be modified at runtime. You can view all of these parameters by running `sysctl -a`.

The parameter that configures where core dump files go is `kernel.core_pattern` ([documentation](https://docs.kernel.org/admin-guide/sysctl/kernel.html#core-pattern)). You can find where it goes by running `sysctl kernel.core_pattern` (or by reading the `/proc/sys/kernel/core_pattern` file). By default, it should be set to `core`, meaning that if an application produces a core dump it will appear in your current working directory as a file named `core`. However, the default value is usually modified by the Linux distro being used.

In any case, you can restore the default value (`core`) by modifying `kernel.core_pattern` in one of two ways:

1. You can run `sudo sysctl kernel.core_pattern=core`
2. You can overwrite the file `/proc/sys/kernel/core_pattern`. This requires `sudo`.

I personally recommend setting the pattern to `core-%e-%t`, which appends the application name (`%e`) and a timestamp (`%t`). This makes it easier to organize and identify a core dump when dealing with many of them.

Since this is a kernel parameter, the `sudo sysctl` command will need to be run outside of any docker containers. If the core file pattern refers to an absolute path, it will be the absolute path in the host machine, not in a container.

To set this pattern permanently, you can add the line `kernel.core_pattern = core-%e-%t` to `/etc/sysctl.conf`, or to another .conf file under `/etc/sysctl.d/`.

### Configuring ulimit

The `kernel.core_pattern` parameter was a **kernel-wide limit**, affecting everything. In the Linux Kernel, there are also some **per-user** or **per-process** limits. These limits can be found by running `ulimit -a` (or by running `cat /proc/$$/limits`). `ulimit` is not (and cannot be) an executable: it is a shell builtin command because it needs to modify the running shell, and is documented for bash [here](https://www.man7.org/linux/man-pages/man1/bash.1.html). The `csh` shell uses a builtin called `[limit](https://linux.die.net/man/1/csh)`, and the syntax is slightly different.

There are two types of limits: a **hard limit** and a **soft limit**. The soft limit is the effective limitation, and can be configured by the user to be increased up to a maximum of the hard limit. Privileged (root) processes can increase or decrease the hard limit. Non-privileged processes can decrease the hard limit, but usually cannot increase it back to the original hard limit. You can add a `-H` or `-S` to `ulimit -a` to print the hard or soft limit respectively.

> The process limits within a `sudo` command may be different than the limits in your normal shell; you can verify those limits by running `sudo sh -c "ulimit -Hc && ulimit -Sc"`.
{: .prompt-info }

The limit relevant to core dumps is the “core file size” limit, and can be printed by running `ulimit -c`. This value is the maximum size of a generated coredump in kilobytes. If set to 0, then core dumps will not be generated at all. You can increase the limit to unlimited by running `ulimit -c unlimited`. Do not use `sudo`. Fortunately, the hard limit is *usually* already set to `unlimited`. The following shows an example of reading the hard and soft limits, then increasing the soft limit to the maximum.

```console
$ ulimit -Hc           # Read the Hard core limit
unlimited
$ ulimit -Sc           # Read the Soft core limit
0
$ ulimit -c unlimited  # Set the (soft) limit to unlimited
$ ulimit -c            # Equivalent to ulimit -Sc
unlimited
```

Like environment variables, `ulimit` settings are inherited by sub-processes, including sub-shells, but they do not affect parent processes, nor do they persist over reboots. To set limits permanently, you can modify [`/etc/security/limits.conf`](https://www.man7.org/linux/man-pages/man5/limits.conf.5.html). As an example, the following lines set the hard limit to unlimited and the soft limit to 0 for all users:

```conf
# /etc/security/limits.conf
*       hard    core    unlimited
*       soft    core    0
```

> On some systems, `*` may not include the `root` user for security purposes.
{: .prompt-warning }

If you run into permission errors when trying to increase a soft limit, verify that the hard limit is higher than what you are setting the soft limit to. If you get permission errors when setting the hard limit, ensure you have root privileges, or try to increase your users' limits through the `/etc/security/limits.conf` configuration  instead (and reboot).

If you need to set the limits in a process that is run automatically you may need to configure the global limits through `/etc/security/limits.conf`.

There are some more complications that may occur within Docker containers, which are expanded upon in the next section.

### Docker Containers
Docker containers bring in a new level of complexity as it pertains to process limits. The limit can be set when the container is [built](https://docs.docker.com/reference/cli/docker/buildx/build/#ulimit), is [run](https://docs.docker.com/reference/cli/docker/container/run/#ulimit), or they can be set through the Docker daemon configuration, [dockerd](https://docs.docker.com/reference/cli/dockerd/#default-ulimit-settings).

If none of those are set, then the values are taken from the Docker daemon process itself: how was the Docker daemon started? Was it started as a system service, or was it started manually by running `sudo dockerd &`?

> To verify the limits within a docker container, you can use the official `bash` container:
> `docker run bash sh -c "ulimit -Hc && ulimit -Sc"`
{: .prompt-info }

The easiest way to configure the limits within docker is to use the [docker daemon configuration file](https://docs.docker.com/reference/cli/dockerd/#daemon-configuration-file) to configure the ulimits used by the docker daemon globally. [On Linux](https://docs.docker.com/reference/cli/dockerd/#on-linux), this is set to `/etc/docker/daemon.json`. Here's an example of a configuration file that sets the hard limit to unlimited and the soft limit to 0:

```json
{
  "default-ulimits": {
    "core": {
      "Name": "core",
      "Hard": -1,
      "Soft": 0
    }
  }
}
```

The `"Name"` field does not choose the name of the core dump file; it's just the way Docker parses the configuration json. Since Docker uses the host's kernel, the core dump filename will follow the same format as the host machine. If the filename contains an absolute path, ensure that the path exists within the docker container.

### Other Possible Issues

Fixing the `kernel.core_pattern` and `ulimit -c` settings solve most problems. If those settings are valid and core dumps are still not being generated, there may be issues during the writing of the core file itself. For example, the destination directory may not be writable. This can happen if the directory does not exist, if it has a missing write permission, if the filesystem ran out of space or if it was mounted without write permissions. Another reason could be if the destination file already exists and cannot be overwritten.

There are some other edge cases that are covered in the [core dump manual](https://www.man7.org/linux/man-pages/man5/core.5.html) that mainly revolve around the access permissions of the directory or of the running process' binary.

## Debugging Core Dumps

Assuming that core dumps are being generated correctly, lets use the following code sample to generate a core dump and debug it:

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

### Missing Debug Symbols

If you build your application by running `gcc -o app main.c` and run it (`./app`), it should immediately print `Segmentation fault (core dumped)`. If it doesn’t, or if you cannot find where the core file was dumped, follow the instructions in the first section to generate them.

With a core file named `core`, you can use the gdb debugger to inspect the core dump by running `gdb app core`. You should get something like the following:

```console
Reading symbols from app...
(No debugging symbols found in app)
...
Core was generated by `./app'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0x00005633ff9aa161 in foo ()
```

Notice that `gdb` is able to determine that the program crashed when running a function called `foo`. If you type in `backtrace` at the gdb prompt, you should see where this function got called:

```console
(gdb) backtrace
#0  0x00005633ff9aa161 in foo ()
#1  0x00005633ff9aa18e in main ()
```

So `foo` was called from `main`. This is good information, but we can do better if we resolve gdb’s warning at the start: `No debugging symbols found in app`.

### Adding Debug Symbols

If you now compile your program by adding the [-g compile flag](https://gcc.gnu.org/onlinedocs/gcc-14.2.0/gcc/Debugging-Options.html) like so: `gcc -g -o app main.c`, you will now have debug symbols in your debugger. If you’re using CMake, you should use the `-DCMAKE_BUILD_TYPE=Debug` or `-DCMAKE_BUILD_TYPE=RelWithDebInfo` options to enable debug information.

With debug information in your application (you don’t need to regenerate another core dump), `gdb app core` will print:

```console
Reading symbols from app...
warning: exec file is newer than core file.
...
Core was generated by `./app'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0x0000562332807161 in foo () at main.c:5
5           printf("*ptr = %d\n", *ptr);
```

Now, we can see exactly which line was running (`main.c:5`) when the program crashed. Getting the stack trace also provides more information:

```console
(gdb) backtrace
#0  0x0000562332807161 in foo () at main.c:5
#1  0x000056233280718e in main () at main.c:9
```

This is the ideal setup - you now have the debug information loaded, and can debug your program with relative ease.

### Debugging Core Dumps from Another System

Sometimes, you may receive a core dump generated in another environment. If the environment is too different (e.g. different hardware/platform), then you will need to use a cross-compiled gdb, which is beyond the scope of this post. However, if the environment is only *slightly* different, like another distro running on the same platform, or a docker container running on the same host, then you should be able to debug those core dumps as well. We’ll call the other environment the **target** environment, and the debugging environment (where you’ll run gdb) the **host** environment.

Not only do you need to retrieve the core dump (lets call it `core-target`) from the target environment, you also need to retrieve the application (`app-target`) from that target if you’re not able to generate the **exact** same one locally. Sometimes, you may be able to run `gdb app-target core-target` and begin debugging immediately. Other times, you won’t be so lucky:

```console
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

gdb doesn’t know how to resolve any of those symbols. 

> Another reason for gdb not being able to resolve any symbols is if the core dump does not match the application, i.e. you are trying to debug a core dump from app2 as if it came from app1, or even from a different version of app1.
{: .prompt-info }

Applications run code from more than just the application that was built: they also dynamically load a number of shared libraries. You can copy those shared libraries to a local directory and inform gdb to look in that directory by running `set sysroot ${directory}` and `set solib-search-path ${directory}`.

For example, to debug an application that crashes in a docker container, you may need to run something like:

```bash
#!/bin/bash
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

Adjust the LIB\_PATHS to the library paths on your specific system.

## Conclusion

Core dumps are a handy way of conducting a post-mortem analysis to debug a crashing program. We’ve gone over how to generate core dumps, and how to begin analyzing them in gdb. We did not go over the various features supported by gdb: there are enough tutorials for that going around.
