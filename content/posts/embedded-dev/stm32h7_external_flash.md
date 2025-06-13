---
title: STM32H7系列在GCC环境下如何使用外部flash运行代码
keywords: stm32h7,openocd,gcc,xip
date: 2025-06-13
categories:
  - 嵌入式开发
tags:
  - stm32
  - gcc
toc: true
---

由于stm32h7一些系列的 flash 较小，经常需要外挂flash来运行程序。在keil开发环境中，网上很容易找到对应的烧录算法和bootloader，而对于gcc/clang用户来说，无法直接使用keil的烧录算法来进行下载和调试。本文将介绍 stm32 系列如何在 gcc 和 clion 开发环境下使用外部 flash 运行代码，并且能够使用 openocd 进行调试。

在gcc+openocd环境下，外部flash运行代码的原理还是一致的，因此bootloader部分可以复用keil的方案，但是烧录算法将由openocd完成。


>[!IMPORTANT]+
> STM32H7有多个系列，不同系列的qspi/octospi寄存器不一致。以`STM32H743/H750`为代表的老片子对应的外设是`Quad-SPI`，而以`STM32H7B0/H730/H723`等新片子对应的外设则是`Octo-SPI`。两个外设的配置不能通用。
> 

## H750系列QuadSPI环境配置


### QuadLine 方案

TODO

### QuadLine with dual Bank 方案
对于八线的方案 ，openocd其实已经提供了大量的例子，在`share/openocd/scripts/board`目录中我们可以找到对应的`cfg`文件。我们比对和上述的四线方案的差异做出简单修改即可。

TODO

## H7B0系列OctolSPI环境配置

H7B0
### Bootloader 配置

TODO
### QuadLine
TODO

### OctoLine
TODO
