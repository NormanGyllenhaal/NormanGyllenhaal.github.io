---
layout: post
title: "Gtk2 64-bit Windows Runtime Environment Installer"
date: 2014-02-08 12:47
comments: true
categories: [Gtk2, MinGW-w64, runtime installer, Windows 64-bit]
---

**Update: this post has been largely superseded by http://tschoonj.github.io/blog/2014/09/29/gtk2-64-bit-windows-runtime-environment-installer-now-on-github **


As I already mentioned in my [last post](http://tschoonj.github.io/blog/2014/01/29/building-a-64-bit-version-of-hdf5-with-mingw-w64/), I have been busy compiling software with MinGW-w64 for the Windows 64-bit platform.f. One of the packages I rely on is [Gtk+](http://www.gtk.org) (version 2 for now), and its many dependencies. Fortunately however, the [Gtk+ website](http://www.gtk.org/download/win64.php) offers binaries for this architecture, so I didn't have to bother compiling them myself.

In the end I also managed to build all the other dependencies I needed for [XMI-MSIM](http://github.com/tschoonj/xmimsim), so the last step for me was to update my [NSIS](http://nsis.sourceforge.net/Main_Page) installer for Windows 64-bit machines. It didn't take long for me to realize that one of the crucial components I install is the [Gtk Windows Runtime Environment Installer](http://gtk-win.sourceforge.net/home/), which really facilitates installing Gtk+ based applications. Unfortunately, Alexander Shaduri, the developer only distributes a 32-bit version of this package, and he has assured me through some emailing that he had no plans to write a 64-bit version due to [the experimental nature](http://www.gtk.org/download/win64.php) of the packages that are produced by the Gtk+ people.

This being said, I still needed such an installer so I wrote one based on his NSIS script, using the official 64-bit Gtk+ packages for Windows 64-bit. The only major change I introduced was dropping support for the compatibility DLLs, which were redundant anyway. I am sharing the modified NSIS script as Github Gist so start hacking away at it. This script has some dependencies, apart from the DLLs, which are necessary to build the installer. Since I didn't change them at all, use the files that the original developer is sharing on [sourceforge](http://sourceforge.net/p/gtk-win/code/HEAD/tree/).

The installer itself can be obtained [here](http://lvserver.ugent.be/gtk-win64/). If you would like to see an example on how to use this installer from within your own NSIS script, have a look at e.g. [my XMI-MSIM Windows 64-bit installer](https://github.com/tschoonj/xmimsim/blob/master/nsis/xmimsim-win64.nsi.in).

<!-- more -->

The observant reader may have noticed that the version of Gtk+ that is included is 2.22.1, which is relatively old. In the future I may become sufficiently bold and adventurous, and end up compiling the latest Gtk+ 2.24.x version myself and releasing an updated installer. If this happens, I will update this blogpost...


{% gist 8882727 %}


