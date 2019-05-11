---
layout: post
title: Recipe for generating portable binaries for ONT tools
author: Hasindu Gamaarachchi
---

Compiling a software can sometimes be a nightmare due to numerous dependencies. This is specifically the case for bio-informatics tools that utilise signal level data from
Oxford Nanopore (ONT) sequencers. According to my experience, the major cause behind compilation troubles in ONT tools is the  Hierarchical Data Format 5 (HDF5) library[^1]. While a system admin may enjoy tedious compilations, it is not the case for users of bio-informatics tools. What if the tool developers release pre-compiled binaries? Some would object this as it is not a perfect solution. Nevertheless, I believe that it is far better than releasing an unusable tool due to users giving up at the compilation stage. Further, pre-compiled binaries are less bulky compared to docker images. Generating a "portable binary" that runs on numerous Linux distributions/version is tricky, but possible with some additional work from the developer's side.

Explained below is a recipe (or probably some key points) to generate "portable binaries" for ONT tools.  In summary, this strategy uses a combination of static linking and dynamic linking to generate a "portable binary". Dynamically linking all libraries means that the user would have to install exact version of the library as the developer. On the other end, statically linking everything is also not ideal[^2]. Thus, a hybrid static and dynamic linking strategy is the way to go. However, this recipe is limited to C/C++ based tools. In addition, portability here means that a binary compiled for a particular operating system would work on other distributions (and versions) of the same operating system. For example, the binary for Linux on x86_64 architecture would run despite the distribution (whether Ubuntu, Debian, Fedora or Red hat) or the version. Portability here does NOT mean that the Linux version will run on Windows or that x86_64 version will run on ARM.

### Key points ###

The key points for a successful portable binary are:

1. Avoid unnecessary dependencies as much as possible.

2. Identify libraries which causes problems when dynamically linked. Such libraries are good candidates for linking statically. Examples are:
 - libraries which do not honour backward compatibility
 - non-mature libraries which the API frequently change
 - the name/location of the shared object (.so) file installed by the package manager is different on different distributions.


3. Identify libraries which are better to be left dynamically linked. The best example is glibc which is not recommended to be statically linked[^2]. Luckily glibc sufficiently maintains backward compatibility and can be left dynamically linked.

4. Generate the binaries on a machine (a virtual machine is sufficient) with an old Linux distribution (eg: Ubuntu 14 or even better if Ubuntu 12) installed with older libraries. For instance, glibc which we decided to be left dynamically linked is NOT forward-compatible[^3].

5. Try to avoid package manager's version for libraries when statically linking. Instead, compile those libraries yourself with minimal features that you require for your tool. For instance, statically linking HDF5 package manager's version, also require linking additional libraries such as
*libsz* (a lossless compression library for scientific data) and *libaec* (Adaptive Entropy Coding library). Those can be avoided if we compile HDF5 ourselves without those features (if your tool does not require those additional features).


### A case study with F5C ###

Now let's go through the above points with reference to [f5c](https://github.com/hasindu2008/f5c/]), a tool which we are currently developing that utilises ONT raw data.

1. We tried our best to avoid dependencies. However, three external dependencies HDF5, HTSlib (high-throughput sequencing), zlib (data compression library) and obviously standard libraries (such as glibc, pthreads) could not be avoided. We generate the binaries for f5c on Ubuntu 14.


2. HDF5 and HTSlib are statically linked. The location of the .so file of HDF5 is not consistent across distributions and even different versions in the same distribution. HTSlib that comes with the package manager is an older version and f5c required a newer version to support long reads. Thus, we statically link HDF5 and HTSlib. For the CUDA supported version of f5c, we statically link the CUDA runtime library as well, which is explained later.

3. zlib and other standard libraries are dynamically linked. Executing the command *ldd* on a release binary of f5c ("portable binary") gives the list of dynamically linked libraries shown below. Note that  HD5F and HTSlib were statically linked and thus not seen in the *ldd* output.


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


4. We compile on a virtual machine with Ubuntu 14.


