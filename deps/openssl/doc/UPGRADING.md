## How to upgrade openssl library in Node.js

This document describes the procedure to upgrade openssl from 1.0.2e
to 1.0.2f in Node.js. This procedure might be applied to upgrading
any versions in 1.0.2.

### Build System and Upgrading Overview
The openssl build system is based on the `Configure` perl script in
`deps/openssl/openssl`. For example, running `Configure linux_x86-64`
in the openssl repository generates `Makefile` and `opensslconf.h` for
the linux_x86_64 target architecture.

The `Makefile` contains the list of asm files which are generated by
perl scripts during build so that we can get the most of use of the
hardware performance according to the type of cpus.

`Configure TABLE` shows various build parameters that depend on each
os and arch.

In Node.js, build target is defined as `--dest-os` and `--dest-cpu` in
configure options which are different from the one that is defined in
openssl and it's build system is gyp that is based on python,
therefore we cannot use the openssl build system directly.

In order to build openssl with gyp in node, files of opensslconf.h and
asm are generated in advance for several supported platforms.

Here is a map table to show conf(opensslconf.h) and asm between
the openssl target and configuration parameters of os and cpu in node.
The tested platform in CI are also listed.

| --dest-os | --dest-cpu | conf | asm  | openssl target     | CI |
|---------- |----------- |----- |----- |------------------- |--- |
| aix       | ppc        | o    | x(*2)| aix-gcc            | o  |
| aix       | ppc64      | o    | x(*2)| aix64-gcc          | o  |
| linux     | ia32       | o    | o    |linux-elf           | o  |
| linux     | x32        | o    | x(*2)|linux-x32           | x  |
| linux     | x64        | o    | o    |linux-x86_64        | o  |
| linux     | arm        | o    | o    |linux-arm           | o  |
| linux     | arm64      | o    | o    |linux-aarch64       | o  |
| mac       | ia32       | o    | o    |darwin-i386-cc      | -  |
| mac       | x64        | o    | o    |darwin64-x86_64-cc  | o  |
| win       | ia32       | o    | o(*3)|VC-WIN32            | x  |
| win       | x64        | o    | o    |VC-WIN64A           | o  |
| solaris   | ia32       | o    | o    |solaris-x86-gcc     | o  |
| solaris   | x64        | o    | o    |solaris64-x86_64-gcc| o  |
| freebsd   | ia32       | o    | o    |BSD-x86             | o  |
| freebsd   | x64        | o    | o    |BSD-x86_64          | o  |
| openbsd   | ia32       | o    | o    |BSD-x86             | x  |
| openbsd   | x64        | o    | o    |BSD-x86_64          | x  |
| others    | ia32       | x(*1)| o    | -                  | x  |
| others    | x64        | x(*1)| o    | -                  | x  |
| others    | arm        | x(*1)| o    | -                  | x  |
| others    | arm64      | x(*1)| o    | -                  | x  |
| others    | others     | x(*1)| x(*2)| -                  | x  |

- (*1) use linux-elf as a fallback configuration
- (*2) no-asm used
- (*3) currently masm (Microsoft Macro Assembler) is used but it's no
longer supported in openssl. We need to move to use nasm or yasm.

All parameters such as sources, defines, cflags and others generated
in openssl Makefile are written down into `deps/openssl/openssl.gypi`.

The header file of `deps/openssl/openssl/crypto/opensslconf.h` are
generated by `Configure` and varies on each os and arch so that we
made a new `deps/openssl/config/opensslconf.h`, where it includes each
conf file from `deps/openssl/config/archs/*/opensslconf.h` by using
pre-defined compiler macros. This procedure can be processed
automatically with `deps/openssl/config/Makefile`

Assembler support is one of the key features in openssl, but asm files
are dynamically generated with
`deps/openssl/openssl/crypto/*/asm/*.pl` by perl during
build. Furthermore, these perl scripts check the version of assembler
and generate asm files according to the supported instructions in each
compiler.

Since perl is not a build requirement in node, they all should be
generated in advance and statically stored in the repository. We
provide two sets of asm files, one is asm_latest(avx2 and addx
supported) in `deps/openssl/asm` and the other asm_obsolete(without
avx1/2 and addx) in `deps/openssl/asm_obsolute`, which depends on
supported features in assemblers. Each directory has a `Makefile`
to generate asm files with perl scripts in openssl sources.

