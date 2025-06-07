# Advanced Build Environment (ABE)

ABE is a bourne shell rewrite of the original Cbuildv1 system used by
Linaro. Bourne shell was used as Cbuildv was a collection of Makefiles
so it was possible to build on the older code. I noticed recently that
ABE has not been maintained by Linaro for many years, so as the
original author and primary developer I'm going to maintain this fork
and use upstream versions of all the code instead of the very out of
date Linaro versions.

ABE is a complicated program using many advanced bourne shell
features, and looks a bit like C code as most library functions return
a value and or a data structure. To build a cross toolchain, or
crazier a Canadian Cross (Windows hosted cross compiler built on
Linux) is a complicated process and each step must be done in the
right order with the right options. ABE buries all this complexity so
it's relatively easy to buiild a cross compiler for Linux, Android, or
base metal.

## Configuring ABE

While it is possible to run ABE from it's source tree, this
isn't recommended. The best practice is to create a new build
directory, and configure Abe in that directory. That makes it
easy to change branches, or even delete subdirectories. There
are defaults for all paths, which is to create them under the same
directory configure is run in.

There are several directories that ABE needs. These will be created
when you run ABE. These are the following:

* snapshots - This is where all the downloaded sources get
stored. The default is the current directory under snapshots.
* sysroots - This is the sysroot for the target
* builds - This is where all the executables get built. The
default is to have builds go in a top level directory which is the
full hostname of the build machine. Under this a directory is
created for the target architecture.

If configure is executed without any parameters. the defaults are
used. It is also possible to change those values at configure
time. For example:

You can execute ./configure --help to get the full list of configure
time parameters.

The configure process produces a host.conf file, with the default
settings. This file is read by abe at runtime, so it's possible to
change the values and rerun abe.sh to use the new values. Each
toolchain component also has a config file. The default version is
copied at build time to the build tree. It is also possible to edit
this file, usually called something like gcc.conf, to change the
toolchain component specific values used when configuring the
component.

  You can see what values were used to configure each component by
looking in the top of the config.log file, which is produced at
configure time. For example:

	  head ${hostname}/${target}/${component}/config.log

## Default Behaviours

There are several behaviours that may not be obvious, so they're
documented here. One relates to GCC builds. When built natively, GCC
only builds itself once, and is fully functional. When cross
compiling, this works differently. A fully functional GCC can't be
built without a sysroot, so a minimum compiler is built to build the C
library. This is called stage1. This is then used to compiler the C
library. One this library is installed, then stage2 of GCC can be
built.

When building an entire toolchain using the 'all' option to --build,
all components are built in the correct order, including both GCC
stages. It is possible to specify building only GCC, in this case the
stage1 configure flags are used if the C library isn't installed. If
the C library is installed, then the stage2 flags are used. A message
is displayed to document the automatic configure decision.

## Specifying the source works like this

It's possible to specify a full tarball name, minus the compression
part, which will be found dynamically. If the word 'tar' appears in
the suppplied name, it's only looked for on the remote directory. If
the name is a URL for bzr, sv, http, or git, the the source are
instead checked out from a remote repository instead with the
appropriate source code control system.

It's also possible to specify an 'alias', ie ... 'gcc-4.8' instead. In
this case, the remote snapshot directory is checked first. If the name
is not unique, but matches are found, and error is generated. If the
name isn't found at all, then a URL for the source repository is
extracted from the sources.conf file, and the code is checkout out.

There are a few ways if listing the files to see what is
available. There ar two primary remote directories where files that
can be downloaded are stored. These are 'snapshots' or
'infrastructure'. Infrastructure is usualy only installed once per
host, and contains the other packages needed to build GCC, like
gmp. Snapshots is the primary location of all source tarballs. To list
all the available snapshots, you can do this"

    abe.sh --list snapshots

To build a specific component, use the --build option to
abe. the --target option is also used for cross builds. For
example:

	abe.sh --build gcc --target aarch64-linux-android

This would fetch the source tarball for this release, build anything
it needs to compile, the binutils for example, and then build these
sources. You can also specify a URL to a source repository
instead. For example:

	abe.sh --target arm-linux-gnueabihf git://git.linaro.org/toolchain/eglibc.git

To build an entire cross toolchain, the simplest way is to let
ABE control all the steps. Although it is also possible to do each
step separately. To build the full toolchain, do this:

	abe.sh --target arm-linux-gnueabihf --build all

## Features

* Ability to mix & match all toolchain components
* Test locally or remotely
* Builds cross compilers
* Can build a cross toolchain for Android
* Can build Windows hosted cross compilers on Linux
* Builds binary release tarballs

## abe.sh command line arguments:

* --build (architecture for the build machine, default native)
* --target (architecture for the target machine, default native)
* --snapshots XXX (URL of remote host or local directory)
* --libc {newlib,eglibc,glibc} (C library to use)
* --list {gcc,binutils,libc} (list possible values for component versions)
* --set {gcc,binutils,libc,latest}=XXX (change config file setting)
* --binutils (binutils version to use, default $PATH)
* --gcc (gcc version to use, default $PATH)
* --config XXX (alternate config file)
* --clean (clean a previous build, default is to start where it left off)
* --dispatch (run on LAVA build farm, probably remote)
* --sysroot XXX (specify path to alternate sysroot)

## Notes on Bourne Shell Scripting

These scripts use a few techniques in many places that relate to
shell functions. One is heavy use of bourne shell functions to reduce
duplication, and make the code better organized. Any string echo'd by
a function becomes it's return value. Bourne shell supports 2 types of
return values though. One is the string returned by the function. This
is used whenever the called function returns data. This is captured by
the normal shell commands like this: values="`call_function $1`". The
other type of return value is a single integer. Much like system
calls, these scripts all return 0 for success, and 1 for errors. This
enables the calling function to trap errors, and handle them in a
clean fashion.

A few good habits to mention, always enclose a sub shell execution
in double quotes. If the returned string contains spaces, this
preserves the data, otherwise it'll get truncated.

Another good habit is to always prepend a character when doing
string comparisons. If one of the two strings is undefined, the script
will abort. So always using "test x${foo} = xbar" prevents that.

## Adding support for a new target

In order to add support for a missing target, one has to tell
ABE about a few things peculiar to the target:

* which libc to choose: in abe.sh, look for "powerpc" and add similar
  handling code
* select which languages to build: in config/gcc.conf, look for
  "powerpc" again for an example
* convert the target name into the linux name when installing the
  linux kernel headers: in lib/make.sh, look for powerpc again
 
