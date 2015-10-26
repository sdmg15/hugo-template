---
layout: post
title: C++ Format packages for Linux and OS X
date: 2015-10-26
---

{{ page.title }}
================

<div class="separator" style="clear:right; float:right; margin-left:1em; margin-bottom:1em">
  <img border="0" src="/img/packaging.jpg" width="300"
       title="Packagin iz hard">
</div>

Ease of use has always been one of the main features of
[C++ Format](https://github.com/cppformat/cppformat). You only need to add two files,
`format.h` and `format.cc` to your project in order to start using the library.
And now C++ Format is available on major Linux distributions and Homebrew on OS X
so you can install it on your system with a package manager. Here is a possibly
incomplete list of supported systems:

* Debian:
  [stretch](https://packages.debian.org/source/stretch/cppformat),
  [sid](https://packages.debian.org/source/sid/cppformat)
* [Ubuntu](https://launchpad.net/ubuntu/+source/cppformat)
* [Fedora](https://admin.fedoraproject.org/pkgdb/package/cppformat/)
* [Arch Linux](https://aur.archlinux.org/packages/cppformat/) (user repository)
* [Homebrew](http://brewformulas.org/Cppformat)

Thanks go to the following people

* Anatoliy Bulukin for a Homebrew formula
* Dave Johansen for Fedora and RHEL packaging
* Eugene V. Lyubimkin for Debian packaging
* Xentec for Arch Linux packaging

Special thanks to Eugene V. Lyubimkin and Dave Johansen who suggested a number of
improvements and submitted bug reports and patches. Due to their help, the library
is now compatible with a wide range of platforms including ARM, ppc64, ppc64le and
s390x.

If you know about other distributions that support C++ Format, please let me know
in the comments section below.
