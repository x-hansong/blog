title: spice源码分析之server
date: 2015-07-29 16:29:33
tags: spice
categories: spice源码分析
---

_注:下面的分析省略了所有关于迁移功能的函数与变量.只注重分析关键性函数和变量._

# spice.h
## 公开的函数接口
**部分宏定义**
- **SPICE_GNUC_DEPRECATED**,其定义是`#define SPICE_GNUC_DEPRECATED  __attribute__((__deprecated__))`表示该函数以及被弃用,在编译时会给出警告
- **SPICE_GNUC_VISIBLE**,其定义是`#define SPICE_GNUC_VISIBLE __attribute__ ((visibility ("default")))`用于控制符号的可见性,设置为对外可见.spice动态链接库默认隐藏对外部的可见性,即外部文件不能调用库里面的函数,有这个声明的函数可以被外部文件调用,即为公共函数.

跟SpiceServer相关的函数在spice.h中定义,在reds.c中被定义,提供给外部调用
```
SPICE_GNUC_VISIBLE int spice_server_get_num_clients(SpiceServer *s)
SPICE_GNUC_VISIBLE int spice_server_add_client(SpiceServer *s, int socket, int skip_auth)
SPICE_GNUC_VISIBLE int spice_server_add_ssl_client(SpiceServer *s, int socket, int skip_auth)
SPICE_GNUC_VISIBLE void spice_server_char_device_wakeup(SpiceCharDeviceInstance* sin)
SPICE_GNUC_VISIBLE const char** spice_server_char_device_recognized_subtypes(void)
SPICE_GNUC_VISIBLE int spice_server_add_interface(SpiceServer *s,
SPICE_GNUC_VISIBLE int spice_server_remove_interface(SpiceBaseInstance *sin)
SPICE_GNUC_VISIBLE SpiceServer *spice_server_new(void)
SPICE_GNUC_VISIBLE int spice_server_init(SpiceServer *s, SpiceCoreInterface *core)
SPICE_GNUC_VISIBLE void spice_server_destroy(SpiceServer *s)
SPICE_GNUC_VISIBLE spice_compat_version_t spice_get_current_compat_version(void)
SPICE_GNUC_VISIBLE int spice_server_set_compat_version(SpiceServer *s,
SPICE_GNUC_VISIBLE int spice_server_set_port(SpiceServer *s, int port)
SPICE_GNUC_VISIBLE void spice_server_set_addr(SpiceServer *s, const char *addr, int flags)
SPICE_GNUC_VISIBLE int spice_server_set_listen_socket_fd(SpiceServer *s, int listen_fd)
SPICE_GNUC_VISIBLE int spice_server_set_exit_on_disconnect(SpiceServer *s, int flag)
SPICE_GNUC_VISIBLE int spice_server_set_noauth(SpiceServer *s)
SPICE_GNUC_VISIBLE int spice_server_set_sasl(SpiceServer *s, int enabled)
SPICE_GNUC_VISIBLE int spice_server_set_sasl_appname(SpiceServer *s, const char *appname)
SPICE_GNUC_VISIBLE void spice_server_set_name(SpiceServer *s, const char *name)
SPICE_GNUC_VISIBLE void spice_server_set_uuid(SpiceServer *s, const uint8_t uuid[16])
SPICE_GNUC_VISIBLE int spice_server_set_ticket(SpiceServer *s,
SPICE_GNUC_VISIBLE int spice_server_set_tls(SpiceServer *s, int port,
SPICE_GNUC_VISIBLE int spice_server_set_image_compression(SpiceServer *s,
SPICE_GNUC_VISIBLE spice_image_compression_t spice_server_get_image_compression(SpiceServer *s)
SPICE_GNUC_VISIBLE int spice_server_set_jpeg_compression(SpiceServer *s, spice_wan_compression_t comp)
SPICE_GNUC_VISIBLE int spice_server_set_zlib_glz_compression(SpiceServer *s, spice_wan_compression_t comp)
SPICE_GNUC_VISIBLE int spice_server_set_channel_security(SpiceServer *s, const char *channel, int security)
SPICE_GNUC_VISIBLE int spice_server_get_sock_info(SpiceServer *s, struct sockaddr *sa, socklen_t *salen)
SPICE_GNUC_VISIBLE int spice_server_get_peer_info(SpiceServer *s, struct sockaddr *sa, socklen_t *salen)
SPICE_GNUC_VISIBLE int spice_server_is_server_mouse(SpiceServer *s)
SPICE_GNUC_VISIBLE int spice_server_add_renderer(SpiceServer *s, const char *name)
SPICE_GNUC_VISIBLE int spice_server_kbd_leds(SpiceKbdInstance *sin, int leds)
SPICE_GNUC_VISIBLE int spice_server_set_streaming_video(SpiceServer *s, int value)
SPICE_GNUC_VISIBLE int spice_server_set_playback_compression(SpiceServer *s, int enable)
SPICE_GNUC_VISIBLE int spice_server_set_agent_mouse(SpiceServer *s, int enable)
SPICE_GNUC_VISIBLE int spice_server_set_agent_copypaste(SpiceServer *s, int enable)
SPICE_GNUC_VISIBLE int spice_server_set_agent_file_xfer(SpiceServer *s, int enable)
SPICE_GNUC_VISIBLE int spice_server_migrate_connect(SpiceServer *s, const char* dest,
SPICE_GNUC_VISIBLE int spice_server_migrate_info(SpiceServer *s, const char* dest,
SPICE_GNUC_VISIBLE int spice_server_migrate_start(SpiceServer *s)
SPICE_GNUC_VISIBLE int spice_server_migrate_client_state(SpiceServer *s)
SPICE_GNUC_VISIBLE int spice_server_migrate_end(SpiceServer *s, int completed)
SPICE_GNUC_VISIBLE int spice_server_migrate_switch(SpiceServer *s)
SPICE_GNUC_VISIBLE void spice_server_vm_start(SpiceServer *s)
SPICE_GNUC_VISIBLE void spice_server_vm_stop(SpiceServer *s)
SPICE_GNUC_VISIBLE void spice_server_set_seamless_migration(SpiceServer *s, int enable)
```

