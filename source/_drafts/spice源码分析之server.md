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
    void (*update_area_complete)(QXLInstance *qin, uint32_t surface_id,
                                 struct QXLRect *updated_rects,
                                 uint32_t num_updated_rects);
    void (*set_client_capabilities)(QXLInstance *qin,
                                    uint8_t client_present,
                                    uint8_t caps[58]);
    /* returns 1 if the interface is supported, 0 otherwise.
     * if monitors_config is NULL nothing is done except reporting the
     * return code. */
    int (*client_monitors_config)(QXLInstance *qin,
                                  VDAgentMonitorsConfig *monitors_config);
```
- SpiceCharDeviceInterface
```
    void (*state)(SpiceCharDeviceInstance *sin, int connected);
    int (*write)(SpiceCharDeviceInstance *sin, const uint8_t *buf, int len);
    int (*read)(SpiceCharDeviceInstance *sin, uint8_t *buf, int len);
    void (*event)(SpiceCharDeviceInstance *sin, uint8_t event);
```
- SpiceCoreInterface: 创建,添加,取消定时和监听事件
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
- SpiceKbdInterface
```
    void (*push_scan_freg)(SpiceKbdInstance *sin, uint8_t frag);
    uint8_t (*get_leds)(SpiceKbdInstance *sin);
```
- SpiceMigrateInterface
```
    void (*migrate_connect_complete)(SpiceMigrateInstance *sin);
    void (*migrate_end_complete)(SpiceMigrateInstance *sin);
```
- SpiceMouseInterface
```
    void (*motion)(SpiceMouseInstance *sin, int dx, int dy, int dz,
                   uint32_t buttons_state);
    void (*buttons)(SpiceMouseInstance *sin, uint32_t buttons_state);
```
- SpicePlaybackInterface
- SpiceRecordInterface
- SpiceTabletInterface
```
    void (*set_logical_size)(SpiceTabletInstance* tablet, int width, int height);
    void (*position)(SpiceTabletInstance* tablet, int x, int y, uint32_t buttons_state);
    void (*wheel)(SpiceTabletInstance* tablet, int wheel_moution, uint32_t buttons_state);
    void (*buttons)(SpiceTabletInstance* tablet, uint32_t buttons_state);

```

## 重要的结构体
### SpiceServer
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
Channel是spice服务器与qemu和客户端传输信息的通道.
## red_channel.h
服务器包括了两种Channel,一种是与qemu连接的,一种是与客户端连接.

### 重要的结构体
- **RedChannel**:所有Channel的父结构体.
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
    - SpiceCoreInterface
    - OutgoingHandlerInterface:定义处理输出消息的函数接口
    - IncomingHandlerInterface:定义处理输入消息的函数接口
    - ChannelCbs:用于处理客户端数据流事件的回调函数,被监听数据流事件的线程调用
    - ClientCbs:用于处理客户端事件的回调函数,例如连接事件等,被处理RedClient的线程调用

    RedChannel持有已连接的客户端通道(RedChannelClient),当连接断开时负责销毁它们.

- **RedChannelClient**:与客户端连接的通道结构体的父结构体.
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

- **RedClient**:客户端结构体,包含客户端的信息
    ```
RingItem link;
Ring channels;//其他通道的环
MainChannelClient *mcc;
pthread_mutex_t lock; // different channels can be in different threads
pthread_t thread_id;
int refs; 
    ```
    RedClient在reds_handle_main_link中被创建,并且加入reds的环中.然后MainChannel创建一个MainChannelClient,并且将其赋予RedClient

### 重要函数
- **red_channel_create**
    ```
 RedChannel *red_channel_create(int size,
                               SpiceCoreInterface *core,
                               uint32_t type,//通道类型
                               uint32_t id,
                               int handle_acks,
                               channel_handle_message_proc handle_message,
                               ChannelCbs *channel_cbs,
                               uint32_t migration_flags)
    ```
    通道类型包括
    ```
    SPICE_CHANNEL_MAIN = 1,
    SPICE_CHANNEL_DISPLAY,
    SPICE_CHANNEL_INPUTS,
    SPICE_CHANNEL_CURSOR,
    SPICE_CHANNEL_PLAYBACK,
    SPICE_CHANNEL_RECORD,
    SPICE_CHANNEL_TUNNEL,
    SPICE_CHANNEL_SMARTCARD,
    SPICE_CHANNEL_USBREDIR,
    SPICE_CHANNEL_PORT,
    SPICE_CHANNEL_WEBDAV,
    ```
- **red_channel_client_create**
```
RedChannelClient *red_channel_client_create(int size, RedChannel *channel, RedClient  *client,
                                            RedsStream *stream,
                                            int monitor_latency,
                                            int num_common_caps, uint32_t *common_caps,
                                            int num_caps, uint32_t *caps)
