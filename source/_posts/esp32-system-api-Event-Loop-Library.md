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

当事件到达后，运行的函数名为`事件处理函数`。

`事件处理函数`要在`事件循环`创建完成之后调用注册函数注册进入`事件循环`。

`事件处理函数`定义如下：

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

## EVENT_BASE与EVENT_ID

每个`事件循环`都应该有`事件处理函数`，否则`事件循环`毫无意义。

在`事件处理函数`中区别`事件`的唯一方式就是通过`EVENT_BASE`与`EVENT_ID`。

这两个类似于Linux的major与minor。

如`EVENT_BASE`可以是`WIFI_EVENT`，`EVENT_ID`可以是`CONNECTED`、`GOT_IP`等。

`EVENT_BASE`定义方式如下：

```c
ESP_EVENT_DECLARE_BASE(MY_EVENT_BASE);
ESP_EVENT_DEFINE_BASE(MY_EVENT_BASE);
```

`EVENT_ID`推荐通过`enum`定义：

```c
enum {
    MY_EVENT_ID1,
    MY_EVENT_ID2,
    MY_EVENT_ID3,
};
```

## API介绍

该章节涵盖了常用API以及简介，欲知更详细，请见[1]

- esp_event_loop_create:  该函数创建了`事件循环`，两个入参一个是`事件循环句柄`，一个是`事件循环参数`。
- esp_event_loop_create_default: 该函数创建了默认的事件循环，由于一些事件的推送不能由用户提交到队列，如wifi连接事件，获取ip事件等，所以便有了默认的事件循环，具体见[4]
- esp_event_loop_delete: 删除事件循环，入参是`事件循环句柄`。
- esp_event_loop_delete_default: 删除默认事件循环
- esp_event_handler_register_with: 注册`事件处理函数`到某个`事件循环`，需要提供该事件处理函数所处理的`EVENT_BASE`和`EVENT_ID`，一个`事件处理函数`可用不同的`EVENT`注册多次，也可以使用`ESP_EVENT_ANY_ID`去处理整个`EVENT_BASE`下的`ID`
- esp_event_handler_unregister_with: 注销某个`事件处理函数`
- esp_event_handler_register: 为`默认事件循环`注册`事件处理函数`，除了不需要提供`事件循环句柄`以外，与`esp_event_handler_register`无异。
- esp_event_handler_unregister: 为`默认事件循环`注销`事件处理函数`，除了不需要提供`事件循环句柄`以外，与`esp_event_handler_register`无异。
- esp_event_post_to: 向指定的`事件循环`发送事件。
- esp_event_post: 向默认的`事件循环`发送事件
- esp_event_loop_run: 当在事件循环参数内没有配置`task_name`时，就不用有`专有线程`去读取`队列`并且执行`事件处理函数`，此时需要调用`esp_event_loop_run`手动读取并调用`事件处理函数`，该函数两个入参分别为`事件循环句柄`和`运行tick`

## Ref
[1]https://docs.espressif.com/projects/esp-idf/zh_CN/v5.3/esp32/api-reference/system/esp_event.html
[2]https://github.com/espressif/esp-idf/tree/v5.3/examples/system/esp_event/user_event_loops
[3]https://github.com/espressif/esp-idf/tree/v5.3/examples/system/esp_event/default_event_loop
[4]https://docs.espressif.com/projects/esp-idf/zh_CN/v5.3/esp32/api-reference/system/esp_event.html#esp-event-default-loops