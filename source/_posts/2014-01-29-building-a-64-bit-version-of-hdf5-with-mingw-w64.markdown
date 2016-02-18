---
layout: post
title: "Building a 64-bit version of HDF5 with MinGW-w64"
date: 2014-01-29 16:35
comments: true
categories: [HDF5, MinGW-w64, MSYS]
---

I have finally decided to try building a 64-bit version of the Windows package of [XMI-MSIM](http://github.com/tschoonj/xmimsim).
As is the case for my 32-bit builds, I am using the MinGW compiler as provided by [TDM-GCC](http://tdm-gcc.tdragon.net), although obviously this time I opted for the 64-bit variant.
Now XMI-MSIM has a considerable amount of dependencies (GTK+, GSL, HDF5, libxml2, libxslt, curl, json-glib and others), I am again forced to compile these all again from source before I can even think of compiling my actualy software. One exception here is GTK+ and its dependencies since the good people from the GTK project are offering 64-bit builds (compiled with MinGW-w64) on their [website](http://www.gtk.org/download/win64.php).

In the past, the HDF5 build has always cost me most of my time, compared to the other packages, to get it compiled on MSYS, especially the Fortran bindings part. Now since I have dropped the dependency for these Fortran bindings in my latest version, because of my past experiences, I hoped that compiling them with MinGW-w64 might somehow have become easier.

Alas...

<!--more-->

As it turns out the HDF5 people have decided to drop support for building HDF5 with MinGW with their 1.8.5 release, and are currently only supporting builds on Windows with either cygwin (with the configure script) or Visual Studio (with CMake), none of which was suitable for me.
I downloaded the latest version 1.8.12 from the [HDF5 website](http://www.hdfgroup.org/HDF5/release/obtainsrc.html), unpacked the tarball and ran:

```
$ ./configure --host=x86_64-w64-mingw32 --build=x86_64-w64-mingw32 --disable-hl --prefix=$HOME
```

The two first options ensure that the configure script realizes that I am using a 64-bit MinGW compiler, which may be necessary for some tests to pass (at least with other packages I compiled this was necessary).
The third option disable the high-level bindings, while the last option makes sure that they files will be installed in subfolders of my home directory.

Running make resulted in the compilation process starting but it halted after some time with errors that were related to some preprocessor macros that were not defined. A quick google search yielded [this](http://mail.lists.hdfgroup.org/pipermail/hdf-forum_lists.hdfgroup.org/2012-April/005699.html). After implementing this fix I resumed compilation but ran again into trouble. This time google was not helpful but I found the fix myself. 

The solution to both problems was to append the following lines to the src/H5pubconf.h within your _builddirectory_:

``` c 
//fixes first problem
#ifndef H5_HAVE_WIN32_API
#ifdef WIN32 /* defined for all windows systems */
#define H5_HAVE_WIN32_API 1
#endif
#endif

#ifndef H5_HAVE_MINGW
#ifdef __MINGW32__ /*defined for all MinGW compilers */
#define H5_HAVE_MINGW 1
#endif
#endif

//fixes second problem
#define H5_BUILT_AS_DYNAMIC_LIB 1
```

Afterwards everything builds fine, and even make check went pretty well except for one error, which seemed harmless.

There's a good chance that future HDF5 versions will either fix this by including this patch (although I wouldn't count on it) or will break building HDF5 with MinGW-w64 once again in new and fascinating ways (I put my money on this). Since I will be stuck with HDF5 for a while longer, don't be surprised if this post gets updated...



