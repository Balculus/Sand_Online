---
layout: post
title: STM32 看门狗
date: 2022-08-17
author: 来自第一世界
tags: [STM32]
comments: false
---
本篇将介绍 SMT32 的两个看门狗，独立看门狗和窗口看门狗，这两个看门狗分别承担不同功能。

# 独立看门狗（IWDG）

独立看门狗 (IWDG) 由其专用低速时钟 (LSI) 驱动，因此即便在主时钟发生故障时仍然保持 工作状态。窗口看门狗 (WWDG) 时钟由 APB1 时钟经预分频后提供，通过可配置的时间窗 口来检测应用程序非正常的过迟或过早的操作。IWDG 最适合应用于那些需要看门狗作为一个在主程序之外，能够完全独立工作，并且对时 间精度要求较低的场合。WWDG 最适合那些要求看门狗在精确计时窗口起作用的应用程序。

其基本原理在于，当通过对关键字寄存器 (IWDG_KR) 写入值 0xCCCC 启动独立看门狗时，计数器开始从复位值 0xFFF 递减计数。当计数器计数到终值 (0x000) 时会产生一个复位信号（IWDG 复位）。

需要注意的是，IWDG_PR 和 IWDG_RLR 寄存器具有写保护功能。要修改这两个寄存器的值，必须先向IWDG_KR 寄存器中写入0x5555。将其他值写入这个寄存器将会打乱操作顺序，寄存器将重新被保护。重装载操作(即写入0xAAAA)也会启动写保护功能。

## 库函数写法

写函数为：

```c
IWDG_WriteAccessCmd(IWDG_WriteAccess_Enable);
```

看门狗预分频系数设置函数：

```c
void IWDG_SetPrescaler(uint8_t IWDG_Prescaler); //设置IWDG 预分频值
```

预分频值这里可选 4~256，皆为2的指数。

设置看门狗的重装载值的函数是：

```c
void IWDG_SetReload(uint16_t Reload); //设置IWDG 重装载值
```

重载值可选 0~0x0FFF。

库函数里面重载计数值的函数:

```c
IWDG_ReloadCounter(); //按照IWDG 重装载寄存器的值重装载IWDG 计数器
```

启动看门狗函数：

```c
IWDG_Enable(); //使能IWDG
```

独立看门狗需要在重载时间之内重载。

### 设置独立看门狗步骤

1. 取消寄存器写保护
2. 设置预分频系数和重装载值
3. 重载计数值喂狗（向IWDG_KR 写入0XAAAA）
4. 启动看门狗(向IWDG_KR 写入0XCCCC)

## HEL 库写法

取消 IWDG_PR 和 IWDG_RLR 寄存器 的写保护， 这样才可以设置寄存器 IWDG_PR 和 IWDG_RLR 的值，函数为：

```c
HAL_StatusTypeDef HAL_IWDG_Init(IWDG_HandleTypeDef *hiwdg);
```

参数 hiwdg 为结构体，定义如下：

```c
typedef struct 
{ 
	IWDG_TypeDef *Instance; 
	IWDG_InitTypeDef Init; 
	HAL_LockTypeDef Lock; 
	__IO HAL_IWDG_StateTypeDef State;
} IWDG_HandleTypeDef;
```

成员变量 Instance 用来设置看门狗寄存器基地址，实际上在 HAL库中已经通过标识符定义了，这里对于独立看门狗直接设置为标识符 IWDG即可。

成员变量 Init 是一 个 IWDG_InitTypeDef 结构体类型，该结构体只有 2个成员变量，分别用来设置独立看门狗的预分频系数和重装载值，定义如下：

```c
typedef struct 
{ 
	uint32_t Prescaler; 
	uint32_t Reload; 
} IWDG_InitTypeDef;
```

成员变量 Lock 是一个锁存变量，该变量在 HAL 库中当操作配置 IWDG 之前设置为LOCK，当配置操作完成之后设置为 UNLOCK，实际上是一个操作状态标识符。
成员变量 State 也是 HAL定义的一个过程标识符，用来记录 IWDG处理状态。

