---
layout: post
title: "Building libtool modules: Python bindings"
date: 2013-09-04 10:47
comments: true
categories: [python-bindings, libtool, swig, autotools, automake]
---

One of the software packages ([xraylib](http://github.com/tschoonj/xraylib)) I am working on, consists of a C-library with bindings to a number of other languages such as Python, Fortran 2003, IDL, Java, Ruby, Lua and Perl. Apart from IDL and Fortran 2003, the source code for these bindings is generated automatically using [swig](http://www.swig.org), although with considerable language-specific modifications to the swig interface file.

The bindings source code afterwards needs to be compiled into a dynamically loadable library, which will then be loaded at runtime by the program whenever the user needs to use a function or variable from the bindings (actually from the underlying C-library, but exposed through the swig generated bindings).
As a rule, each programming language that supports such dynamically loadable libraries comes with specific instructions on how to generate the libraries from the bindings source code, often using platform independent tools.

However, these tools never seem to integrate well with autoconf and automake and lead to quite complicated Makefile.am files. A considerable more easy approach would consist of relying on libtool's functionality of generating these dynamically loadable libraries (called modules in the libtool documentation).
In this series of posts I will share my experiences on generating bindings to the aforementioned languages using libtool in a relatively clear and easy way. All the code has been tested on Linux and Mac OS X. It may also work on Windows though, provided you install a bash shell with the required tools.

In this first post I will discuss Python extension modules.

<!--more-->

## Checking for python development tools

If you want to create an autotools based project consisting of a C-library and python bindings, then one of the first things that configure will need to check for is the presence of the python interpreter and the development kit consisting of the python headers and some essential tools.
I accomplished this by adding the following lines into configure.ac

{% codeblock lang:bash %}
#configure.ac snippet
#copy paste into your own configure.ac file

#check for swig
AC_CHECK_PROGS([SWIG],[swig],["noswig"])

#present configure with a command-line option to disable the python bindings
AC_ARG_ENABLE([python],[AS_HELP_STRING([--disable-python],[build without the python bindings])],[enable_python=$enableval],[enable_python=check])
#default behavior is to install the python bindings into subfolders of $prefix
#however, this may require the user to set the PYTHONPATH environment variable
#in order to avoid this, invoke configure with the --enable-python-integration option

AC_ARG_ENABLE([python-integration],[AS_HELP_STRING([--enable-python-integration],[install the python bindings in the interpreters site-packages folder])],[enable_python_integration=$enableval],[enable_python_integration=check])

VALID_PYTHON=

if test "x$SWIG" = xnoswig && test "x$enable_python" = xyes ; then
        #don't even bother when swig is not found
        AC_MSG_ERROR([--enable-python was given as an option but swig was not found on the system])
elif test "x$SWIG" = xswig && test "x$enable_python" != xno ; then
        #verify the python installation
        AM_PATH_PYTHON(,[PYTHON_FOUND=true],[PYTHON_FOUND=false])
        if test "x$PYTHON_FOUND" = xtrue ; then
                PYTHON_CPPFLAGS=
                PYTHON_LDFLAGS=
                AX_PYTHON_DEVEL
                if test "x$PYTHON" = x ; then
                        if test "x$enable_python" = xyes ; then
                                AC_MSG_ERROR([Incomplete python development package])
                        else
                                AC_MSG_WARN([Incomplete python development package])
                        fi
                        VALID_PYTHON=no
                else
                        VALID_PYTHON=yes
                fi

        fi
fi


if test "x$VALID_PYTHON" = xyes ; then
	AC_MSG_NOTICE([Building with Python bindings])


	if test "x$enable_python_integration" = xyes ; then
        	pythondir=$PYTHON_SITE_PKG
        	pyexecdir=$PYTHON_SITE_PKG_EXEC
	fi


	AC_SUBST(PYTHONDIR,$pythondir)
	AC_SUBST(PKGPYTHONDIR,$pkgpythondir)
	AC_SUBST(PYEXECDIR,$pyexecdir)
	AC_SUBST(PKGPYEXECDIR,$pkgpyexecdir)
fi

AM_CONDITIONAL([ENABLE_PYTHON],[test x$VALID_PYTHON = xyes])
{% endcodeblock %}

As can be seen from this snippet, I depend on two m4 macros to obtain the required information: [AM_PATH_PYTHON](http://www.gnu.org/software/automake/manual/html_node/Python.html), which searches for Python interpreter and subsequently sets a number of variables that will be used at installation time, and AX_PYTHON_DEVEL (I am using a heavily modified version of the code found at [autoconf archives](http://www.gnu.org/software/autoconf-archive/ax_python_devel.html)), which will check for the required headers and perform a test compilation to make sure everything works.

## Writing the Makefile.am

The Python way of building and installing extensions relies on the [Distutils package](http://docs.python.org/3.3/distutils/setupscript.html#describing-extension-modules), and is guaranteed to work on all platforms but integrates with difficulty into autotools based installation scripts.
However, recent versions of [automake](http://www.gnu.org/software/automake/manual/html_node/Python.html) come with built-in support for installing python source files (and even byte-compile them), and comes with some recommendations on how to build your Python extensions module (possibly linked to a C-library as in our case) using libtool. The following example code is based upon these recommendations but also contains additional Makefile targets that deal with generating the Python extension module source through swig. Let's assume our original C-library is called `mytest`, and is located in the `src` subfolder of the source tree. The Python bindings will be built in the `python` subfolder.

In this example, I followed the Python recommendations with regard to naming the extension module: 
{% blockquote http://www.python.org/dev/peps/pep-0008/#package-and-module-names PEP 8 -- Style Guide for Python Code %}
When an extension module written in C or C++ has an accompanying Python module that provides a higher level (e.g. more object oriented) interface, the C/C++ module has a leading underscore.
{% endblockquote %}

In this case, the accompanying Python module will be automatically generated by swig, and will be called `mytest.py`, while the extension module will receive the libtool name `_mytest.la`. The actual library extension will be platform dependent (.so on Linux/Mac OS X, .dll on Windows).

{% codeblock lang:makefile %}

#python bindings will only be built if all buildtools are available, hence the following automake conditional
if ENABLE_PYTHON
#our python extension module
pyexec_LTLIBRARIES = _mytest.la
_mytest_la_CFLAGS = $(PYTHON_CFLAGS) -I$(top_srcdir)/include $(PYTHON_CPPFLAGS)
#link to the C-library
#probably on Windows one will need to link against the python dll as well
_mytest_la_LIBADD = ../src/mytest.la
#the source code for our extensions module
#nodist because this file will be generated by swig
nodist__mytest_la_SOURCES = mytest_wrap.c
#-module forces libtool to generate a dynamically loadable module
_mytest_la_LDFLAGS = -avoid-version -module -shared -export-dynamic


#nodist because this file will be generated by swig
nodist_python_PYTHON = mytest.py

#this line assumes that the swig interface file mytest.i is located in the src subdirectory
mytest_wrap.c: $(top_srcdir)/src/mytest.i
	$(SWIG) -I${top_srcdir}/include -includeall -o mytest_wrap.c -python ${top_srcdir}/src/mytest.i

mytest.py: mytest_wrap.c

clean-local:
	rm -rf mytest_wrap.c mytest.py mytest.pyc

endif

{% endcodeblock %}

When writing an autotools based project using these guidelines, you should have no problem compiling and installing your python bindings and the C-library it depends on, with the familiar commands:

```
./configure
make
make install
```

## The code

The full code follows: as usual it's a github gist so feel free to clone it and hack away at it.

{% gist 6441999 %}