`configure` and gyp check the version of assemblers such as gnu
as(gas), llvm and Visual Studio. `deps/openssl/openssl.gypi`
determines what asm files should be used, in which the asm_latest
needs the version of gas >= 2.23, llvm >= 3.3 or MSVS_VERSION>='2012'
(ml64 >= 12) as defined in
https://github.com/openssl/openssl/blob/OpenSSL_1_0_2-stable/crypto/sha/asm/sha512-x86_64.pl#L112-L129,
otherwise asm_obsolete are used.

The following is the detail instruction steps how to upgrade openssl
version from 1.0.2e to 1.0.2f in node.

*This needs to run Linux
enviroment.*

### 1. Replace openssl source in `deps/openssl/openssl`
Remove old openssl sources in `deps/openssl/openssl` .
Get original openssl sources from
https://www.openssl.org/source/openssl-1.0.2f.tar.gz and extract all
files into `deps/openssl/openssl` .

```sh
ohtsu@ubuntu:~/github/node$ cd deps/openssl/
ohtsu@ubuntu:~/github/node/deps/openssl$ rm -rf openssl
ohtsu@ubuntu:~/github/node/deps/openssl$ tar zxf ~/tmp/openssl-1.0.2f.tar.gz
ohtsu@ubuntu:~/github/node/deps/openssl$ mv openssl-1.0.2f openssl
ohtsu@ubuntu:~/github/node/deps/openssl$ git add --all openssl
ohtsu@ubuntu:~/github/node/deps/openssl$ git commit openssl
````
The commit message can be

>deps: upgrade openssl sources to 1.0.2f
>
>This replaces all sources of openssl-1.0.2f.tar.gz into
>deps/openssl/openssl

### 2. Replace openssl header files in `deps/openssl/openssl/include/openssl`
all header files in `deps/openssl/openssl/include/openssl/*.h` are
symbolic links in the distributed release tar.gz. They cause issues in
Windows. They are copied from the real files of symlink origin into
the include directory. During installation, they also copied into
`PREFIX/node/include` by tools/install.py.
`deps/openssl/openssl/include/openssl/opensslconf.h` and
`deps/openssl/openssl/crypto/opensslconf.h` needs to be changed so as
to refer the platform independent file of `deps/openssl/config/opensslconf.h`

The following shell script (copy_symlink.sh) is my tool for working
this procedures to invoke it in the `deps/openssl/openssl/include/openssl/`.

```sh
#!/bin/bash
for var in "$@"
do
    if [ -L $var ]; then
	origin=`readlink $var`
	rm $var
	cp $origin $var
    fi
done
rm opensslconf.h
echo '#include "../../crypto/opensslconf.h"' >  opensslconf.h
rm ../../crypto/opensslconf.h
echo '#include "../../config/opensslconf.h"' > ../../crypto/opensslconf.h
````

This step somehow gets troublesome since openssl-1.0.2f because
symlink headers are removed in tar.gz file and we have to execute
./config script to generate them. The config script also generate
unnecessary platform dependent files in the repository so that we have
to clean up them after committing header files.

```sh
ohtsu@ubuntu:~/github/node/deps/openssl$ cd openssl/
ohtsu@ubuntu:~/github/node/deps/openssl/openssl$ ./config

make[1]: Leaving directory `/home/ohtsu/github/node/deps/openssl/openssl/test'

Configured for linux-x86_64.
ohtsu@ubuntu:~/github/node/deps/openssl/openssl$ cd include/openssl/
ohtsu@ubuntu:~/github/node/deps/openssl/openssl/include/openssl$ ~/copy_symlink.sh *.h
ohtsu@ubuntu:~/github/node/deps/openssl/openssl/include/openssl$ cd ../..
ohtsu@ubuntu:~/github/node/deps/openssl/openssl$ git add include
ohtsu@ubuntu:~/github/node/deps/openssl/openssl$ git commit include/ crypto/opensslconf.h
ohtsu@ubuntu:~/github/node/deps/openssl/openssl$ git clean -f
ohtsu@ubuntu:~/github/node/deps/openssl/openssl$ git checkout Makefile Makefile.bak
````
The commit message can be

>deps: copy all openssl header files to include dir
>
>All symlink files in `deps/openssl/openssl/include/openssl/`
>are removed and replaced with real header files to avoid
>issues on Windows. Two files of opensslconf.h in crypto and
>include dir are replaced to refer config/opensslconf.h.

### 3. Apply floating patches
At the time of writing, there are four floating patches to be applied
to openssl.

- Two fixes for assembly errors on ia32 win32.

- One fix for openssl-cli built on win. Key press requirement of
  openssl-cli in win causes timeout failures of several tests.

- Adding a new `-no_rand_screen` option to openssl s_client. This
  makes test time of test-tls-server-verify be much faster.

These fixes can be applied via cherry-pick. The first three will merge without conflict.
The last commit can be landed using a recursive strategy that prefers newer changes.

```sh
git cherry-pick c66c3d9fa3f5bab0bdfe363dd947136cf8a3907f
git cherry-pick 42a8de2ac66b6953cbc731fdb0b128b8019643b2
git cherry-pick 2eb170874aa5e84e71b62caab7ac9792fd59c10f
git cherry-pick --strategy=recursive -X theirs 664a659
```

If you attempted to cherry-pick the last commit you would have the following conflict

```
# do not do this
git cherry-pick 664a6596960655e214fef25e74d3285097703e95
error: could not apply 664a659... deps: add -no_rand_screen to openssl s_client
hint: after resolving the conflicts, mark the corrected paths
hint: with 'git add <paths>' or 'git rm <paths>'
hint: and commit the result with 'git commit'
git cherry-pi
```

the conflict is in `deps/openssl/openssl/apps/app_rand.c` as below.

```sh
ohtsu@omb:openssl$ git diff
diff --cc deps/openssl/openssl/apps/app_rand.c
index 7f40bba,b6fe294..0000000
--- a/deps/openssl/openssl/apps/app_rand.c
+++ b/deps/openssl/openssl/apps/app_rand.c
@@@ -124,7 -124,16 +124,20 @@@ int app_RAND_load_file(const char *file
      char buffer[200];

  #ifdef OPENSSL_SYS_WINDOWS
  ++<<<<<<< HEAD
   +    RAND_screen();
   ++=======
   +     /*
   +      * allocate 2 to dont_warn not to use RAND_screen() via
   +      * -no_rand_screen option in s_client
   +      */
   +     if (dont_warn != 2) {
   +       BIO_printf(bio_e, "Loading 'screen' into random state -");
   +       BIO_flush(bio_e);
   +       RAND_screen();
   +       BIO_printf(bio_e, " done\n");
   +     }
   ++>>>>>>> 664a659... deps: add -no_rand_screen to openssl s_client
     #endif

      if (file == NULL)
