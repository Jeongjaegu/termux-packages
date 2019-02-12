# Build Documentation

This document is intended to describe how to build a package.

## Flow of a Build

### Basics

Package build flow is controlled by script [build-package.sh](../build-package.sh) and is split into the following stages:

1. Sets up a patched stand-alone Android NDK toolchain if necessary.

2. Reads `packages/$PKG/build.sh` to find out where to find the source code of the package and how to build it.

3. Extracts the source in `$HOME/.termux-build/$PKG/src`.

4. Applies all patches in packages/$PKG/\*.patch.

5. Builds the package under `$HOME/.termux-build/$PKG/` (either in the build/ directory there or in the
  src/ directory if the package is specified to build in the src dir) and installs it to `$PREFIX`.

6. Extracts modified files in `$PREFIX` into `$HOME/.termux-build/$PKG/massage` and massages the
  files there for distribution (removes some files, splits it up in sub-packages, modifies elf files).

7. Creates a deb package file for distribution in `debs/`.

### Details Table

| Order | Function Name | Overridable | Description |
| -----:|:-------------:| -----------:|:----------- |
| 0.1   | `error_exit` | no | Stop script and output error. |
| 0.2   | `download` | no | Utility function to download any file. |
| 0.3   | `setup_golang` | no | Setup Go Build environment. |
| 0.4   | `setup_rust` | no | Setup Cargo Build. |
| 0.5   | `setup_ninja` | no | Setup Ninja make system. |
| 0.6   | `setup_meson` | no | Setup Meson configure system. |
| 0.7   | `setup_cmake` | no | Setup CMake configure system. |
| 1     | `handle_arguments` | no | Handle command line arguments. |
| 2     | `setup_variables` | no | Setup essential variables like directory locations and flags. |
| 3     | `handle_buildarch` | no | Determines architecture to build for. |
| 4     | `get_repo_files` | no | Install dependencies if `-i` option supplied. |
| 4.1   | `download_deb` | no | Download packages for installation |
| 5     | `start_build` | no | Setup directories and files required. Read `build.sh` for variables. |
| 6     | `extract_package` | yes | Download source package. |
| 7     | `post_extract_package` | yes | Hook to run commands before host builds. |
| 8     | `handle_host_build` | yes | Determine whether a host build is required. |
| 8.1   | `host_build` | yes | Conduct a host build. |
| 9     | `setup_toolchain` | no | Setup C Toolchain from Android NDK. |
| 10    | `patch_package` | no | Patch all `*.patch` files as specified in the package directory. |
| 11    | `replace_guess_scripts` | no | Replace `config.sub` and `config.guess` scripts. |
| 12    | `pre_configure` | yes | Hook to run commands before configures. |
| 13    | `configure` | yes | Determine the configure method. |
| 13.1  | `configure_autotools` | no | Run `configure` by GNU Autotools. |
| 13.2  | `configure_cmake` | no | Run `cmake`. |
| 13.3  | `configure_meson` | no | Run `meson`. |
| 14    | `post_configure` | yes | Hook to run commands before make. |
| 15    | `make` | yes | Make the package. |
| 16    | `make_install` | yes | Install the package. |
| 17    | `post_make_install` | yes | Hook before extraction. |
| 18    | `extract_into_massagedir` | No with `make_install` | Extracts installed files. |
| 19    | `massage` | no | Remove unusable files and creates subpackages. |
| 20    | `post_massage` | yes | Final hook before packaging. |
| 21    | `create_datatar` | no | Archive package files. |
| 22    | `create_debfile` | no | Create package. |
| 22.1  | `create_debscripts` | yes | Create additional Debian package files. |
| 23    | `compare_debfiles` | no | Compare packages if `-i` option is specified. |
| 24    | `finish_build` | no | Notification of finish. |

Order specifies function sequence. 0 order specifies utility functions.

Suborder specifies a function triggered by the main function. Functions with different suborders are not executed simultaneously.

For more detailed descriptiom on each step, you can read [build-package.sh](../build-package.sh)

## Normal Build Process

Remarks: Software Developers should provide build instructions. Otherwise good luck trying how to build :joy:.

Follow the instructions until you get a working build. If a build succeeds after any step, skip the remaining steps.

1. Create a `build.sh` file using the [sample package template](sample/build.sh).

2. Create a `subpackage.sh` for each subpackage using the [sample package template](sample/subpackage.sh).

