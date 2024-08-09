title: esp32初始化结构体问题
tags:
  - esp32
  - mcu
categories:
  - mcu
  - esp32
---
在`c++`使用`espidf`进行`wifi`连接时，发现如下代码可以连接：

```c++
wifi_config_t wifi_config = {
            .sta = {
                    .ssid = "HBDT-23F",
                    .password = "hbishbis"
            }
};
```
但如下代码不可连接：

```c++
wifi_config_t wifi_config;
strcpy(wifi_config.sta.ssid, "HBDT-23F");
strcpy(wifi_config.sta.password, "hbishbis");
```

经过排查发现`espidf`对于连接阶段除了`ssid`和`password`还使用到了其他变量，所以应该清零结构体内存：

```c++
wifi_config_t wifi_config{};
```

一个很低级的问题...记录下来时刻警醒。