```

## main_channel.h

## inputs_channel.h



# dispatcher

## dispatcher.h

### 重要的结构体

- **Dispatcher**
```
    SpiceCoreInterface *recv_core;
    int recv_fd;
    int send_fd;
    pthread_t self;
    pthread_mutex_t lock;
    DispatcherMessage *messages;//调度消息动态数组,根据max_message_type的值分配.
    int stage;  /* message parser stage - sender has no stages */
    size_t max_message_type;//消息的种类
    void *payload; /* allocated as max of message sizes */
    size_t payload_size; /* used to track realloc calls */
    void *opaque;
    dispatcher_handle_async_done handle_async_done;
```
- **DispatcherMessage**
    ```
    size_t size;
    int ack;
    dispatcher_handle_message handler;
    ```
### 重要函数
- **dispatcher_init**
    ```
void dispatcher_init(Dispatcher *dispatcher, size_t max_message_type,
                     void *opaque)
    ```
    打开进程间套接字(send_fd,recv_fd),初始化线程锁(lock)和消息类型(message).
    
## red_dispatcher.h
定义了跟QXLWorker和red_dispatcher相关的函数.
- 调度器与QXL设备实例一对一
- 从QXL设备和Reds中封装worker内部构件
- 为每个QXL设备实例初始化一个worker并创建一个worker线程
- 使用socketpair分发给worker
- QXL设备使用QXLWorker接口，这个接口在red dispathcher中实现，它把设备调用翻译成消息通过red worker管道传输。这种方式使QXL设备和调度器分开并且逻辑上独立。
- Reds使用定义在red_dispatcher.h中的接口来使用调度器的功能如调度器初始化，改变图像压缩率，改变视频流状态，鼠标模式设置，额外渲染。
### 重要的结构体
- **RedDispatcher**
    ```
    QXLWorker base;
    QXLInstance *qxl;
    Dispatcher dispatcher;
    pthread_t worker_thread;
    uint32_t pending;
    int primary_active;
    int x_res;
    int y_res;
    int use_hardware_cursor;
    RedDispatcher *next;
    Ring async_commands;
    pthread_mutex_t  async_lock;
    QXLDevSurfaceCreate surface_create;
    ```
    用于调度RedWorker,QXLWorker.对应一个worker线程.主要负责图像处理的调度.

### 重要函数

- **对外可见函数**
    ```
