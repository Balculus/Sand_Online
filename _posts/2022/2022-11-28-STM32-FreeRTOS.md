---
layout: post
title: STM32 FreeRTOS 相关
date: 2022-11-28
author: 来自第一世界
tags: [STM32]
comments: false
---
总结一下 STM32 FreeRTOS 系统的一些知识。

# FreeRTOS

111

## 任务

### 任务状态

FreeRTOS 中的任务永远处于下面几个状态中的某一个:

* 运行态：当一个任务正在运行时，那么就说这个任务处于运行态，处于运行态的任务就是当前正在使用处理器的任务。如果使用的是单核处理器的话那么不管在任何时刻永远都只有一个任务处于运行态；
* 就绪态：处于就绪态的任务是那些已经准备就绪（这些任务没有被阻寒或者挂起），可以运行的任务，但是处于就绪态的任务还没有运行，因为有一个同优先级或者更高优先级的任务正在运行；
* 阻塞态：如果一个任务当前正在等待某个外部事件的话就说它处于阻塞态，比如说如果某个任务调用了函数 `vTaskDelay()` 的话就会进入阻塞态，直到延时周期完成。任务在等待队列、信号量、事件组、通知或互斥信号量的时候也会进入阻塞态。任务进入阻塞态会有一个超时时间，当超过这个超时时间任务就会退出阻塞态，即使所等待的事件还没有来临；
* 挂起态：像阻塞态一样，任务进入挂起态以后也不能被调度器调用进入运行态，但是进入挂起态的任务没有超时时间。任务进入和退出挂起态通过调用函数 `vIaskSuspend()` 和 `xTaskResume()`。

### 任务优先级

每个任务都可以分配一个从 0~(configMAX_PRIORITIES-1) 的优先级，这个参数在文件 FreeRTOSConfig.h 中有定义。

优先级数字越低表示任务的优先级越低，0 的优先级最低，空闲任务的优先级最低，为 0。FreeRTOS 调度器确保处于就绪态或运行态的高优先级的任务获取处理器使用权，换句话说就是处于就绪态的最高优先级的任务才会运行。

### 任务实现

函数 `xTaskCreate()` 或者 `xTaskCreateStatic() `用于创建任务。函数的第一个参数为 `pxTaskCode`，为此任务的任务函数，即想要实现的功能。

任务函数的模板为：

```c
void vATaskFunction(void *pvParameters)
{
	for(;;)
	{
		# code
		vTaskDelay();
	}
	vTaskDelete(NULL); # 中途退出
}
```

### 任务创建及相关函数

函数 `xTaskCreate()` 用于创建一个任务，其中所需 RAM 自动分配。新创建的任务默认就绪态的，如果当前没有比它更高优先级的任务运行那么此任务就会立即进入运行态开始运行 ，不管在任务调度器启动前还是启动后，都可以创建任务。

函数原型为：

```c
BaseType_t xTaskCreate( 
	TaskFunction_t pxTaskCode, 
	const char * const pcName, 
	const uint16_t usStackDepth, 
	void * const pvParameters, 
	UBaseType_t uxPriority, 
	TaskHandle_t * const pxCreatedTask )
```

* pxTaskCode：任务函数；
* pcName：任务名字，一般用于追踪和调试，任务名字长度不能超过 configMAX_TASK_NAME_LEN；
* usStackDepth：任务堆栈大小，注意实际申请到的堆栈是 usStackDepth的 4倍。其中空闲任务的任务堆栈大小 为 configMINIMAL_STACK_SIZE；
* pvParameters：传递给任务函数的参数 ；
* uxPriotiry：任务优先级，范围 0 ~ configMAX_PRIORITIES-1；
* pxCreatedTask：任务句柄 ，任务创建成功以后会返回此任务的任务句柄 这个句柄其实就是任务的任务堆栈。 此参数就用来保存这个任务句柄。其他 API函数可能会使用到这个句柄。

返回值：

* pdPASS: 任务创建成功；
* errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY 任务创建失败，因为堆内存不足！

函数 `xTaskCreateStatic()` 与函数 `xTaskCreate()` 功能相同，但是所需 RAM 需要由用户来提供。

```c
TaskHandle_t xTaskCreateStatic( 
	TaskFunction_t pxTaskCode, 
	const char * const pcName, 
	const uint32_t ulStackDepth, 
	void * const pvParameters, 
	UBaseType_t uxPriority, 
	StackType_t * const puxStackBuffer, 
	StaticTask_t * const pxTaskBuffer )
```

* pxTaskCode：任务函数；
* pcName：任务名字，一般用于追踪和调试，任务名字长度不能超过 configMAX_TASK_NAME_LEN；
* usStackDepth：任务堆栈大小，由于本函数是静态方法创建任务，所以任务堆栈由用户给出，一般是个数组，此参数就是这个数组的大小；
* pvParameters：传递给任务函数的参数；
* uxPriotiry：任务优先级，范围 0 ~ configMAX_PRIORITIES-1；
* puxStackBuffer：任务堆栈，一般为数组，数组类型要为 StackType_t类型；
* pxTaskBuffer：任务控制块。

返回值：

* NULL：任务创建失败；
* 其他值：任务创建成功，返回任务的任务句柄。

函数 vTaskDelete() 用于删除上述两个函数所创建的任务




111