````

We want to opt for the changes from 664a659 instead of the changes present on HEAD.
`git cherry-pick --strategy=recursive -X theirs` will do just that!

### 4. Change `opensslconf.h` so as to fit each platform.
opensslconf.h includes defines and macros which are platform
dependent. Each files can be generated via `deps/openssl/config/Makefile`
We can regenerate them and commit them if any diffs exist.

```sh
ohtsu@ubuntu:~/github/node/deps/openssl$ cd config
ohtsu@ubuntu:~/github/node/deps/openssl/config$ make clean
find archs -name opensslconf.h -exec rm "{}" \;
ohtsu@ubuntu:~/github/node/deps/openssl/config$ make
cd ../openssl; perl ./Configure no-shared no-symlinks aix-gcc > /dev/null
ohtsu@ubuntu:~/github/node/deps/openssl/config$ git diff
ohtsu@ubuntu:~/github/node/deps/openssl/config$ git commit .
````
The commit message can be

>deps: update openssl config files
>
>Regenerate config files for supported platforms with Makefile.

### 5. Update openssl.gyp and openssl.gypi
This process is needed when source files are removed, renamed and added.
It seldom happen in the minor bug fix release. Build errors would be
thrown if it happens. In case of build errors, we need to check source
files in Makefiles of its platform and change openssl.gyp or
openssl.gypi according to the changes of source files. Please contact
@shigeki if it is needed.

### 6. ASM files for openssl
We provide two sets of asm files. One is for the latest assembler
and the other is the older one. sections 6.1 and 6.2 describe the two
types of files. Section 6.3 explains the steps to update the files.
In the case of upgrading 1.0.2f there were no changes to the asm files.

Files changed between two tags can be manually inspected using:
```
https://github.com/openssl/openssl/compare/OpenSSL_1_0_2e...OpenSSL_1_0_2f#files_bucket
```
If any source files in `asm` directory were changed then please follow the rest of the
steps in this section otherwise these steps can be skipped.

### 6.1. asm files for the latest compiler
This was made in `deps/openssl/asm/Makefile`
- Updated asm files for each platforms which are required in
  openssl-1.0.2f.
- Some perl files need CC and ASM envs. Added a check if these envs
  exist. Followed asm files are to be generated with CC=gcc and
  ASM=nasm on Linux. See
  `deps/openssl/openssl/crypto/sha/asm/sha512-x86_64.pl`
- Added new 32bit targets/rules with a sse2 flag (OPENSSL_IA32_SSE2)
  to generate asm for use SSE2.
- Generating sha512 asm files in x86_64 need output filename which
  has 512. Added new rules so as not to use stdout for outputs.
- PERLASM_SCHEME of linux-armv4 is `void` as defined in openssl
  Configure. Changed its target/rule and all directories are moved
  from arm-elf-gas to arm-void-gas.
- add a new rule for armv8 asm generation

With export environments of CC=gcc and ASM=nasm, then type make
command and check if new asm files are generated.
If you don't have nasm please install it such as `apt-get install nasm`.

### 6.2. asm files for the older compiler
For older assembler, the version check of CC and ASM should be
skipped in generating asm file with perl scripts.
Copy files from `deps/openssl/asm` into
`deps/openssl/asm/asm_obsolete` and change rules to generate asm files
into this directories and remove the check of CC and ASM envs.

Without environments of CC and ASM, then type make command and check
if new asm files for older compilers are generated.

The following steps includes version check of gcc and nasm.

### 6.3 steps

```sh
ohtsu@ubuntu:~/github/node/deps/openssl/config$ cd ../asm
ohtsu@ubuntu:~/github/node/deps/openssl/asm$ gcc --version
gcc (Ubuntu 4.8.4-2ubuntu1~14.04) 4.8.4
Copyright (C) 2013 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

