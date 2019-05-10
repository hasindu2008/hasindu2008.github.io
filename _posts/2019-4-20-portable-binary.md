---
layout: post
title: Recipe for generating portable binaries for Nanopore data processing tools
author: Hasindu Gamaarachchi
---

Compiling a software can be tedious due to numerous dependencies. This is specifically the case for Nanopore data processing tools mainly due to the HDF5 library[^1]. While an expert in system administration would find such compilation enjoyable and easy, it is typically not for users of bio-informatics tools. It is understandable that providing compiled binaries of such tools are not a perfect solution. Nevertheless I believe that it is far better than releasing a tool that will be unusable due to many users giving up at the compilation stage. Some reasons why statically linked binaries are good for bio-informatics tools are [here](http://lh3.github.io/2014/07/12/about-static-linking).


Here I describe a recipe that can be used to generate "portable binaries" for Nanopore data processing tools. The recipe is explained with reference to the tool [f5c](https://github.com/hasindu2008/f5c/]) which we are currently developing. In summary, this strategy use a combination of static linking and dynamic linking to generate a "portable binary"[^2]. However, note that this recipe is limited to C/C++ based tools. In addition, portability here means that a binary compiled for a particular operating system would work on other distributions (and versions) of the same o/s. For example, the binary for Linux on x86_64 architecture would be compatible with different distros whether Ubuntu, Debian, Fedora or Red hat. Portability here does NOT mean that the Linux version will run on Windows or the x86_64 version will run on ARM.


The key points for a successful portable binary are:

1. Avoid unnecessary dependencies as much as possible.

2. Identify libraries which causes problems when dynamically linked. Examples are :
- libraries which do not honour backward compatibility
- non mature libraries which the API frequently change
- the name/location of the shared object (.so) file installed by the package manager are different on different distributions.
Such libraries are good candidates for linking statically.

3. Identify libraries which are better to be left dynamically linked. The best example is glibc which is not recommended to be statically linked[^2]. Luckily glibc sufficiently maintains backward compatibility and can be left dynamically linked.

4. Generate the binaries on a machine (a virtual machine is sufficient) with an old Linux distribution (eg: Ubuntu 12 or 14) installed with older libraries. For instance, glibc which we decided to be left dynamically linked is NOT forward-compatible[^3].

5. Try to avoid package manager's version for libraries which are statically linked. Instead, compile those libraries yourself with minimal features that you require. For instance, statically linking hdf5 package manager's version, also require linking additional libraries such as
libsz and libaec which can be avoided if we compile HD5 ourselves without those features.



Now let's take f5c as an example. We tried our best to avoid dependencies. However, three external dependencies HDF5, HTSlib and zlib (and obviously standard libraries such as glibc, pthreads) could not be avoided. We generate the binaries for f5c on Ubuntu 14. HDF5 and HTDlib are statically linked while zlib and standard libraries are dynamically linked. Executing the command *ldd* on a release binary of f5c ("portable binary") gives the list of dynamically linked libraries:

```sh
$ldd ./f5c
linux-vdso.so.1 =>  (0x00007fffc91fb000)
libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f61550d0000)
libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f6154eb0000)
libz.so.1 => /lib/x86_64-linux-gnu/libz.so.1 (0x00007f6154c90000)
libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f61548f0000)
libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f61545e0000)
libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f61543c0000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f6153fe0000)
/lib64/ld-linux-x86-64.so.2 (0x00007f6155400000)
```

Note that  HD5F and HTSlib are statically linked and thus not seen in the *ldd* output.


To highlight why compiling external libraries ourselves with minimal features (point 5 above) let us see the additional output of *ldd* when f5c was dynamically linked with the package managers' HDF5.

```sh
$ldd ./f5c
...
libhdf5_serial.so.10 => /usr/lib/x86_64-linux-gnu/libhdf5_serial.so.10 (0x00007f1b21f30000)
libsz.so.2 => /usr/lib/x86_64-linux-gnu/libsz.so.2 (0x00007f1b20c30000)
libaec.so.0 => /usr/lib/x86_64-linux-gnu/libaec.so.0 (0x00007f1b20800000)
...
```
Observe that now in addition to the actual HDF5 library (libhdf5_serial.so) we have got two additional dependencies (libsz.so and libaec.so). Thus if one is to statically link this package manager's HDF5 version, then libsz and libaec also would have to be statically linked. Compiling HDF5 ourselves let us drop these features which we do not want.



<!-- g++ -g -Wall -O2 -std=c++11    main.o f5c.o events.o nanopolish_read_db.o model.o align.o meth.o hmm.o -lhdf5 -lz -lhts -Lhtslib/ -Lhdf5/lib -lpthread -lz -ldl   -o f5c -->

#### Note on CUDA libraries ####

CUDA runtime is not both forward and backward compatible and requires the exact version to be installed. Hence dynamically linked CUDA runtime is of no much use. Luckily CUDA runtime library has been designed to support static linking. In fact, NVIDIA recommends statically compiling the CUDA runtime library (refer the [CUDA best practices guide](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html)) and the default behaviour of the CUDA C compiler (nvcc) 5.5 or is to statically link the CUDA runtime. However, CUDA runtimes are coupled with CUDA driver versions. NVIDIA states that CUDA Driver API is backward compatible but not forward compatible (see [here](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#cuda-compatibility-and-upgrades)) and thus CUDA Runtime compiled against a particular Driver will work on later driver releases, but may not work on earlier driver versions.  As a result generating the binary should better be done with an old CUDA toolkit version. Otherwise, the users will have to install latest drivers to run this binary. For f5c we installed the CUDA 6.5 toolkit version on the Ubuntu 14 virtual machine to generate CUDA binaries.

Credits to [@danielltb](https://github.com/danielltb) for sharing valuable knowledge about static linking and compatibility.


----

[^1]: Package managers' versions of HDF5 exists, but there seem to be some inconsistencies across various distributions and a software developed on one distribution will rarely compile with no trouble on a different system. For example in one distribution the header file <hdf5.h> can be directly in the system include directory as while in some other distribution it can be <hdf5/hdf5.h> or <hdf5/serial/hdf5.h>. Meanwhile there are multiple versions of HDF5 (eg: serial, parallel and mpi etc) which can be confusing to a non-expert. Alternatively, locally compiling HDF5 is even more tedious as it is a big source code.
[^2]: If you dynamically link all libraries, the user may have to install exact version of the library which the developer used. On the other end, statically linking every thing is not ideal due to libraries such as GNU libc being non portable (see [here](http://stevehanov.ca/blog/?id=97). Thus, a hybrid static and dynamic linking strategy is the way to go.
[^3]: binaries compiled for an older glibc version will run on a system with a newer glibc (glibc is backward compatible). However, binaries compiled for newer glibc versions will not always work with an older glibc (glibc is not forward compatible).
