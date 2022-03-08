---
title: "Ventoy"
subtitle: ""
date: 2022-03-08T00:03:01+08:00
lastmod: 2022-03-08T00:03:01+08:00
draft: false
authors: []
description: ""

tags: []
categories: []
series: []

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: false
featuredImagePreview: "featured-image-preview.png"

toc:
  enable: false
math:
  enable: false
lightgallery: true
license: ""
---

简单来说，Ventoy 是一个制作可启动 U 盘的开源工具。

<!--more-->

有了 Ventoy 你就无需反复地格式化 U 盘，你只需要把 ISO/WIM/IMG/VHD(x)/EFI 等类型的文件直接拷贝到 U 盘里面就可以启动了，无需其他操作。

你可以一次性拷贝很多个不同类型的镜像文件，Ventoy 会在启动时显示一个菜单来供你进行选择。

{{< image src="ventoy_screen_uefi.png" width="100%" >}}

你还可以在 Ventoy 的界面中直接浏览并启动本地硬盘中的 ISO/WIM/IMG/VHD(x)/EFI 等类型的文件。

Ventoy 安装之后，同一个 U 盘可以同时支持 x86 Legacy BIOS、IA32 UEFI、x86_64 UEFI、ARM64 UEFI 和 MIPS64EL UEFI 模式，同时还不影响 U 盘的日常使用。

Ventoy 支持大部分常见类型的操作系统 （Windows/WinPE/Linux/ChromeOS/Unix/VMware/Xen ...）

目前已经测试了各类超过 820+ 个镜像文件。 支持 [distrowatch.com](https://distrowatch.com) 网站上收录的 90%+ 的操作系统。

<p>
  <a href="https://www.ventoy.net/cn/download.html" target="_blank"><img alt="React" src="https://img.shields.io/badge/-Download-175ddc?style=for-the-badge&logoColor=white" /></a>
  <a href="https://github.com/ventoy/Ventoy" target="_blank"><img alt="Github" src="https://img.shields.io/badge/-GitHub-12100E.svg?&style=for-the-badge&logo=Github&logoColor=white" /></a>
</p>
