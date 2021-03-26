基于kafka2.1.0源码阅读笔记

生产者：
examples/src/main/java/kafka/examples/Producer.java

1. 初始化kafkaProducer对象

	1.1 设置clientId，分区器，序列化器等  元数据更新时间，消息限制，RecordAccumulator缓存大小，压缩格式等信息

	1.2 创建一个RecordAccumulator

	1.3 初始化拉取元数据

	1.4 初始化一个网络管理组件NetworkClient

	1.5 sender线程启动（重要）

2. 异步发送
	2.1 拦截器处理

	2.2 doSend

		2.2.1 等待拉取元数据

			2.2.1.1 内存中有直接返回

			2.2.1.2 没有则wait等待sender线程拉取后唤醒更新元数据，要么超时

		2.2.2 序列化key value

		2.2.3 根据分区器选择分区

			2.2.1.1 Record中有则返回（可能重试）

			2.2.1.2 （默认分区器）指定key则按key哈希取模，否则取模轮询

		2.2.4 检查消息是否超过限制（单条以及RecordAccumulator大小）

		2.2.5 根据元数据信息，封装分区对象

		2.2.5 为每条消息绑定其回调函数

		2.2.6 往RecordAccumulator添加

			ConcurrentMap<TopicPartition, Deque<RecordBatch>> batches //分区->队列 Kafka封装的CopyOnWriteMap 读写分离 写上锁

			2.2.6.1 根据分区选择到对应的队列

				锁队列，写消息 tryAppend（队列中有批次则尝试往最后一个批次加，无批次或批次空间不够则返回失败）

				（需要新建批次）计算一个批次的大小，并申请内存
						申请内存如果为标准批次大小且内存池的free队列中有则直接取，否则到availableMemory中去申请。内存不够等释放（释放的时候，没人等，标准的批次放队列里，如果有人登或者不标准的拆了放availableMemory中）

				锁队列，写消息（假设并发两人的话都能申请到内存，但上一批次有位置则写上一批次，并还内存，否则写自己申请的批次）（有压缩），批次放队尾

		2.2.7 唤醒sender线程


sender线程 run

1. 获取元数据（内存中获取）

2. 判断哪些partition有消息可以发（partition的leader partition对应的broker主机）

	2.1 遍历batch  ConcurrentMap<TopicPartition, Deque<RecordBatch>>

		leader没有值记录

		队头获取批次，批次写满了，批次数量大于1，时间到了，内存不够，关闭线程 满足其一则加入ready集合

3. 标识没拉到元数据topic，并标记需要更新元数据

4. 遍历就绪节点，检查网络连接是否OK

	4.1 判断该主机是否具备发送消息条件（不是正在更新元数据）

	4.2 判断能否连接（缓存中没有，从未连接过；缓存中有，但重试时间到了）

		4.2.1 初始化连接 尝试连接selector.connect

			4.2.1.1 基于NIO 非阻塞，关闭Nagle算法（收集小包组大包发送）socketChannel往Selector注册OP_CONNECT事件。立刻连接上的做取消注册处理

5. 按照broker进行分组，同一broker的partition同一组（如p0，p2在同一台broker上则合并）

6. 是否有授权

7. 放弃超时的批次

8. 创建发送请求 并 Selector send 

9. poll

	9.1 封装拉取元数据的请求

	9.2 selector.poll()
	
		9.2.1 遍历所有有响应的selectionKeys；根据key找到相应Channel，并根据不同事件处理相应事情

			可连接的
			可写的，写
			可读的，读完后将结果放在待处理集合

		9.2.2 处理待处理结果，放在已处理集合（这不是真处理）

	9.3 处理响应（包含元数据）

		9.3.1 元数据的进行处理

			将二进制流转成结果，更新（版本号这些）唤醒wait的线程

		9.3.2 非原数据的封装后存入List返回

	9.4 处理一些失去连接的，新连接的，超时的等

	9.4 调用回调函数（kafka的回调，里面有用户回调），释放内存，重试等