5. To highlight why compiling external libraries ourselves with minimal features, let us see the additional output of *ldd* when f5c was dynamically linked with the package managers' HDF5 (see below). Observe that now in addition to the actual HDF5 library (*libhdf5_serial.so*) we have got two additional dependencies (*libsz.so* and *libaec.so*). Thus, if one is to statically link this package manager's HDF5 version, then *libsz* and *libaec* also would have to be statically linked. Compiling HDF5 ourselves let us drop these features which we do not want.

```sh
$ldd ./f5c
...
libhdf5_serial.so.10 => /usr/lib/x86_64-linux-gnu/libhdf5_serial.so.10 (0x00007f1b21f30000)
libsz.so.2 => /usr/lib/x86_64-linux-gnu/libsz.so.2 (0x00007f1b20c30000)
libaec.so.0 => /usr/lib/x86_64-linux-gnu/libaec.so.0 (0x00007f1b20800000)
...
```






#### Note on CUDA libraries ####

CUDA runtime is not both forward and backward compatible and requires the exact version to be installed. Hence dynamically linked CUDA runtime is of no much use. Luckily CUDA runtime library has been designed to support static linking. In fact, NVIDIA recommends statically compiling the CUDA runtime library (refer the [CUDA best practices guide](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html)) and the default behaviour of the CUDA C compiler (nvcc) 5.5 or is to statically link the CUDA runtime. However, CUDA runtimes are coupled with CUDA driver versions. NVIDIA states that CUDA Driver API is backward compatible but not forward compatible (see [here](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#cuda-compatibility-and-upgrades)) and thus CUDA Runtime compiled against a particular Driver will work on later driver releases, but may not work on earlier driver versions.  As a result, generating the binary should better be done with an old CUDA toolkit version. Otherwise, the users will have to install latest drivers to run this binary. For f5c we installed the CUDA 6.5 toolkit version on the Ubuntu 14 virtual machine to generate CUDA binaries.


#### Example commands ####

Assume we have HDF5 and HTSlib locally compiled and the static libraries (libhdf5.a and libhts.a) are located in ./build/lib/. These libraries are statically linked as :

```sh
<gcc/g++> [options] <object1.o> <object2.o> <...> build/lib/libhdf5.a -ldl build/lib/libhts.a  -lpthread -lz  -o binary
```

To statically link the CUDA runtime when using gcc or g++:
```
<gcc/g++> [options]   <object1.o> <object2.o> <...> build/lib/libhdf5.a build/lib/libhts.a  -L/usr/local/cuda/lib64 -lcudart_static -lpthread -lz  -lrt -ldl -o binary
```
Alternatively if CUDA toolkit 5.5 higher NVIDIA C compiler *nvcc* links the CUDA runtime statically by default:
```sh
nvcc [options] <object1.o> <object2.o> <...> build/lib/libhdf5.a build/lib/libhts.a  -lpthread -lz  -lrt -ldl -o binary
```

After generating the binary issue the *ldd* command to verify if the intended ones are statically linked. The output of *ldd* lists the dynamically linked libraries and the statically linked libraries should NOT appear in this output.
```
ldd ./binary
```

----

Credits to [@danielltb](https://github.com/danielltb) for sharing valuable knowledge about static linking and compatibility.


----

[^1]: Currently, the raw signal data from the ONT sequencers are stored in HDF5 file format. Possibly due to the complexity of HD5, there are no alternate library  implementations than the official library from the HDF Group. Compiling HDF5 library takes time. Luckily, package managers' versions of HDF5 library exists, but there seem to be some inconsistencies across various distributions. Thus, a software developed on one Linux distribution will rarely compile without any trouble on a different system. For example, the header file <hdf5.h> resides directly *include* directory on certain systems, while in some other system it can be <hdf5/hdf5.h> or <hdf5/serial/hdf5.h>.
[^2]: Static linking is not ideal due to libraries such as GNU libc being non-portable (see [here](http://stevehanov.ca/blog/?id=97). But, may reasons why statically linked binaries are good for bio-informatics tools are [here](http://lh3.github.io/2014/07/12/about-static-linking).
[^3]: binaries compiled for an older glibc version will run on a system with a newer glibc (glibc is backward compatible). However, binaries compiled for newer glibc versions will not always work with an older glibc (glibc is not forward compatible).
