# How to publish native executables for (almost) all platforms

## Abstract (in 10 seconds)

By investigating [Ninja](https://github.com/martine/ninja) & Chromium [depot\_tools](https://chromium.googlesource.com/chromium/tools/depot_tools.git/) approach, I'll
introduce the way to mobile your C++ executable for (almost) all platforms.

## Introduction

You sometimes would like to publish native executables. One day,
you make a nice command written in C++, and using it on Linux x64.
You may write a nice zsh script using this nice executable and it
works as you want.

But you may control dotfiles on git repository and sometimes you
need to sync your nice environment between your working machines.
In the meantime, you may build your nice command on each environment.
However, whenever you fix / enhance your native module, you need to
rebuild it on all your machines. Some machines are the same architecture
as your main machine. But some machines may be OSX. Another machines are
Windows, etc...

### Why don't you write it with Script languages?

You need to setup script languages environment on the each machines.
And I sometimes would like to construct commands based on C++ libraries,
such as LLVM.

**NOTE:**
LLVM can be built as static library. So it is possible to bundle LLVM
libraries to one executable file.

### Why don't you write it with golang?

As the same as the above.

## Windows

When adding a directory to PATH, cmd.exe will search a command through PATH
directories. cmd.exe will find {name}.exe executable file.
".exe" extension! It is not (usually) considered in Linux & OSX. So placing
{name}.exe and {name} command in the same directory, you can provide Windows
and Unix binary at the same time.

So this stroy is very easy. The way is buidling command as name.exe and placing
it on the appropriate directory. That's all.

## OSX & Linux

You need to distinguish them. So the easiest way to start it is making {name}
command `sh` script to execute an appropriate binary.

### Switching binary with shell script

In the script, you can use `uname` command to distinguish the platform.
This command works on FreeBSD, OSX and Linux. This is the definition of (almost) all
platforms.

According to the `uname` man page, `uname -s` will answer the kernel name.
For example,

```
    Linux,
    Darwin,
    FreeBSD
```

And you need to consider is that machine architecture (such as x64). You can get
this by uname too. `uname -m` will return hardware architecture.
For example,

```
    amd64  # On FreeBSD
    x86_64 # On Linux
    i386
    i686
    ...
```

You can see uname implementation of FreeBSD [here](https://github.com/freebsd/freebsd/blob/master/usr.bin/uname/uname.c). And
actual code to gain a architecture name is [here](https://github.com/freebsd/freebsd/blob/6fbc4f383bfcc743bdf31fcae77e28bc91b04a42/sys/kern/kern_mib.c#L246).
And architecture candidates on FreeBSD seems the same to [this](https://github.com/freebsd/freebsd/blob/9075f210e87b68140c7e82d0a6d97e8f37688d1a/sys/Makefile#L18).

```
amd64 arm i386 ia64 mips pc98 powerpc sparc64 x86
```

And on the Linux case. According to the Linux's Makefile ([here](https://github.com/torvalds/linux/blob/fa389e220254c69ffae0d403eac4146171062d08/Makefile#L171)), we can get subarchitecture by this script.
```sh
uname -m | sed -e s/i.86/x86/ -e s/x86_64/x86/ \
               -e s/sun4u/sparc64/ \
               -e s/arm.*/arm/ -e s/sa110/arm/ \
               -e s/s390x/s390/ -e s/parisc64/parisc/ \
               -e s/ppc.*/powerpc/ -e s/mips.*/mips/ \
               -e s/sh[234].*/sh/ -e s/aarch64.*/arm64/
```

So by adding `amd64` case to it, we can get the command line to get sub architecture.

```sh
uname -m | sed -e s/i.86/x86/ -e s/x86_64/x86/ -e s/amd64/x86/ \
               -e s/sun4u/sparc64/ \
               -e s/arm.*/arm/ -e s/sa110/arm/ \
               -e s/s390x/s390/ -e s/parisc64/parisc/ \
               -e s/ppc.*/powerpc/ -e s/mips.*/mips/ \
               -e s/sh[234].*/sh/ -e s/aarch64.*/arm64/
```

And finally, you need to get if CPU is 64bit / 32bit. To get it on Linux, FreeBSD and Darwin,
you can use `getconf LONG_BIT` according to the depot\_tools source.

Then here's the complete switch script.

```sh
#!/bin/sh
OS="$(uname -s)"
ARCH="$(uname -m | sed -e s/i.86/x86/ -e s/x86_64/x86/ -e s/amd64/x86/ \
                       -e s/sun4u/sparc64/ \
                       -e s/arm.*/arm/ -e s/sa110/arm/ \
                       -e s/s390x/s390/ -e s/parisc64/parisc/ \
                       -e s/ppc.*/powerpc/ -e s/mips.*/mips/ \
                       -e s/sh[234].*/sh/ -e s/aarch64.*/arm64/)"
BIT="$(getconf LONG_BIT)"
COMMAND="${0}-${OS}-${ARCH}-${BIT}"

if [ -e "$COMMAND" ]; then
    exec "$COMMAND"
else
    echo "Command ${COMMAND} not found."
    echo "Unsupported architecture ${OS} ${ARCH} ${BIT}"
fi
```

So placing {name}-Linux-x86-64 in the same directory to this script,
you can make command available on Linux x64 environment.

### Building executable -static option

Native executables (built by g++, clang or so) usually depends on libc, libgcc, libcompiler-rt, libsup++, libc++ and so on.
But as you may know, you can build native executables not depending on them. You can use `-static` option.

`g++ main.cc -static`

Then g++ produces a executable not depending on the above libraries. You can ensure it by executing `ldd {name}`.
So moving it to the other platform, it works correctly if OS, ARCH, BIT are the same.

**NOTE:**
Without `-static` option, `ldd a.out` result may be the following.
```
$ ldd a.out
        linux-vdso.so.1 =>  (0x00007fff799fe000)
        libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f67bd054000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f67bcc8c000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f67bc987000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f67bd383000)
        libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f67bc771000)
```
First, `linux-vdso.so.1` is contained in the linux kernel (Linux Virtual Dynamic Shared Objects).
So it isn't needed as a shared library.
And since `libstdc++.so.6`, `libc.so.6`, `libm.so.6`, `libgcc_s.so.1` and `/lib64/ld-linux-x86-64.so.2` are placed with the
same name in a lot of Linux distributions. So if you don't worry about the edge case, `-static` is not necessary.
Compiled binary is basically works on the other Linux system.
But of cource, the other libraries should be linked statically.
[This](http://stackoverflow.com/questions/4156055/gcc-static-linking-only-some-libraries) shows how to control linking with libraries.

### Cross compiling the 32 / 64 bit 

Just googled and found the [article](http://aaronbonner.io/post/14969163463/cross-compiling-to-32bit-with-gcc) and [this](http://stackoverflow.com/questions/4643197/missing-include-bits-cconfig-h-when-cross-compiling-64-bit-program-on-32-bit).
Especially, the latter article works correctly on my environment (Ubuntu 13.10, FreeBSD 10)

## Result

So I've created the sample project `portable` to test it. It includes the above script and executables.
(Windows 32bit, OSX 64bit, Linux 32bit, Linux 64bit. Their architecture is x86)

So just copying (or git clone) it and append this directory to the PATH,
you can *sync*, *execute*, *publish* this command on the (almost) all platforms.
