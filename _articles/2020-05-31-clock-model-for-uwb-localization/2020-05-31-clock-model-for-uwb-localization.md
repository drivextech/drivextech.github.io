---
# layout: page
layout: single
title: "UWB定位系统时钟模型"
title-en: "Clock Model for UWB Localization"
author: drivextech
date: 2020-05-31 15:00:00 +0800
last_modified_at: 2020-06-26 15:00:00 +0800
categories: UWB
excerpt_separator: <!--more-->
---

<!-- ## 摘要 -->

UWB定位对时钟精度有很高的要求，因而需要有一个时钟模型来尽可能的近似物理时钟，本文推导了一个考虑晶振时钟频率偏移，相位偏移，以及天线延迟的UWB时钟模型，该模型在分析双向测距误差的文章中得到具体应用。

<!--more-->

## 符号定义

***[注：若不另作说明，本文中提到的时间戳都是转化到SI单位制系统的时间]***

- $t$: 系统的全局参考时间，该参考时间可以从GPS模块获得，也可以从专用的硬件时钟源获得，还可以从某一个特定的UWB时钟Anchor获得.
- $\theta$: UWB模块的时钟相位，和UWB模块的时间戳$\bar{t}$具有固定的比例转化关系，即$\theta = K \cdot \bar{t}$. 此外，$\theta$是系统全局参考时间$t$的函数，即$\theta \triangleq \theta(t)$.
- $f$: UWB模块的时钟频率，定义为两个相邻时间点上UWB时间戳的差值与系统全局参考时间的比值，即$f = \frac{\Delta \theta}{\Delta t}$. 此外，$f$是系统全局参考时间$t$的函数，即$f \triangleq f(t)$.
- $f_c$: 系统的全局参考时钟中心频率.

## 晶振时钟简化模型

对于全局参考时钟，有

$$
\theta_c = f_c t
$$

而对于UWB模块，记其时间戳为$t_M$:

$$
\theta^M = f^M t^M
$$

定义UWB模块时钟参数相对全局参考时钟参数的误差为：

<p align="center">
  <img src="imgs/clock_model_params_error_def.svg">
</p>

从而，UWB模块时钟与全局参考时钟的误差为：

<p align="center">
  <img src="imgs/clock_time_error.svg">
</p>

以及

<p align="center">
  <img src="imgs/clock_model_without_ant_delay.svg">
</p>

## UWB系统时钟模型

对于UWB定位模块，除了UWB模块晶振时钟与系统全局时钟之间存在误差外，另一个重要的误差源是天线延迟误差，主要由PCB线路延迟，MCU处理延迟，以及天线传输延迟等组成。不同的UWB模块，或是同一模块在不同的工作环境温度下，该延迟的大小随之变化。尽管这样，相较于不同模块由于制造工艺差异导致的不同延迟大小变化，同一模块在不同的工作温度下导致的延迟大小变化要相对较小，因而在一定的工作温度范围内，可假定该模块的延迟大小近似恒定。

加入UWB模块天线延迟误差后，系统时钟模型为：

<p align="center">
  <img src="imgs/clock_model_with_ant_delay.svg">
</p>

由于UWB模块的发送延迟和接收延迟大小在同一模块的较大温度范围内近似保持恒定，则上述时钟模型可重写为：

<p align="center">
  <img src="imgs/clock_model_with_ant_delay_alt_sr.svg">
</p>
