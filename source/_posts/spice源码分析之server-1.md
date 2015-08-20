title: spice源码分析之server(1)
date: 2015-08-19 15:36:41
tags: spice
categories: spice源码分析
---

# Spice简介
> Spice是一个开源的云计算解决方案，使客户端能显示远程虚拟主机的操作界面并且使用其设备，如键盘，鼠标，声音等。Spice给用户提供了一种如同操作本地机器一样的体验，同时尽可能把密集的CPU和GPU任务在客户端上执行。Spice能在局域网和互联网间使用，而不减少用户体验。

Spice的基本组成包括:
- Spice协议
- Spice服务器
- Spice客户端
Spice的相关组件包括:
- QXL设备
- QXL驱动
其中,Spice服务器基于libspice（一个虚拟设备接口可插拔库）。VDI提供了一个通过软件组件来发布虚拟设备接口的标准方式，使得软件组件能够与虚拟设备交互。
- 服务器使用Spice协议与客户端交互。
- 服务器通过VDI接口与VDI主机程序（如QEMU）交互。
也就是说spice服务器处于主机与客户端中间,是整个Spice的核心所在.下面我们开始从代码层面分析spice服务器
更多资料查看本人翻译的spice新手文档[Spice入门](https://www.gitbook.com/book/xhansong/spice-guidebook),以及[官方网站](http://www.spice-space.org/)

# Spice server
{% asset_img 00.png [图1] %}
图1是spice服务器的核心架构,贯穿整个源码的组织结构.
值得一提的是,spice server是作为一个库提供给qemu使用的,编译出来就是libspice,所以代码中没有main函数.
下面我们先了解一个server源码中使用到的一些核心概念,在看源码之前推荐大家先看一遍[Spice入门](https://www.gitbook.com/book/xhansong/spice-guidebook),否则理解代码中的某些核心概念会很吃力.
## 部分宏定义
- **SPICE_GNUC_DEPRECATED**,其定义是`#define SPICE_GNUC_DEPRECATED  __attribute__((__deprecated__))`表示该函数以及被弃用,在编译时会给出警告
- **SPICE_GNUC_VISIBLE**,其定义是`#define SPICE_GNUC_VISIBLE __attribute__ ((visibility ("default")))`用于控制符号的可见性,设置为对外可见.spice作为动态链接库给qemu使用,默认隐藏函数对外部的可见性,即外部文件不能调用库里面的函数,有这个声明的函数可以被外部文件调用,即为公共函数.

## 公共函数
Server的公共函数主要在两个头文件中:
- **spice.h**:与SpiceServer结构体相关的函数,是qemu调用spice的主要函数
- **red_dispatcher.h**:与QXL设备相关的函数

对server的分析,主要围绕这三个公共函数:
- **spice_server_init**:负责初始化spice_server
- **spice_server_add_interface**:给server注册VDI接口
- **spice_server_add_client**:处理qemu接收到的客户端连接消息

## VDI接口
VDI接口的定义在spice.h中,结构体内部的函数指针实现都在qemu的源码里面(ui/spice-core.c)
- SpiceCoreInterface:核心接口,用于创建,添加,取消定时和监听事件
```
    SpiceTimer *(*timer_add)(SpiceTimerFunc func, void *opaque);
    void (*timer_start)(SpiceTimer *timer, uint32_t ms);
    void (*timer_cancel)(SpiceTimer *timer);
    void (*timer_remove)(SpiceTimer *timer);
    SpiceWatch *(*watch_add)(int fd, int event_mask, SpiceWatchFunc func, void *opaque);
    void (*watch_update_mask)(SpiceWatch *watch, int event_mask);
    void (*watch_remove)(SpiceWatch *watch);
    void (*channel_event)(int event, SpiceChannelEventInfo *info);
```
- QXLInterface:QXL设备接口
```
    void (*attache_worker)(QXLInstance *qin, QXLWorker *qxl_worker);
    void (*set_compression_level)(QXLInstance *qin, int level);
    void (*set_mm_time)(QXLInstance *qin, uint32_t mm_time);
    void (*get_init_info)(QXLInstance *qin, QXLDevInitInfo *info);
    int (*get_command)(QXLInstance *qin, struct QXLCommandExt *cmd);
    int (*req_cmd_notification)(QXLInstance *qin);
    void (*release_resource)(QXLInstance *qin, struct QXLReleaseInfoExt release_info);
    int (*get_cursor_command)(QXLInstance *qin, struct QXLCommandExt *cmd);
    int (*req_cursor_notification)(QXLInstance *qin);
    void (*notify_update)(QXLInstance *qin, uint32_t update_id);
    int (*flush_resources)(QXLInstance *qin);
    void (*async_complete)(QXLInstance *qin, uint64_t cookie);
    void (*update_area_complete)(QXLInstance *qin, uint32_t surface_id,struct QXLRect *updated_rects,uint32_t num_updated_rects);
    void (*set_client_capabilities)(QXLInstance *qin,uint8_t client_present,uint8_t caps[58]);
    /* returns 1 if the interface is supported, 0 otherwise.
     * if monitors_config is NULL nothing is done except reporting the
     * return code. */
    int (*client_monitors_config)(QXLInstance *qin,VDAgentMonitorsConfig *monitors_config);
```
- SpiceCharDeviceInterface:字符型设备接口
```
    void (*state)(SpiceCharDeviceInstance *sin, int connected);
    int (*write)(SpiceCharDeviceInstance *sin, const uint8_t *buf, int len);
    int (*read)(SpiceCharDeviceInstance *sin, uint8_t *buf, int len);
    void (*event)(SpiceCharDeviceInstance *sin, uint8_t event);
```
- SpiceKbdInterface:键盘接口
```
    void (*push_scan_freg)(SpiceKbdInstance *sin, uint8_t frag);
    uint8_t (*get_leds)(SpiceKbdInstance *sin);
```
- SpiceMigrateInterface:迁移接口
```
    void (*migrate_connect_complete)(SpiceMigrateInstance *sin);
    void (*migrate_end_complete)(SpiceMigrateInstance *sin);
```
- SpiceMouseInterface:鼠标接口
```
    void (*motion)(SpiceMouseInstance *sin, int dx, int dy, int dz,uint32_t buttons_state);
    void (*buttons)(SpiceMouseInstance *sin, uint32_t buttons_state);
```
- SpiceTabletInterface:触摸板接口
```
    void (*set_logical_size)(SpiceTabletInstance* tablet, int width, int height);
    void (*position)(SpiceTabletInstance* tablet, int x, int y, uint32_t buttons_state);
    void (*wheel)(SpiceTabletInstance* tablet, int wheel_moution, uint32_t buttons_state);
    void (*buttons)(SpiceTabletInstance* tablet, uint32_t buttons_state);
```
- SpicePlaybackInterface:声音接口
- SpiceRecordInterface:录音接口