一个初始化实例：

```c
IWDG_HandleTypeDef IWDG_Handler; //独立看门狗句柄
IWDG_Handler.Instance=IWDG; //独立看门狗
IWDG_Handler.Init.Prescaler=IWDG_PRESCALER_64; //设置 IWDG分频系数
IWDG_Handler.Init.Reload=500; //重装载值
HAL_IWDG_Init(&IWDG_Handler);
```

重载计数值喂狗——喂狗，向 IWDG_KR写入 0XAAAA：

```c
HAL_StatusTypeDef HAL_IWDG_Refresh(IWDG_HandleTypeDef *hiwdg);
```

启动看门狗 ，向 IWDG_KR写入 0XCCCC：

```c
HAL_StatusTypeDef HAL_IWDG_Start(IWDG_HandleTypeDef *hiwdg);
```

## 独立看门狗时间计算

独立看门狗的超时周期由分频系数和重载值决定，该时间的计算方式为：

$$
T_{out} = (rlr \times pre)/T_{clk}
$$

其中，$T_{out}$为超时时间，$pre$为预分频值，$rlr$为预装载值。

一个STM32F4xx中文参考手册中的超时计算表格：

**表格 32 kHz (LSI) 频率条件下 IWDG 超时周期的最小值/最大值**

| 预分频器 | PR[2:0] 位 | 最短超时 (ms)<br />RL[11:0]= 0x000 | 最长超时 (ms)<br />RL[11:0]= 0xFFF |
| -------- | ---------- | ---------------------------------- | ---------------------------------- |
| /4       | 0          | 0.125                              | 512                                |
| /8       | 1          | 0.25                               | 1024                               |
| /16      | 2          | 0.5                                | 2048                               |
| /32      | 3          | 1                                  | 4096                               |
| /64      | 4          | 2                                  | 8192                               |
| /128     | 5          | 4                                  | 16384                              |
| /256     | 6          | 8                                  | 32768                              |

# 窗口看门狗（WWDG）

窗口看门狗通常被用来监测，由外部干扰或不可预见的逻辑条件造成的应用程序背离正常的运行序列而产生的软件故障。除非递减计数器的值在 T6 位变成 0 前被刷新，看门狗电路在 达到预置的时间周期时，会产生一个 MCU 复位。如果在递减计数器达到窗口寄存器值之前，刷新控制寄存器中的 7 位递减计数器值，也会产生 MCU 复位。这意味着必须在限定的时间窗口内刷新计数器。

复位特性：

* 当递减计数器值小于 0x40 时复位（如果看门狗已激活）
* 在窗口之外重载递减计数器时复位（如果看门狗已激活）

