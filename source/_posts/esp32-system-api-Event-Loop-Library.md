title: 'esp32-system-api: Event Loop Library'
date: '2024-08-07 22:39:33'
updated: '2024-08-07 22:39:35'
tags:
  - mcu
  - esp32
categories:
  - mcu
  - esp32
---
## Env

esp-idf: v5.3-stable

## 数据结构介绍

### 事件处理函数

当事件到达后，运行的函数名为事件处理函数。

事件处理函数定义如下：

```c
typedef void (*esp_event_handler_t)(void* event_handler_arg,
                                    esp_event_base_t event_base,
                                    int32_t event_id,
                                    void* event_data); 
/**< function called when an event is posted to the queue */
```

### 事件循环句柄

事件循环句柄的`内存`与`值`由`API`决定，用户并不参与，所以用户在创建事件循环句柄的时候应该创建为指针，调用`esp_event_loop_create`函数创建事件循环句柄，经该函数创建的事件循环句柄称为`用户事件循环句柄`。

事件循环句柄类型：`esp_event_loop_handle_t`
PS: 这个类型已经为一个指针。

e.g.: 
```c
esp_event_loop_handle_t loop_handle;
```

### 事件循环配置项

上面提到`事件循环句柄`由API创建，那么如果想配置`事件循环`就要用到`事件循环配置项`。

`事件循环配置项`定义如下：

```c
/// Configuration for creating event loops
typedef struct {
    int32_t queue_size;                         /**< size of the event loop queue */
    const char *task_name;                      /**< name of the event loop task; if NULL,
                                                        a dedicated task is not created for event loop*/
    UBaseType_t task_priority;                  /**< priority of the event loop task, ignored if task name is NULL */
    uint32_t task_stack_size;                   /**< stack size of the event loop task, ignored if task name is NULL */
    BaseType_t task_core_id;                    /**< core to which the event loop task is pinned to,
                                                        ignored if task name is NULL */
} esp_event_loop_args_t;
```

配置项解析如下：
- queue_size: 事件循环队列大小，指在`事件`没发送到`事件处理函数`之前，该`事件循环`最多能容纳的事件数量。
- task_name: 是否选择定义一个`线程`去读取`事件循环`的队列，并且在读取到数据后执行`事件处理函数`，如果该参数为`NULL`，则没有`专用线程`读取事件并执行`事件处理函数`，需要手动调用`esp_event_loop_run`函数去读取`队列`执行`事件处理函数`，并且下面三个成员配置项无效。
- task_priority: `事件循环专用线程`的优先级配置，可使用`uxTaskPriorityGet(NULL)`函数获取当前线程优先级附加给它。
- task_stack_size: 该`专用线程`的堆栈大小，不建议太小，会溢出，`3072`是一个普遍安全值。
- task_core_id: 该事件循环`专用线程`在哪个核心上执行，应该≥`0` && ＜ `CONFIG_FREERTOS_NUMBER_OF_CORES`，该宏在`sdkconfig`定义了最大核心数。也可以使用`tskNO_AFFINITY`宏定义不指定任何一个核心，由`FreeRTOS`指定。

## Ref
[1]https://docs.espressif.com/projects/esp-idf/zh_CN/v5.3/esp32/api-reference/system/esp_event.html
[2]https://github.com/espressif/esp-idf/tree/v5.3/examples/system/esp_event/user_event_loops
[3]https://github.com/espressif/esp-idf/tree/v5.3/examples/system/esp_event/default_event_loop