## 定义spice的各种接口:
- QXLInterface
- SpiceCharDeviceInterface
- SpiceCoreInterface: 创建,添加,取消定时和监听事件
- SpiceKbdInterface
- SpiceMigrateInterface
- SpiceMouseInterface
- SpicePlaybackInterface
- SpiceRecordInterface
- SpiceTabletInterface

## 重要的结构体
###SpiceServer
其实是定义在reds-private.h中的RedsState结构体.
```
int listen_socket;
int secure_listen_socket;
SpiceWatch *listen_watch;
SpiceWatch *secure_listen_watch;
VDIPortState agent_state;
Ring clients;
int num_clients;//
MainChannel *main_channel;
int num_of_channels;
Ring channels;
int mouse_mode;
int is_client_mouse_allowed;
int dispatcher_allows_client_mouse;
MonitorMode monitor_mode;
SpiceTimer *mm_timer;
RedsClientMonitorsConfig client_monitors_config;
int mm_timer_enabled;
uint32_t mm_time_latency;
```
可以看到,RedsState拥有通道环和客户端环

# channels

## red_channel.h

### 重要的结构体
- RedChannelClient
```
RingItem channel_link;
RingItem client_link;
RedChannel *channel;
RedClient  *client;
RedsStream *stream;
OutgoingHandler outgoing;//缓冲区信息+OutgoingHandlerInterface
IncomingHandler incoming;//缓冲区信息+IncomingHandlerInterface
struct ack_data;
struct send_data;
RedChannelClientLatencyMonitor latency_monitor;//延迟监听器
RedChannelClientConnectivityMonitor connectivity_monitor;//连接监听器
```
- RedChannel
    ```
    uint32_t type;
    uint32_t id;
    uint32_t refs;
    RingItem link; // channels link for reds
    SpiceCoreInterface *core;
    int handle_acks;
    Ring clients;//实质是RedChannelClient
    uint32_t clients_num;

    OutgoingHandlerInterface outgoing_cb;
    IncomingHandlerInterface incoming_cb;

    ChannelCbs channel_cbs;
    ClientCbs client_cbs;

    RedChannelCapabilities local_caps;
    uint32_t migration_flags;

    void *data;

    pthread_t thread_id;
    ```
    所有channel的父类,虽然C没有类的概念,但是隐含了这一层的意思,可以说是C的类实现.
    - SpiceCoreInterface
    - OutgoingHandlerInterface:定义处理输出消息的函数接口
    - IncomingHandlerInterface:定义处理输入消息的函数接口
    - ChannelCbs:用于处理客户端数据流事件的回调函数,被监听数据流事件的线程调用
    - ClientCbs:用于处理客户端事件的回调函数,例如连接事件等,被处理RedClient的线程调用

    RedChannel持有已连接的客户端通道(RedChannelClient),当连接断开时负责销毁它们.

- RedClient
```
RingItem link;
Ring channels;//其他通道的环
MainChannelClient *mcc;
pthread_mutex_t lock; // different channels can be in different threads
pthread_t thread_id;
int refs; 
```

### 重要函数
- red_channel_create
```
 RedChannel *red_channel_create(int size,
                               SpiceCoreInterface *core,
                               uint32_t type, uint32_t id,
                               int handle_acks,
                               channel_handle_message_proc handle_message,
                               ChannelCbs *channel_cbs,
                               uint32_t migration_flags)
```
