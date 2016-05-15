title: RedisHttpSession 的设计与实现
date: 2016-05-15 10:49:39
tags: [Java, Session]
categories: Java
---

## 前言
[RedisHttpSession][1] 是我的一个 Java 开源项目，通过将 Session 存储在 Redis 中实现多服务器间共享 Session，同时这一过程是完全透明的。主要用于支持 RESTfuls API。下面我将对其核心类进行分析，阐述它的设计以及实现细节。

## RedisHttpSession
这个类实现了`HttpSession`接口，用于替换默认的`HttpSession`实现。`RedisHttpSession`将`HttpSession`的接口方法重写了一遍，将`HttpSession`的属性存储到了 Redis 中。

每个`RedisHttpSession`都有一个 UUID 与之对应，该字段加上`session:`前缀作为存储在 Redis 中的键值。例如：

    localhost:63679> keys *
    1) "session:fd9ec3cf-fb9b-4672-ade6-67a810e7db9f"
    2) "session:cbaa057c-85a4-475d-b399-38c320e85dcc"
    3) "session:13e030f5-de3d-458f-8d25-fd5643c40ff0"
    4) "session:262596b3-3d13-4df1-8328-714153c1ae83"
    5) "session:0b7d04c6-eaac-4eed-a9aa-8366f25f04f0"

同时，`RedisHttpSession`中的属性是直接序列化成字节数组存储在 Redis 中的，存储在对应键中的哈希表里面。例如：

    localhost:63679> hgetall session:fd9ec3cf-fb9b-4672-ade6-67a810e7db9f
    1) "lastAccessedTime"
    2) "\xac\xed\x00\x05sr\x00\x0ejava.lang.Long;\x8b\xe4\x90\xcc\x8f#\xdf\x02\x00\x01J\x00\x05valuexr\x00\x10java.lang.Number\x86\xac\x95\x1d\x0b\x94\xe0\x8b\x02\x00\x00xp\x00\x00\x01T\x91\x03\"\xec"
    3) "maxInactiveInterval"
    4) "\xac\xed\x00\x05sr\x00\x11java.lang.Integer\x12\xe2\xa0\xa4\xf7\x81\x878\x02\x00\x01I\x00\x05valuexr\x00\x10java.lang.Number\x86\xac\x95\x1d\x0b\x94\xe0\x8b\x02\x00\x00xp\x00\x00\a\b"
    5) "creationTime"
    6) "\xac\xed\x00\x05sr\x00\x0ejava.lang.Long;\x8b\xe4\x90\xcc\x8f#\xdf\x02\x00\x01J\x00\x05valuexr\x00\x10java.lang.Number\x86\xac\x95\x1d\x0b\x94\xe0\x8b\x02\x00\x00xp\x00\x00\x01T\x91\x03\"\xb4"

另外，Session 的自动过期是通过 Redis 设置键的过期时间实现的。

## RedisHttpSessionProxy
`RedisHttpSessionProxy`是`RedisHttpSession`的代理类，使用了 JDK 的动态代理。为什么要引入代理？这是基于下面的考虑做出的设计。

1. 由于 Session 存储在 Redis 中，在执行每个`RedisHttpSession`的接口方法之前都需要检查 Redis 连接是否可用。
2. 访问一个已经被注销的 Session 需要抛出异常。
3. 每次访问 Session 需要刷新过期时间和最后访问时间。

基于上面的考虑，对于每个`RedisHttpSession`的接口方法，我们都需要进行重复的操作，因此使用动态代理对每个接口方法进行增强是最合适的。代码如下：

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        RedisHttpSession session = (RedisHttpSession) originalObj;
        //check redis connection
        RedisConnection connection = session.getRedisConnection();
        if (!connection.isConnected()){
            connection.close();
            session.setRedisConnection(repository.getRedisConnection());
        }
        //For every methods of interface, check it valid or not
        if (session.isInvalidated()){
            throw new IllegalStateException("The HttpSession has already be invalidated.");
        } else {
            Object result =  method.invoke(originalObj, args);
            //if not invalidate method, refresh expireTime and lastAccessedTime;
            if (!method.getName().equals("invalidate")) {
                session.refresh();
                session.setLastAccessedTime(System.currentTimeMillis());
            }
            return result;
        }
    }

## RedisHttpSessionFilter
`RedisHttpSessionFilter`作为过滤器，将请求和响应替换成`RedisSessionRequestWrapper`和`RedisSessionResponseWrapper`，利用了装饰器模式动态的给请求和响应进行增强：

1. `RedisSessionRequestWrapper`：重写了`getSession`，替换默认的`HttpSession`实现为`RedisHttpSession`。
2. `RedisSessionResponseWrapper`：在响应的头部中加入`x-auth-token`字段，作为 Session 的 ID。客户端之后的请求都需要附带该字段，以便服务端识别对应的 Session。

## 总结
[RedisHttpSession][1] 利用 Filter 将请求的 Session 替换成 `RedisHttpSession`，在过滤阶段偷梁换柱，在之后对 Session 的操作都无需关心其内部实现，整个过程都是透明的。


  [1]: https://github.com/x-hansong/RedisHttpSession