![](https://raw.githubusercontent.com/Balculus/picbed/master/202302061031759.png)

如图所示，T[6:0]就是控制寄存器 （WWDG_CR）的低七位，W[6:0]即是配置寄存器（WWDG->CFR ）的低七位。T[6:0]就是窗口看门狗的计数器，而W[6:0]则是窗口看门狗的上窗口，下窗口值是固定的（0X40）。当窗口看门狗的计数器在上窗口值之外被刷新，或者低于下窗口值都会产生复位。上窗口值（W[6:0]）是由用户自己设定的，根据实际要求来设计窗口值，但是一定要确保窗口值大于0X40，否则窗口就不存在了。

## 库函数窗口看门狗函数

设置窗口值的函数，参数必须低于0x80：

```c
void WWDG_SetWindowValue(uint8_t WindowValue);
```

设置分频值函数，可选参数为 WWDG_Prescaler_1（2/4/8）：

```c
void WWDG_SetPrescaler(uint32_t WWDG_Prescaler)
```

开启看门狗函数，参数需在 0x40~0x7F之间：

```c
void WWDG_Enable(uint8_t Counter)
```

这里有个需要指出的点，在官方手册中给出了一个表格，展示了一个计算超时时间的例子，设置的是$t[0:5]$位从 0x00 到 0x3F 的计时时间，实际上是忽略了最高一位，因为第7位WDGA位始终置1，为激活位，而第六位则因为窗口限制，始终也为1。

同时还有一个函数，设置重载值的：

```c
void WWDG_SetCounter(uint8_t Counter);
```

开启看门狗中断，这里是计数值临近复位值前的一个提醒中断：

```c
WWDG_EnableIT(); //开启窗口看门狗中断
```

中断服务程序：

```c
void WWDG_IRQHandler(void)
```

中断程序中需要重设看门狗值 `WWDG_SetCounter()`以及清除提前唤醒中断位 `WWDG_ClearFlag()`。

### 设置窗口看门狗步骤

1. 使能WWDG时钟。`RCC_APB1PeriphClockCmd(RCC_APB1Periph_WWDG, ENABLE);`
2. 设置窗口值和分频系数
3. 开启中断以及分组
4. 设置计数器初始值并使能看门狗
5. 编写中断服务函数

## HEL 库窗口看门狗

使能 WWOG 时钟函数：

```c
__HAL_RCC_WWDG_CLK_ENABLE(); //使能窗口看门狗时钟
```

设置窗口值 ,分频数 和计数器初始值：

```c
HAL_StatusTypeDef HAL_WWDG_Init(WWDG_HandleTypeDef *hwwdg);
```

参数为结构体：

```c
typedef struct 
{ 
	WWDG_TypeDef *Instance; 
	WWDG_InitTypeDef Init; 
	HAL_LockTypeDef Lock; 
	__IO HAL_WWDG_StateTypeDef State; 
} WWDG_HandleTypeDef;
```

第二个参数为成员变量：

```c
typedef struct { 
	uint32_t Prescaler; //预分频系数
	uint32_t Window; //窗口值
	uint32_t Counter; //计数器值
} WWDG_InitTypeDef;
```

一个初始化实例：

```c
WWDG_HandleTypeDef WWDG_Handler; //窗口 看门狗句柄
WWDG_Handler.Instance=WWDG; //窗口看门狗
WWDG_Handler.Init.Prescaler=WWDG_PRESCALER_8; //设置分频系数为 8 
WWDG_Handler.Init.Window=0X5F; //设置窗口值 0X5F 
WWDG_Handler.Init.Counter=0x7F; //设置计数器值 0x7F 
HAL_WWDG_Init(&WWDG_Handler); //初始化 WWDG
```

HAL库中开启 WWDG的函数有两个：

```c
HAL_StatusTypeDef HAL_WWDG_Start(WWDG_HandleTypeDef *hwwdg); 
HAL_StatusTypeDef HAL_WWDG_Start_IT(WWDG_HandleTypeDef *hwwdg);
```

函数 HAL_WWDG_Start仅仅只是用来开启 WWDG，而函数 HAL_WWDG_Start_IT除了启动 WWDG，还同时启动 WWDG中断。

使能中断通道并配置优先级：

```c
HAL_NVIC_SetPriority(WWDG_IRQn,2,3); //抢占优先级 2，子优先级为 3 
HAL_NVIC_EnableIRQ(WWDG_IRQn); //使能窗口看门狗中断
```

窗口看门狗中断服务函数为：

```c
void WWDG_IRQHandler(void);
```

在函数中 HEL 库调用以下函数处理中断：

```c
void HAL_WWDG_IRQHandler(WWDG_HandleTypeDef *hwwdg);
```

并在处理中调用回调函数：

```c
void HAL_WWDG_WakeupCallback(WWDG_HandleTypeDef* hwwdg);
```

在此函数中实现喂狗以及其他功能，喂狗函数为：

```c
HAL_StatusTypeDef HAL_WWDG_Refresh(WWDG_HandleTypeDef *hwwdg, uint32_t cnt);
```

## 窗口看门狗超时计算时间

窗口看门狗超时计算时间如下：

$$
T_{wwdg} = T_{pclk1}\times 4096 \times \ 2^{WDGTB} \times (t[5:0] + 1)
$$

其中，$T_{wwdg}$为窗口看门狗超时时间，$WDGTB$为预分频系数，$T_{pclk1}$为APB1的时钟频率，$t[5:0]$为计数器低6位。
