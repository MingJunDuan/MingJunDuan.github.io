---
layout: post
title: "Redis-通信Resp协议剖析"
author: "Inela"
---

Redis客户端与服务器交互采用**序列化协议（RESP）,**请求以**字符串数组**的形式来表示要执行**命令的参数。**用命令特有（command-specific）数据类型作为回复。

**通信协议的特点**
客户端和服务器通过 **TCP 连接来进行数据交互**， 服务器默认的端口号为 **6379** 。
客户端和服务器发送的命令或数据一律以 **\r\n （CRLF）结尾**。
在这个协议中， 所有发送至 Redis 服务器的参数都是**二进制安全**（binary safe）的。
简单，高效，易读。



我们都知道调用Redis的命令是这样的：`set username afei`，`hmset Person:1 username afei password 123456`，那么Redis真正接收的请求是什么样的呢？即Redis定义的协议是怎么样的?



## Redis Resp协议

Redis Resp协议官方定义如下：

```
*<number of arguments> CR LF //参数个数
$<number of bytes of argument 1> CR LF //参数的bytes数
<argument data> CR LF  //参数
...
$<number of bytes of argument N> CR LF //参数的bytes数
<argument data> CR LF  //参数
```

例子如下：

```
*3
$3
SET
$5
mykey
$7
myvalue
```

上面的命令看上去像是单引号字符串，所以可以在查询中看到每个字节的准确值：

```
"*3\r\n$3\r\nSET\r\n$5\r\nmykey\r\n$7\r\nmyvalue\r\n"
```

注：CR LF事实上就是\r\n，即window平台的换行

## Redis协议文件样例

通过Redis对协议的定义，我们可以自己写出redis协议样例, 如下所示：

**String类型**之`set username afei`对应的redis协议文本：

```bash
*3 //3个参数：set,username,afei
$3 //参数set占用3bytes
set //参数set
$8 //参数username占用8bytes
username //参数username
$4 //参数afei的bytes
afei //参数afei
```

### Redis协议文件解读:

`*3` 表示这个命令有3个参数；
 `$3` 表示第一个参数的长度是3
 `set` 表示定义长度为$3的参数
 `$8` 表示第二个参数的长度是8
 `usename` 表示定义长度为$8的参数
 `$4` 表示第三个参数的长度是4
 `afei` 表示定义长度为$4的参数

## Redis的Jedis客户端

通过Jedis这个redis的client包，执行一个基本的命令:`get Device:99`为例，跟踪Jedis源码可知调用了redis.clients.jedis.Jedis中的get方法：

```java
public String get(final String key) {
    checkIsInMultiOrPipeline();
    client.sendCommand(Protocol.Command.GET, key);
    return client.getBulkReply();
  }
```

再调用redis.clients.jedis.Connection中的：



```java
protected Connection sendCommand(final Command cmd, final String... args) {
    final byte[][] bargs = new byte[args.length][];
    for (int i = 0; i < args.length; i++) {
      bargs[i] = SafeEncoder.encode(args[i]);
    }
    return sendCommand(cmd, bargs);
  }
```

**代码解读**：SafeEncoder.encode(args[i])实际上就是执行这个方法：str.getBytes(Protocol.CHARSET);，我这里的测试时cache key为Device:99，SafeEncoder.encode(args[i])的结果是：new byte[]{68,101,118,105,99,101,58,57,57}，68这个ASCII值对应的字符就是D，101这个ASCII值对应的字符就是e，这个byte[]数值就是表示此次测试的cache key：Device:99；



最终实际调用redis.clients.jedis.Protocol.java中的：

```java
private static void sendCommand(final RedisOutputStream os, final byte[] command,
      final byte[]... args) {
    try {
      os.write(ASTERISK_BYTE);
      os.writeIntCrLf(args.length + 1);
      os.write(DOLLAR_BYTE);
      os.writeIntCrLf(command.length);
      os.write(command);
      os.writeCrLf();

      for (final byte[] arg : args) {
        os.write(DOLLAR_BYTE);
        os.writeIntCrLf(arg.length);
        os.write(arg);
        os.writeCrLf();
      }
    } catch (IOException e) {
      throw new JedisConnectionException(e);
    }
  }
```



> **代码解读**：从上面这段Java代码可以看出一些redis协议的端倪了：
>  `os.write(ASTERISK_BYTE)`: 即向Redis服务器Socket端口写入一个星号"*"；
>  `os.writeIntCrLf(args.length + 1)` 写入本次执行命令的总参数长度，为参数长度1+1，1表示有一个GET命令；
>  `os.write(DOLLAR_BYTE)` 写入一个Dollar符号："$"
>  `os.writeIntCrLf(command.length)` 写入一个GET命令的长度并写入"\r\n"（CrLf）；
>  `os.write(command)` 写入GET命令：GET命令已经转为byte数值（new byte[]{71, 69, 84}）
>  `os.writeCrLf()` 写入"\r\n"（CrLf）
>  接下来for循环遍历参数，本次测试只有一个参数：`Device:99`，所以for遍历一次即可；for循环中解读：
>  `os.write(DOLLAR_BYTE)` 写入一个Dollar符号："$"
>  `os.writeIntCrLf(arg.length)` 写入参数的长度并写入"\r\n"（CrLf）；
>  `os.write(arg)` 写入参数
>  `os.writeCrLf()` 写入"\r\n"（CrLf）；

### 反推Redis协议文本

通过上面的代码解读，反推出执行`GET Device:99`这个命令的Redis Resp协议:

```bash
*2\r\n
$3\r\n
71 69 84\r\n
$9\r\n
68 101 118 105 99 101 58 57 57\r\n
```

转化为协议文本（文本需要unix2dos从而补上\r\n）：

```bash
*2
$3
GET
$9
Device:99
```



## 参考：

1. https://www.jianshu.com/p/36a5935db85b
2. https://www.jianshu.com/p/78b94407f59c