ohtsu@ubuntu:~/github/node/deps/openssl/asm$ nasm -v
NASM version 2.10.09 compiled on Dec 29 2013
ohtsu@ubuntu:~/github/node/deps/openssl/asm$ export CC=gcc
ohtsu@ubuntu:~/github/node/deps/openssl/asm$ export ASM=nasm
ohtsu@ubuntu:~/github/node/deps/openssl/asm$ make clean
find . -iname '*.asm' -exec rm "{}" \;
find . -iname '*.s' -exec rm "{}" \;
find . -iname '*.S' -exec rm "{}" \;
ohtsu@ubuntu:~/github/node/deps/openssl/asm$ make
ohtsu@ubuntu:~/github/node/deps/openssl/asm$ cd ../asm_obsolete/
ohtsu@ubuntu:~/github/node/deps/openssl/asm_obsolete$ unset CC
ohtsu@ubuntu:~/github/node/deps/openssl/asm_obsolete$ unset ASM
ohtsu@ubuntu:~/github/node/deps/openssl/asm_obsolete$ make clean
find . -iname '*.asm' -exec rm "{}" \;
find . -iname '*.s' -exec rm "{}" \;
find . -iname '*.S' -exec rm "{}" \;
ohtsu@ubuntu:~/github/node/deps/openssl/asm_obsolete$ make
ohtsu@ubuntu:~/github/node/deps/openssl$ git status
ohtsu@ubuntu:~/github/node/deps/openssl$ git commit asm asm_obsolete
````
The commit message can be

>deps: update openssl asm and asm_obsolete files
>
>Regenerate asm files with Makefile and CC=gcc and ASM=nasm where gcc
>version was 5.4.0 and nasm version was 2.11.08.
>
>Also asm files in asm_obsolete dir to support old compiler and
>assembler are regenerated without CC and ASM envs.