3. Run `./build-package.sh $PKG` to see what errors are found.

4. If any steps complain about an error line, first copy the file to another directory.

5. Edit the original file.

6. When tests succeed for the file, create a patch by `diff -u <original> <new> > packages/<pkg>/<filename>.patch`

7. Repeat steps 2-4 for each error file.

8. If extra configuration or make arguments are needed, specify in `build.sh` as shown in sample package.

9. If there are subpackages, include them in `subpackage.sh`.

10. (optional but appreciated) Test the package by yourself.

## Common Porting Problems

- The Android bionic libc does not have iconv and gettext/libintl functionality built in. A `libandroid-support` package contains these and may be used by all packages.

- "error: z: no archive symbol table (run ranlib)" usually means that the build machines libz is used instead of the one for cross compilation, due to the builder library -L path being setup incorrectly.

- rindex(3) does not exist, but strrchr(3) is preferred anyway.

- &lt;sys/termios.h&gt; does not exist, but &lt;termios.h&gt; is the standard location.

- &lt;sys/fcntl.h&gt; does not exist, but &lt;fcntl.h&gt; is the standard location.

- &lt;sys/timeb.h&gt; does not exist (removed in POSIX 2008), but ftime(3) can be replaced with gettimeofday(2).

- &lt;glob.h&gt; does not exist, but is available through the `libandroid-glob` package.

- SYSV shared memory is not supported by the kernel. A `libandroid-shmem` package, which emulates SYSV shared memory on top of the [ashmem](http://elinux.org/Android_Kernel_Features#ashmem) shared memory system, is available. Use it with `LDFLAGS+=" -landroid-shmem`.

- SYSV semaphores is not supported by the kernel. Use unnamed POSIX semaphores instead (named semaphores are unimplemented).

### dlopen() and RTLD&#95;&#42; flags

&lt;dlfcn.h&gt; declares
```C
RTLD_NOW=0; RTLD_LAZY=1; RTLD_LOCAL=0; RTLD_GLOBAL=2;       RTLD_NOLOAD=4; // 32-bit
RTLD_NOW=2; RTLD_LAZY=1; RTLD_LOCAL=0; RTLD_GLOBAL=0x00100; RTLD_NOLOAD=4; // 64-bit
```
These differs from glibc ones in that

1. They differ in value from glibc ones, so cannot be hardcoded in files (DLFCN.py in python does this)

2. They are missing some values (`RTLD_BINDING_MASK`, ...)

### Android Dynamic Linker

The Android dynamic linker is located at `/system/bin/linker` (32-bit) or `/system/bin/linker64` (64-bit). Here are source links to different versions of the linker:

- [Android 5.0 linker](https://android.googlesource.com/platform/bionic/+/lollipop-mr1-release/linker/linker.cpp)

- [Android 6.0 linker](https://android.googlesource.com/platform/bionic/+/marshmallow-mr1-release/linker/linker.cpp)

- [Android 7.0 linker](https://android.googlesource.com/platform/bionic/+/nougat-mr1-release/linker/linker.cpp)

Some notes about the linker:

- The linker warns about unused [dynamic section entries](https://docs.oracle.com/cd/E23824_01/html/819-0690/chapter6-42444.html) with a `WARNING: linker: $BINARY: unused DT entry: type ${VALUE_OF_d_tag}` message.

- The supported types of dynamic section entries has increased over time.

- The Termux build system uses [termux-elf-cleaner](https://github.com/termux/termux-elf-cleaner) to strip away unused ELF entries causing the above mentioned linker warnings.

- Symbol versioning is supported only as of Android 6.0, so is stripped away.

- `DT_RPATH`, the list of directories where the linker should look for shared libraries, is not supported, so is stripped away.

- `DT_RUNPATH`, the same as above but looked at after `LD_LIBRARY_PATH`, is supported only from Android 7.0, so is stripped away.

- Symbol visibility when opening shared libraries using `dlopen()` works differently. On a normal linker, when an executable linking against a shared library libA dlopen():s another shared library libB, the symbols of libA are exposed to libB without libB needing to link against libA explicitly. This does not work with the Android linker, which can break plug-in systems where the main executable dlopen():s a plug-in which doesn't explicitly link against some shared libraries already linked to by the executable. See [the relevant NDK issue](https://github.com/android-ndk/ndk/issues/201) for more information.