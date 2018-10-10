---
layout: post
title: MSYS和MinGW介绍
key: 20180910
tags: msys2 mingw pacman
modify_date: 2018-09-10
---

MSYS和MinGW是Cygwin之后在Windows系统上运行Unix程序的一个很好的解决方案，很多Unix工具的windows版本实际上都是内置了MSYS或者MinGW，如git for windows, rubyinstaller devkit, 如果直接安装这些软件的话，会使得系统中存在多份MSYS和MinGW，如果你也有和我一样的Do not repeat yourself的程序猿原则（洁癖），可考虑按照本文的步骤安装MSYS2和MinGW-w64，然后在其上直接安装git和ruby这类Unix工具。

<!--more-->

- MSYS，Minimal SYStem，可以将其看成是Cygwin的简化版。其核心是一个Arch Linux系统，也提供了少量的Linux命令行工具。其升级版MSYS2提供了一个很有用的包管理器pacman，可以用pacman来安装各种运行于Linux系统中的软件，如vim，git，以及c语言编译器mingw32和mingw64。
- MinGW，Minimalist GNU for Windows，从名字可以看出，这并不是一个独立的软件，而是一个软件集合，其实他是一个用于开发原生windows应用的工具环境，提供了一系列GNU工具集，主要包含开发工具，如gcc，这些工具使用标准的win32 api进行开发。

在MSYS2的pacman仓库中，包含了多个MinGW的软件包。使用pacman查询时，可以看到这些软件包均以mingw打头。

pacman -Syu
pacman -Su
pacman -Fy
pacman -S vim
pacman -S git
pacman -S mingw-w64-x86_64-gcc

$ pacman -Ssq perl$
mingw-w64-i686-perl
mingw-w64-x86_64-perl
perl
perl-DBI
perl-Math-Int64
perl-Path-Class
perl-Probe-Perl
perl-XML-LibXML
perl-XML-Parser
perl-XML-Simple
perl-libwww
