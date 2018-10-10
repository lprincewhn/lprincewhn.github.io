---
layout: post
title: C编译过程介绍
key: 20180910
tags: c gcc clang
modify_date: 2018-09-10
---

本文介绍各个平台下的C语言程序编译命令和编译过程。

<!--more-->

macos

```
# clang -o list list.c -I ~/Downloads/ffmpeg-4.0.2-macos64-dev/include -L ~/Downloads/ffmpeg-4.0.2-macos64-shared/bin/ -lavcodec.58 -lavformat.58 -lavutil.56
# ./list
dyld: Library not loaded: @loader_path/libavcodec.58.dylib
  Referenced from: ****/./list
  Reason: image not found
[1]    48995 abort      ./list

# export DYLD_LIBRARY_PATH=/Users/lprince/Downloads/ffmpeg-4.0.2-macos64-shared/bin
# ./list
usage: ./list OPERATION entry1 [entry2]
API example program to show how to manipulate resources accessed through AVIOContext.
OPERATIONS:
list      list content of the directory
move      rename content in directory
del       delete content in directory
```
win
Developer Command Prompt for VS2017:
```
> cl /I ..\ffmpeg-4.0.2-win32-dev\include\  list.c /link /libpath:..\ffmpeg-4.0.2-win32-dev\lib\ avcodec.lib avformat.lib avutil.lib
> set PATH=C:\Users\huaning.wang\learning\imooc\FFmpeg音视频核心技术精讲与实战\ffmpeg-4.0.2-win32-shared\bin;%PATH%
> list.exe
```
x64 Native Tools Command Prompt for VS 2017
```
> cl /I ..\ffmpeg-4.0.2-win64-dev\include\  list.c /link /libpath:..\ffmpeg-4.0.2-win64-dev\lib\ avcodec.lib avformat.lib avutil.lib
> set PATH=C:\Users\huaning.wang\learning\imooc\FFmpeg音视频核心技术精讲与实战\ffmpeg-4.0.2-win64-shared\bin;%PATH%
> list.exe
```
