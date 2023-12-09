---
layout: post
title: "Redis-Jedis的MOVED_ASK实现"
author: "Inela"
---

​	Jedis-2.9.0版本，最近在看Redis Cluster的槽位(Slot)迁移，其中涉及MOVED、ASK命令

1. MOVED：key已被迁移到目标节点node上(已迁移完成)，则返回MOVED命令并携带目标节点的node，刷新客户端的槽缓存，Redis客户端收到后再次请求新目标节点
2. ASK：为什么需要ASK命令，为什么不能仅用MOVED解决？MOVED命令表示后续该key都由新节点处理，ASK命令仅仅表示单个这次的key处理由新节点尝试解决，后面的该key还是继续会发到之前的节点



​	Jedis客户端实现如下，

1. 通过CRC16计算出槽位对应的目标节点，之后请求目标节点
2. 如果请求出错，则Jedis会重试maxAttemps次，重试完抛出Too many Cluster redirections
3. 如果Redis Cluster返回MOVED、ASK命令，底层是抛出异常，而后捕获到，如果是MOVED命令则刷新本地的槽位信息(发送slots命令给Redis节点)，这时槽位对应的node可能变了，再次从本地缓存中槽位对应的node进行请求。
4. 如果是ASK命令，则请求ASK命令携带的目标节点B，使用的是ASKING命令发送到B，之后再次将业务命令发送到B节点

![Jedis MOVED ASK ASKING命令处理](https://github.com/MingJunDuan/mingjunduan.github.io/raw/main/images/mjduan/2023-12-08/Jedis-MOVED-ASK-ASKING命令处理.png)



​	#JedisClusterCommand.java

```
...
	public T run(String key) {
    if (key == null) {
      throw new JedisClusterException("No way to dispatch this command to Redis Cluster.");
    }
		//1.maxAttempts表示尝试次数
    return runWithRetries(SafeEncoder.encode(key), this.maxAttempts, false, false);
  }
...

  private T runWithRetries(byte[] key, int attempts, boolean tryRandomNode, boolean asking) {
    //2.尝试完attempts则抛出异常
    if (attempts <= 0) {
      throw new JedisClusterMaxRedirectionsException("Too many Cluster redirections?");
    }

    Jedis connection = null;
    try {
			
      if (asking) {
        // TODO: Pipeline asking with the original command to make it
        // faster....
        connection = askConnection.get();
        //7.先发送ASKING命令，之后发送Redis的命令(如Get/Set)
        connection.asking();

        // if asking success, reset asking flag
        asking = false;
      } else {
        if (tryRandomNode) {
          connection = connectionHandler.getConnection();
        } else {
          //3.通过CRC16计算出槽位对应的节点
          connection = connectionHandler.getConnectionFromSlot(JedisClusterCRC16.getSlot(key));
        }
      }
			//请求对应的Redis目标节点
      return execute(connection);
    } catch (JedisNoReachableClusterNodeException jnrcne) {
      throw jnrcne;
    } catch (JedisConnectionException jce) {
      // release current connection before recursion
      releaseConnection(connection);
      connection = null;

      if (attempts <= 1) {
        //We need this because if node is not reachable anymore - we need to finally initiate slots renewing,
        //or we can stuck with cluster state without one node in opposite case.
        //But now if maxAttempts = 1 or 2 we will do it too often. For each time-outed request.
        //TODO make tracking of successful/unsuccessful operations for node - do renewing only
        //if there were no successful responses from this node last few seconds
        this.connectionHandler.renewSlotCache();
        //no more redirections left, throw original exception, not JedisClusterMaxRedirectionsException, because it's not MOVED situation
        throw jce;
      }

      return runWithRetries(key, attempts - 1, tryRandomNode, asking);
    } catch (JedisRedirectionException jre) {
      // if MOVED redirection occurred,
      //4.如果Redis服务端返回MOVED命令，则刷新本地的槽位Slot信息
      if (jre instanceof JedisMovedDataException) {
        // it rebuilds cluster's slot cache
        // recommended by Redis cluster specification
        //发送slots命令给Redis节点获取槽位信息
        this.connectionHandler.renewSlotCache(connection);
      }

      // release current connection before recursion or renewing
      releaseConnection(connection);
      connection = null;
			//5.如果Redis服务端返回ASK命令
      if (jre instanceof JedisAskDataException) {
        asking = true;
        //6.请求ASK命令携带的目标节点
        askConnection.set(this.connectionHandler.getConnectionFromNode(jre.getTargetNode()));
      } else if (jre instanceof JedisMovedDataException) {
      } else {
        throw new JedisClusterException(jre);
      }
			
      return runWithRetries(key, attempts - 1, false, asking);
    } finally {
      releaseConnection(connection);
    }
  }
...
```



​	Jedis解析Redis的Resp协议响应时，头字符是 **-** 则表示异常，[可参考Redis的Resp协议](https://mingjunduan.github.io/2023-11-21/Redis-%E9%80%9A%E4%BF%A1Resp%E5%8D%8F%E8%AE%AE%E5%89%96%E6%9E%90)

#Protocol.java

```
  private static Object process(final RedisInputStream is) {
    final byte b = is.readByte();
    if (b == PLUS_BYTE) {
      return processStatusCodeReply(is);
    } else if (b == DOLLAR_BYTE) {
      return processBulkReply(is);
    } else if (b == ASTERISK_BYTE) {
      return processMultiBulkReply(is);
    } else if (b == COLON_BYTE) {
      return processInteger(is);
    } else if (b == MINUS_BYTE) {
    	//响应命令以-开头则表示错误
      processError(is);
      return null;
    } else {
      throw new JedisConnectionException("Unknown reply: " + (char) b);
    }
  }
  ...
  private static void processError(final RedisInputStream is) {
    String message = is.readLine();
    //是MOVED命令，则抛出JedisMovedDataException命令
    if (message.startsWith(MOVED_RESPONSE)) {
      String[] movedInfo = parseTargetHostAndSlot(message);
      throw new JedisMovedDataException(message, new HostAndPort(movedInfo[1],
          Integer.valueOf(movedInfo[2])), Integer.valueOf(movedInfo[0]));
    //是ASK命令，则抛出JedisAskDataException命令      
    } else if (message.startsWith(ASK_RESPONSE)) {
      String[] askInfo = parseTargetHostAndSlot(message);
      throw new JedisAskDataException(message, new HostAndPort(askInfo[1],
          Integer.valueOf(askInfo[2])), Integer.valueOf(askInfo[0]));
    } else if (message.startsWith(CLUSTERDOWN_RESPONSE)) {
      throw new JedisClusterException(message);
    } else if (message.startsWith(BUSY_RESPONSE)) {
      throw new JedisBusyException(message);
    } else if (message.startsWith(NOSCRIPT_RESPONSE) ) {
      throw new JedisNoScriptException(message);
    }
    throw new JedisDataException(message);
  }
```



参考：

1. Redis Cluster Spec：https://redis.io/docs/reference/cluster-spec/#ask-redirection