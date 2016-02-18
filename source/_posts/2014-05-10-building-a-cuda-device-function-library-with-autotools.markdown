---
layout: post
title: "Building a CUDA device function library with autotools"
date: 2014-05-10 15:08:11 +0200
comments: true
categories: [CUDA, automake, autotools]
---


I have recently started a new development branch in my [xraylib](http://github.com/tschoonj/xraylib) project, where I will gradually be adding support for [nVidia's CUDA platform](https://developer.nvidia.com/about-cuda). Essentially the goal is to create a library containing CUDA device function equivalents of most functions that are currently offered by xraylib. Since there is not really much point to include functions that deal with strings, I will be leaving those out.

Compiling such a library has been supported by nVidia since CUDA version 5.5, but for some reason I never got their examples working. However, their recent 6.0 release seems to have fixed things (at least for me). In this post I will share some of the tricks I had to come up with to generate such a library.

<!--more-->

## Adding configure CUDA support

The configure script that I am using for xraylib allows the user to select which bindings to build. The script would then proceed with checking if all the necessary building dependencies are available on the system. I accomplished this using the `AX_CHECK_CUDA` m4 macro I found at http://wili.cc/blog/cuda-m4.html, of course with some modifications to suit my particular needs. Check the gist later in this post for the code.

Important here are the `NVCC`, `CUDA_CFLAGS`, `CUDA_LDFLAGS` and `CUDA_LIBDIR` variables that get defined and will be used in the Makefile.am's later on.

## Building the library

Compiling CUDA code into a library and then linking this library into an executable is not a trivial task, which becomes clear after reading through the [CUDA documentation on separate compilation](http://docs.nvidia.com/cuda/cuda-compiler-driver-nvcc/index.html#using-separate-compilation-in-cuda).
First of all, we can only create static libraries.
This is not nearly as bad is at sounds: it will make the Makefile.am considerably simpler since we will be able to avoid using libtool this way. Libtool would really make things difficult here since we have to use the nvcc command with its non-standard options and have to define custom compilation rules for our source code files, at least assuming you follow the convention of giving your CUDA source files the .cu extension. In my case I ended up with this code:

{% codeblock lang:makefile %}
xraylibincludedir = ${includedir}/xraylib

if ENABLE_CUDA
lib_LIBRARIES=libxrlcuda.a
xraylibinclude_HEADERS = xraylib-cuda.h
libxrlcuda_a_SOURCES = xraylib-cuda.cu xraylib-cuda.h \
		       xraylib-cuda-private.h \
		       atomiclevelwidth.cu \
		       atomicweight.cu \
		       auger_trans.cu \
		       comptonprofiles.cu \
		       coskron.cu \
		       cross_sections.cu \
		       densities.cu \
		       edges.cu \
		       fi.cu \
		       fii.cu \
		       fluor_yield.cu \
		       jump.cu \
		       radrate.cu \
		       scattering.cu \
		       kissel_pe.cu \
		       xrf_cross_sections_aux.cu \
		       cs_line.cu \
		       splint.cu \
		       fluor_lines.cu \
		       polarized.cu \
		       cs_barns.cu
libxrlcuda_a_CFLAGS = $(CUDA_CFLAGS) -I$(top_srcdir)/src -I$(top_srcdir)/include
libxrlcuda_a_AR = $(NVCC) -arch=sm_20 -lib -o


.cu.o: xraylib-cuda.h xraylib-cuda-private.h
	$(NVCC) $(libxrlcuda_a_CFLAGS) -arch=sm_20 -dc -o $@ $<

endif
{% endcodeblock %}

Note how I disabled libtool here: I included the `lib_LIBRARIES` variable instead of `lib_LTLIBRARIES`. This two letter difference will ensure that libtool is not being used!
Two other things really set this file apart from typical Makefile.am's:

1. `libxrlcuda_a_AR = $(NVCC) -arch=sm_20 -lib -o`: I am overriding the default archiver here to ensure that both the device and the host functions end up in the static library. The regular archiver (`ar`) would have ignored the device functions.
2. The rule for building the CUDA source code: automake has no idea what to do with the .cu files. Enters the suffix rule:
``` makefile
.cu.o: xraylib-cuda.h xraylib-cuda-private.h
	$(NVCC) $(libxrlcuda_a_CFLAGS) -arch=sm_20 -dc -o $@ $<
``` 

Here, the `-dc` option ensures that both device and host code will get compiled into the objects.

## Linking an executable with the library

xraylib contains an example folder with a Makefile.am that will handle the compilation and running of the examples whenever `make check` is invoked after successful building of the core library and its bindings. The example contains both device (CUDA kernels that use the device functions) as well as host code to launch the kernels and handle the I/O with the GPU.
Here are the relevant sections of this file that deal with the cuda bindings example:

{% codeblock lang:makefile %}
if ENABLE_CUDA
  check_PROGRAMS += xrlexample11
endif

if ENABLE_CUDA
  xrlexample11_SOURCES = xrlexample11.cu
  xrlexample11_LDADD = xrlcudalink.o ../src/libxrl.la  $(CUDA_LDFLAGS) ../cuda/libxrlcuda.a -lcudart
  xrlexample11_CFLAGS = -I${top_srcdir}/include -I${top_builddir}/include -I${top_srcdir}/cuda -I${top_srcdir}/src
  TESTS_ENVIRONMENT = DYLD_LIBRARY_PATH="$(CUDA_LIBDIR)" LD_LIBRARY_PATH="$(CUDA_LIBDIR)"
endif

.cu.o: ../cuda/libxrlcuda.a
	$(NVCC) $(xrlexample11_CFLAGS) -arch=sm_20 -dc -o $@ $<

xrlcudalink.o: xrlexample11.o ../cuda/libxrlcuda.a
	$(NVCC) -arch=sm_20 -dlink xrlexample11.o ../cuda/libxrlcuda.a -o xrlcudalink.o
{% endcodeblock %}

It's quite similar to the Makefile.am for the static library: the exact same build rule is required to compile the CUDA source code. Interesting is the `xrlcudalink.o` build rule which turns out to be essential to link the device code in the example with the device code in the library. The resulting file after linking needs to be added to the `LDADD` command to ensure it ends up in the eventual executable. I am not overriding the linker here, so the default one will be used which is just fine.

I have tested this on Mac OS X Mavericks and Linux Centos 6.5, both with CUDA 6.0 installed. Soon I will also give this a try on Windows 7, so expect an update soon.

{% gist a3b1c346d1cef1cf2a23 %}


