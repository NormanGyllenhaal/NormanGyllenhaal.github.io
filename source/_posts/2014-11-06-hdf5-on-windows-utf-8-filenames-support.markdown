---
layout: post
title: "HDF5 on Windows: UTF-8 filenames support"
date: 2014-11-06 18:10:35 +0000
comments: true
categories: [HDF5, MinGW-w64, Windows, Unicode]
---

In a previous [post](http://tschoonj.github.io/blog/2014/01/29/building-a-64-bit-version-of-hdf5-with-mingw-w64/), I have detailed my efforts in getting HDF5 compiled on Windows 64-bit using the MinGW-w64 compiler, which is currently (still) unsupported by the HDF5 developers. Now, I am reporting on yet another problem I encountered with HDF5 on Windows, that is blatantly being ignored by the developers: UTF-8 filename support.

This was brought to my attention by a Japanese user of my software package [XMI-MSIM](http://github.com/tschoonj/xmimsim), who ran into trouble running the program when logged in with a username consisting of Japanese characters. The problem could be traced back to an HDF5 file that has to be opened, which is located in a subdirectory of the user's homedirectory. After investigating the HDF5 code, it became quite clear that this issue is caused by the internal use of the [`_open` function](http://msdn.microsoft.com/en-us/library/z0kc8e3z.aspx), which is known not to support the Unicode UTF-8 characterset, necessary to represent the Japanese characters. The HDF5 website confirmed this issue with the following statement:

<!--more-->

{% blockquote HDF5 documentation http://www.hdfgroup.org/HDF5/doc/Advanced/UsingUnicode/ %}
Linux, Unix, and similar systems generally handle UTF-8 encodings in correct and predictable ways. There is an apparent consensus in the Linux community that "UTF-8 is just the right way to go."
Mac OS systems generally handle UTF-8 encodings correctly.

Windows systems use a different Unicode encoding, UCS-2 (discussed in this UTF-16 article) at the system level. Within an HDF5 file and application on a Windows system, UTF-8 encoding should work correctly and as expected. Problems may arise, however, when that UTF-8 encoding is exposed directly to the Windows system. For example:

File open and close calls on files with UTF-8 encoded names are likely to fail as the HDF5 open and close operations interact directly with the Windows file system interface.
Anytime an HDF5 command-line utility (h5ls or h5dump, for example) emits text output, the Windows system must interpret the character encodings. If that output is UTF-8 encoded, Windows will correctly interpret only those characters in the ASCII subset of UTF-8.
{% endblockquote %}

So basically I was stuck here since there is no way to convert a filename in UTF-8 encoding to the UCS encoding that was expected by `_open`. [A solution](http://stackoverflow.com/questions/23285759/fopen-file-name-with-utf8-string-in-windows) which would have converted the filename to the corresponding short form (8 character filename with 3 character extension) using [`GetShortPathName`](http://msdn.microsoft.com/en-us/library/windows/desktop/aa364989%28v=vs.85%29.aspx) seemed like a very ugly hack and decided not to pursue it. Instead I opted to hack the HDF5 code myself and replace all instances of `_open` with its wide char counterparts, that could be fed the UTF-8 filenames after converting them with [`MultiByteToWideChar`](http://msdn.microsoft.com/en-us/library/windows/desktop/dd319072.aspx).

Fortunately, it turned out that I was not the first one who ran into this situation and found a better solution on the [HDF5 forums](http://mail.lists.hdfgroup.org/pipermail/hdf-forum_lists.hdfgroup.org/2014-August/007988.html). Essentially the solution consists of redefining the `HDopen` macro from `_open` to a function of our own that converts our filename in UTF-8 encoding to the corresponding wide char representation and feed it to `_wopen`. Since we are compiling with MinGW-w64, some other modifications were necessary (the original solution relied on CMake): the src/Makefile.am file was modified in order to compile also the H5FDwindows.c and H5FDwindows.h, the latter being added to the public headers. After running `autoreconf -i -f` (I had to downgrade the required autoconf version in configure.ac for this to work, but this didn't create any problems), and running the configure script, I modified the src/H5pubconf.h file according to my [earlier post](http://tschoonj.github.io/blog/2014/01/29/building-a-64-bit-version-of-hdf5-with-mingw-w64/), but had to add one more line:

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
//additional line necessary for UTF-8 build to succeed
#define H5_HAVE_WINDOWS 1
#endif
#endif

//fixes second problem
#define H5_BUILT_AS_DYNAMIC_LIB 1
```

Keep in mind that this file is recreated after running configure, so these modifications will have to be repeated!!!

This solution is not pretty, and it will have to be repeated every time you compile a new version of HDF5. Ideally, the HDF5 developers would implement a solution based on this hack, which would bring it in line with most other open-source projects that assume UTF-8 filenames on all platforms. I doubt this will happen anytime soon though, since the original solution posted on the HDF5 forums has not been commented on by any of the developers...
