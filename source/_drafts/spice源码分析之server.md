title: spice源码分析之server
date: 2015-07-29 16:29:33
tags: spice
categories: spice源码分析
---

# spice.h
定义spice的各种接口,包括ChannelEvent,MouseState等.
其中比较重要的有:
- SpiceServer
其实是定义在reds-private.h中的RedsState结构体.
相关函数
```
SpiceServer *spice_server_new(void);
int spice_server_init(SpiceServer *s, SpiceCoreInterface *core);
void spice_server_destroy(SpiceServer *s);
```
跟SpiceServer相关的函数的实现在reds.c中.

```
 /* new interface */
SPICE_GNUC_VISIBLE SpiceServer *spice_server_new(void)
{
    /* we can't handle multiple instances (yet) */
    spice_assert(reds == NULL);

    reds = spice_new0(RedsState, 1);//调用一个宏,来针对不同的编译环境进行优化.
    return reds;
}
```
```
SPICE_GNUC_VISIBLE int spice_server_init(SpiceServer *s, SpiceCoreInterface *core)
{
    int ret;

    spice_assert(reds == s);
    ret = do_spice_init(core);//初始化结构体变量
    if (default_renderer) {
        red_dispatcher_add_renderer(default_renderer);
    }
    return ret;
}
```
```
SPICE_GNUC_VISIBLE void spice_server_destroy(SpiceServer *s)
{
    spice_assert(reds == s);
    reds_exit();//关闭主通道
}

```