SPICE_GNUC_VISIBLE void spice_qxl_wakeup(QXLInstance *instance)
SPICE_GNUC_VISIBLE void spice_qxl_oom(QXLInstance *instance)
SPICE_GNUC_VISIBLE void spice_qxl_start(QXLInstance *instance)
SPICE_GNUC_VISIBLE void spice_qxl_stop(QXLInstance *instance)
SPICE_GNUC_VISIBLE void spice_qxl_update_area(QXLInstance *instance, uint32_t surface_id,
SPICE_GNUC_VISIBLE void spice_qxl_add_memslot(QXLInstance *instance, QXLDevMemSlot *slot)
SPICE_GNUC_VISIBLE void spice_qxl_del_memslot(QXLInstance *instance, uint32_t slot_group_id, uint32_t slot_id)
SPICE_GNUC_VISIBLE void spice_qxl_reset_memslots(QXLInstance *instance)
SPICE_GNUC_VISIBLE void spice_qxl_destroy_surfaces(QXLInstance *instance)
SPICE_GNUC_VISIBLE void spice_qxl_destroy_primary_surface(QXLInstance *instance, uint32_t surface_id)
SPICE_GNUC_VISIBLE void spice_qxl_create_primary_surface(QXLInstance *instance, uint32_t surface_id,
SPICE_GNUC_VISIBLE void spice_qxl_reset_image_cache(QXLInstance *instance)
SPICE_GNUC_VISIBLE void spice_qxl_reset_cursor(QXLInstance *instance)
SPICE_GNUC_VISIBLE void spice_qxl_destroy_surface_wait(QXLInstance *instance, uint32_t surface_id)
SPICE_GNUC_VISIBLE void spice_qxl_loadvm_commands(QXLInstance *instance, struct QXLCommandExt *ext, uint32_t count)
SPICE_GNUC_VISIBLE void spice_qxl_update_area_async(QXLInstance *instance, uint32_t surface_id, QXLRect *qxl_area,
SPICE_GNUC_VISIBLE void spice_qxl_add_memslot_async(QXLInstance *instance, QXLDevMemSlot *slot, uint64_t cookie)
SPICE_GNUC_VISIBLE void spice_qxl_destroy_surfaces_async(QXLInstance *instance, uint64_t cookie)
SPICE_GNUC_VISIBLE void spice_qxl_destroy_primary_surface_async(QXLInstance *instance, uint32_t surface_id, uint64_t cookie)
SPICE_GNUC_VISIBLE void spice_qxl_create_primary_surface_async(QXLInstance *instance, uint32_t surface_id,
SPICE_GNUC_VISIBLE void spice_qxl_destroy_surface_async(QXLInstance *instance, uint32_t surface_id, uint64_t cookie)
SPICE_GNUC_VISIBLE void spice_qxl_flush_surfaces_async(QXLInstance *instance, uint64_t cookie)
SPICE_GNUC_VISIBLE void spice_qxl_monitors_config_async(QXLInstance *instance, QXLPHYSICAL monitors_config,
SPICE_GNUC_VISIBLE void spice_qxl_driver_unload(QXLInstance *instance)
    ```
    实际上是对内部函数的封装,提供给外部调用.
## main_dispatcher.h

### 重要的结构体
- **MainDispatcher**
    ```
    Dispatcher base;
    SpiceCoreInterface *core;
    ```

### 重要函数
- **main_dispatcher_init**
    ```
void main_dispatcher_init(SpiceCoreInterface *core)
    ```
    设置SpiceCoreInterface
    监听recv_fd的读事件`core->watch_add(main_dispatcher.base.recv_fd, SPICE_WATCH_EVENT_READ,dispatcher_handle_read, &main_dispatcher.base);`
    配置各种消息的handler(调用dispatcher_register_handler)
    四种消息
    - MAIN_DISPATCHER_CHANNEL_EVENT = 0,
    - MAIN_DISPATCHER_MIGRATE_SEAMLESS_DST_COMPLETE,
    - MAIN_DISPATCHER_SET_MM_TIME_LATENCY,
    - MAIN_DISPATCHER_CLIENT_DISCONNECT,


- **dispatcher_register_handler**
    ```
void dispatcher_register_handler(Dispatcher *dispatcher, uint32_t message_type,
                                 dispatcher_handle_message handler,
                                 size_t size, int ack)
    ```
    设置handler
    设置payload

# red_worker.h
## 重要的结构体
- **RedWorker**
    ```
    DisplayChannel *display_channel;
    CursorChannel *cursor_channel;
    QXLInstance *qxl;
    RedDispatcher *red_dispatcher;

    int channel;//dispatcher的recv_fd
    int id;
    int running;
    uint32_t *pending;
    struct pollfd poll_fds[MAX_EVENT_SOURCES];//poll事件驱动
    struct SpiceWatch watches[MAX_EVENT_SOURCES];//监听事件
    unsigned int event_timeout;
    uint32_t repoll_cmd_ring;
    uint32_t repoll_cursor_ring;
    uint32_t num_renderers;
    uint32_t renderers[RED_MAX_RENDERERS];
    uint32_t renderer;

    RedSurface surfaces[NUM_SURFACES];
    uint32_t n_surfaces;
    SpiceImageSurfaces image_surfaces;

    MonitorsConfig *monitors_config;

    Ring current_list;
    uint32_t current_size;
    uint32_t drawable_count;
    uint32_t red_drawable_count;
    uint32_t glz_drawable_count;
    uint32_t transparent_count;

    uint32_t shadows_count;
    uint32_t containers_count;
    uint32_t stream_count;

    uint32_t bits_unique;

    CursorItem *cursor;
    int cursor_visible;
    SpicePoint16 cursor_position;
    uint16_t cursor_trail_length;
    uint16_t cursor_trail_frequency;

    _Drawable drawables[NUM_DRAWABLES];
    _Drawable *free_drawables;

    _CursorItem cursor_items[NUM_CURSORS];
    _CursorItem *free_cursor_items;

    RedMemSlotInfo mem_slots;

    uint32_t preload_group_id;

    ImageCache image_cache;

    spice_image_compression_t image_compression;
    spice_wan_compression_t jpeg_state;
    spice_wan_compression_t zlib_glz_state;

    uint32_t mouse_mode;

    uint32_t streaming_video;
    Stream streams_buf[NUM_STREAMS];
    Stream *free_streams;
    Ring streams;
    ItemTrace items_trace[NUM_TRACE_ITEMS];
    uint32_t next_item_trace;
    uint64_t streams_size_total;
    //图像压缩算法
    QuicData quic_data;
    QuicContext *quic;
    //图像压缩算法
    LzData lz_data;
    LzContext  *lz;
    //jpeg编码
    JpegData jpeg_data;
    JpegEncoderContext *jpeg;
    //压缩库
    ZlibData zlib_data;
    ZlibEncoder *zlib;

    uint32_t process_commands_generation;

    int driver_cap_monitors_config;
    int set_client_capabilities_pending;

    ```
- **SpiceWatch**
    ```
    struct RedWorker *worker;
    SpiceWatchFunc watch_func;
    void *watch_func_opaque;
    ```
## 重要函数
- **red_worker_main**
    在red_dispatcher_init中创建线程被调用.
    - 处理QXL设备命令（绘图，更新，指针等）
    - 处理从调度器收到的消息
    - 通道传输
    - 管理Display 和 Cursor 通道
    - 图像压缩（使用 quic, lz, glzencoding)
    - 视频流-识别，编码和流创建
    - 缓存-客户端共享像图缓存，指针缓存，调色板缓存
    - 图像移除优化-正在使用项树，容器，阴影，临界区域，不透明项。
    - Cairo 和 OpenGL渲染-canvas,surface
    - 对于环的操作

