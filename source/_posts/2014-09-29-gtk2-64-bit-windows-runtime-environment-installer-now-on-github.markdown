---
layout: post
title: "Gtk2 64-bit Windows Runtime Environment Installer: now on GitHub!"
date: 2014-09-29 12:33:58 +0100
comments: true
categories: [Gtk2, MinGW-w64, runtime installer, Windows 64-bit, github]
---

As predicted in my [first post](http://tschoonj.github.io/blog/2014/02/08/gtk2-64-bit-windows-runtime-environment-installer/) on the Gtk2 64-bit Windows Runtime Environment Installer, I have indeed ventured into compiling Gtk (2.24.24) and all its dependencies, mainly because I was getting increasingly unhappy with the old Gtk 2.22.1 based bundle that [is being distributed](http://www.gtk.org/download/win64.php) by the Gtk project. It was in fact this very bundle that I was using to generate the installer I announced in my first post on this subject.

So, after spending about 10 hours of compiling (and recompiling a couple of times when I didn't get the configure options right) of more than a dozen software packages, I ended up with a fully working (at least until now...) collection of DLLs. I updated the code from my initial installer to include the new DLLs and uploaded the new installer [here](http://lvserver.ugent.be/gtk-win64/). I also uploaded a zip-file (sdk) containing all executables, DLLs, linking libraries (.dll.a) and headers, that can be used by anyone not willing to replicate my compilation effort. In fact I recommend that people using this installer to distribute the Gtk runtime along with their own program, to compile and link against this sdk, in order to avoid any link issues at runtime...

You may notice that I did not use the exact same dependencies in my compilation stack as those offered by the official Gtk bundle. I followed the [Hexchat flowchart](http://hexchat.github.io/gtk-win32/) and ended up with additional dependencies of libffi and harfbuzz. libexpat was replaced by libxml2. This explains why the new installer is considerably larger than the previous one.

Last but not least, I forked the original repository of the installer from [its sourceforge](http://sourceforge.net/projects/gtk-win/) repository and created my own personal copy on [GitHub](https://github.com/tschoonj/GTK-for-Windows-Runtime-Environment-Installer). Check the README file for more information.
