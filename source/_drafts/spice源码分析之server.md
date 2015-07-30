title: spice源码分析之server
date: 2015-07-29 16:29:33
tags: spice
categories: spice源码分析
---

# spice.h

**部分宏定义**
- **SPICE_GNUC_DEPRECATED**,其定义是`#define SPICE_GNUC_DEPRECATED  __attribute__((__deprecated__))`表示该函数以及被弃用,在编译时会给出警告
- **SPICE_GNUC_VISIBLE**,其定义是`#define SPICE_GNUC_VISIBLE __attribute__ ((visibility ("default")))`用于控制符号的可见性,设置为对外可见.spice动态链接库默认隐藏对外部的可见性,即外部文件不能调用库里面的函数,有这个声明的函数可以被外部文件调用,即为公共函数.

**定义spice的各种接口**:
- QXLInterface
- SpiceCharDeviceInterface
- SpiceCoreInterface
- SpiceKbdInterface
- SpiceMigrateInterface
- SpiceMouseInterface
- SpicePlaybackInterface
- SpiceRecordInterface
- SpiceTabletInterface

**重要的结构体有**:
- SpiceServer
其实是定义在reds-private.h中的RedsState结构体.
跟SpiceServer相关的函数的实现在reds.c中.
```
SpiceServer *spice_server_new(void);
int spice_server_init(SpiceServer *s, SpiceCoreInterface *core);
void spice_server_destroy(SpiceServer *s);
```

