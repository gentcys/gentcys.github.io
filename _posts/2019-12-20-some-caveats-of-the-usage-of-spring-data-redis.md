---
layout: post
title: "使用 Spring Data Redis 踩坑总结"
date: 2019-12-20 10:44:55 +0800
category: posts
comments: true
---

## Spring Data Redis 使用踩坑总结

> 本篇博客仅描述本人使用 Spring Data Redis 过程中遇到的一些坑，关于什么是 Spring Data Redis 以及怎么使用在这里不做赘述。

前段时间公司的项目开始使用 Redis 作为缓存。有一天，同事告诉我连接池连接被占满，造成后续请求阻塞但是请求数量并不是很大，让我有空帮忙查一下是什么问题造成的。不管怎样，刷一下源代码先。

Spring Data Redis 提供一个操作类 RedisTemplate 给外部操作 Redis。日常的操作最终都是执行其类中的 `execute` 方法。

```Java
@Nullable
public <T> T execute(RedisCallback<T> action, boolean exposeConnection, boolean pipeline) {

   	Assert.isTrue(initialized, "template not initialized; call afterPropertiesSet() before using it");
   	Assert.notNull(action, "Callback object must not be null");

   	RedisConnectionFactory factory = getRequiredConnectionFactory();
   	RedisConnection conn = null;
	try {

	   	if (enableTransactionSupport) {
	   		// only bind resources in case of potential transaction synchronization
            // 绑定一个连接到当前线程
	   		conn = RedisConnectionUtils.bindConnection(factory, enableTransactionSupport);
	   	} else {
	   		conn = RedisConnectionUtils.getConnection(factory);
	   	}

			boolean existingConnection = TransactionSynchronizationManager.hasResource(factory);

			RedisConnection connToUse = preProcessConnection(conn, existingConnection);

			boolean pipelineStatus = connToUse.isPipelined();
			if (pipeline && !pipelineStatus) {
				connToUse.openPipeline();
			}

			RedisConnection connToExpose = (exposeConnection ? connToUse : createRedisConnectionProxy(connToUse));
			T result = action.doInRedis(connToExpose);

			// close pipeline
			if (pipeline && !pipelineStatus) {
				connToUse.closePipeline();
			}

			// TODO: any other connection processing?
			return postProcessResult(result, connToUse, existingConnection);
		} finally {
            // finally 关键字限定 try catch 最终执行的语句
			RedisConnectionUtils.releaseConnection(conn, factory, enableTransactionSupport);
		}
	}
```

注意 `enableTransactionSupport` 在项目中我们开启了事务，所以将会运行 `conn = RedisConnectionUtils.bindConnection(factory, enableTransactionSupport);` 绑定一个连接到当前线程，具体看看是如何绑定的。

```Java
public static RedisConnection doGetConnection(RedisConnectionFactory factory, boolean allowCreate, boolean bind,
			boolean enableTransactionSupport) {

	Assert.notNull(factory, "No RedisConnectionFactory specified");

    // 先查看当前线程是否已经绑定连接
	RedisConnectionHolder connHolder = (RedisConnectionHolder) TransactionSynchronizationManager.getResource(factory);

	if (connHolder != null) {
	   	if (enableTransactionSupport) {
	   		potentiallyRegisterTransactionSynchronisation(connHolder, factory);
	   	}
			return connHolder.getConnection();
		}

		if (!allowCreate) {
			throw new IllegalArgumentException("No connection found and allowCreate = false");
		}

		if (log.isDebugEnabled()) {
			log.debug("Opening RedisConnection");
		}

        // 没有的话就新建一个连接
		RedisConnection conn = factory.getConnection();

		if (bind) {

			RedisConnection connectionToBind = conn;
			if (enableTransactionSupport && isActualNonReadonlyTransactionActive()) {
				connectionToBind = createConnectionProxy(conn, factory);
			}

			connHolder = new RedisConnectionHolder(connectionToBind);

           // 然后再绑定到当前线程
			TransactionSynchronizationManager.bindResource(factory, connHolder);
			if (enableTransactionSupport) {
				potentiallyRegisterTransactionSynchronisation(connHolder, factory);
			}

            // 返回绑定的连接
			return connHolder.getConnection();
		}

		return conn;
	}
```

`RedisConnectionUtils.bindConnection()` 内部是调用 `doGetConnection()` 方法。

以上就是如何获取一个连接的过程。然后会进行我们的读写操作，之后再释放连接，让我们看看是如何释放连接的吧。
释放连接调用的是 `RedisConnectionUtils.releaseConnection` 方法。

```Java
public static void releaseConnection(@Nullable RedisConnection conn, RedisConnectionFactory factory,
			boolean transactionSupport) {

		if (conn == null) {
			return;
		}

    // 获取与当前线程绑定的连接
		RedisConnectionHolder connHolder = (RedisConnectionHolder) TransactionSynchronizationManager.getResource(factory);

    // 没有使用 @Transactional 注解，所以 connHolder.isTransactionSynchronisationActive() 返回 false。
		if (connHolder != null && connHolder.isTransactionSyncronisationActive()) {
			if (log.isDebugEnabled()) {
				log.debug("Redis Connection will be closed when transaction finished.");
			}
			return;
		}

    // 判断是否和当前线程绑定的连接是同一个连接
		if (isConnectionTransactional(conn, factory)) {

			// release transactional/read-only and non-transactional/non-bound connections.
			// transactional connections for read-only transactions get no synchronizer registered
            // 没有使用 @Transactional 注解，这个判断没过
			if (transactionSupport && TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
				if (log.isDebugEnabled()) {
					log.debug("Unbinding Redis Connection.");
				}
				unbindConnection(factory);
			} else {

				// Not participating in transaction management.
				// Connection could have been attached via session callback.
				if (log.isDebugEnabled()) {
					log.debug("Leaving bound Redis Connection attached.");
				}
			}

		} else {
			if (log.isDebugEnabled()) {
				log.debug("Closing Redis Connection.");
			}
			conn.close();
		}
	}
```

可以看到，当 RedisTemplate 打开事务支持（enableTransactionSupport=true）的时候，执行 Redis 操作的方法如果没有添加 @Transactional 则连接不会被释放，一直绑定到到当前线程
