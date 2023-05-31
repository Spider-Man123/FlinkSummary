# FlinkSummary
# 状态总结
1. Flink中算子根据状态区分
  1. 无状态算子：
    一般对输入数据进行转换操作后输出，计算时候不依赖其他数据，算子有：map,filter,flatmap算子
  2. 有状态算子：
    不仅需要目前输入的数据，还会依赖之前的数据，例如sum()算子，会维护一个状态记录了之前数据的sum结果，window算子会维护一个状态记录之前的数据。还有事件模式，例如定义了一个模式，需要知道这个用户是否“先下单，后支付”那么就需要使用状态保存下单的操作，同样属于状态。聚合算子和窗口算子都属于有状态的算子。
2. 状态的种类型
  1. 托管状态
  统一由flink管理的状态，状态的故障恢复，存储访问，重组等都是flink实现的，有valuestate, liststate,mapstate
  2. 自定义状态
     一般不使用，全部自己定义
3. 托管状态中的状态分类
  1. 算子状态
    1. 不同 key 的数据只要被分发到同一个并行子任务， 就会访问到同一个 Operator State。 一般用在source和sink等与外部连接的算子上。例如：flink和kafka连接会根据并行度设置对应topic的偏移量状态，保证精确一次消费
    2. 数据结构有liststate,unionstate,广播状态
    3. 广播状态就是将数据复制到每个分区，如果并行度变化，那么仅需要再复制一份就行。底层需要以key-value形式存储，所以要传入mapState. 广播变量时发送给Taskmanager的内存中，广播变量不应该太大，将数据广播后，不同的Task都可以在节点上获取到，每个节点只存一份。
    4. 如果不进行广播，每一个Task都会拷贝一份数据集，造成内存资源浪费
  2. 按键分区状态
    1. 使用在keyBy之后，针对于keyedStream,根据不同的key维护不同的状态
    2. 只要有富函数的算子都可以使用，这样map,flatmap,filter也可以使用。
    3. 底层，是一个key-value的形式进行存储，key就是按照key分区后的key, 相同的key能取到同一个状态，将不同key的状态进行隔离。
    4. 涉及并行度变化后的重组问题，会把不同key的的状态组成键组，然后会根据并行度平均分配
4. 状态的定义和使用
  1. 一般在富函数中使用状态的话，由于富函数中对应的实现函数是一条数据调用一次，例如RichFlatmap函数中的flatmap是来一条数据调用一次，因此状态一般声明在生命周期方法open()中
  2. 定义的话需要先在函数内，方法外定义出来。
  3. 注册的话是根据状态描述器，传入名称和类型
5. 状态的存储
  1. 内存型
    memoryStatebackend, 在运行的时候将所需要的state信息存储在taskmanager的jvm的堆内存中，当执行检查点操作时，会将state信息存储到JobManage的内存中，受限于jobManage内存的大小，容易丢失。
  2. 文件型
    fsStatebackend, 在运行的时候将所需要的state信息存储在taskmanage的内存中，当执行检查点操作时，会将state信息存储到指定的文件系统中。
  3. RocksDbStatebackend
    使用嵌入式的本地数据库 RocksDB 将流计算数据状态存储在本地磁盘中，不会受限于TaskManager 的内存大小，在执行检查点的时候，再将整个 RocksDB 中保存的State数据全量或者增量持久化到配置的文件系统中，在 JobManager 内存中会存储少量的检查点元数据。RocksDB克服了State受内存限制的问题，同时又能够持久化到远端文件系统中，比较适合在生产中使用。
6. 状态持久化策略
  1. 全量持久化策略
    每次将全量的State写入到状态存储中（HDFS）。内存型、文件型、RocksDB类型的StataBackend 都支持全量持久化策略。
  2. 增量持久化策略
    增量持久化就是每次持久化增量的State，只有RocksDBStateBackend 支持增量持久化。
7. 状态过期清理策略
  1. DataStream中设置过期时间，State可见性策略
  2. FlinkSql中设置过期时间
# 检查点总结
1. Flink可靠性容错机制的基础
  Flink使用轻量级分布式快照，通过设置检查点来实现可靠的容错机制
2. checkpoint检查点介绍
  checkpoint是根据配置周期性的基于Stream中各个操作的状态保存的snapshot快照，从而将这些状态定期性的存储下来，当遇到故障崩溃的时候，可以有选择的从这些快照中重启任务，从而修正由于故障带来的状态中断
3. checkpoint和state的区别
  1. state一般是对某个具体的task的状态，checkpoint是把所有的task的状态进行存储，进行全局快照
  2. state一般存储在内存中，checkpoint存储在文件系统中
4. savepoint保存点
  基于检查点机制保存的完整快照，用来保存状态，一般由用户手动创建和删除。可以用来手动重启flink的任务。
5. checkpoint和savepoint的区别
  1. checkpoint重点用于自动容错，savepoint用于程序的更新和修改后从状态中恢复
  2. checkpoint是Flink自动触发，savepoint是用户主动触发
  3. checkpoin一般会自动删除，savepoint一般会保留，触发用户手动删除
6. 当作业失败后，检查点如何恢复作业？
  Flink提供了两种恢复作业的方式
  1. 应用自动恢复机制
    1. 定期恢复策略：job失败会重启，超过重启次数，会认定为失败，每两次重启具有一定的间隔时间
    2. 失败率恢复策略：job失败会重启，超过失败率，认定失败
    3. 直接失败策略：job失败不重启
  2. 手动恢复作业机制
    使用checkpoint恢复
7. 作业变更后，使用savepoint恢复的情况
  1. 算子的顺序改变：如果UID没变，那么就能恢复，如果对应的UID变了就会恢复失败
  2. 作业中添加了新的算子：如果是无状态算子，可以恢复，有状态算子，恢复失败
  3. 删除一个有状态算子恢复会报错，可以设置跳过
  4. 添加或者删除无状态算子：手动设置的UID可以恢复，自动设置的可能会失败因为UID是递增生成的。
# Flink的Exactly-once语义
1. 过程
  1. 当checkpoint启动时，进入预提交状态，jobmanager会向source插入barrier
  2. 当source收到barrier的时候，会先保存自己的状态，也就是消费的offset，然后发给下一个Operator。
  3. 后面的算子当收到barrier的时候同样保存自己的状态，然后发送barrier给下游
  4. 从 Source 端开始，每个内部的 transformation 任务遇到 checkpoint barrier（检查点分界线）时，都会把状态存到 Checkpoint 里。数据处理完毕到 Sink 端时，Sink 任务首先把数据写入外部 Kafka，这些数据都属于预提交的事务（还不能被消费），此时的 Pre-commit 预提交阶段下Data Sink 在保存状态到状态后端的同时还必须预提交它的外部事务。
  5. 当所有算子任务的快照完成（所有创建的快照都被视为是 Checkpoint 的一部分），也就是这次的 Checkpoint 完成时，JobManager 会向所有任务发通知，确认这次 Checkpoint 完成，此时 Pre-commit 预提交阶段才算完成。才正式到两阶段提交协议的第二个阶段：commit 阶段。
  6. 其中任意一个pre-commi失败，就会回滚到最近成功的checkpoint处。
2. 注意事项
  1. 所有的operator完成各自的pre-commit，它们会发起一个commit操作
  2. 如果其中任意一个pre-commit失败，所有其他的pre-commit必须停止，并且Flink会会滚到最近成功的Checkpoint.
# Flink调优

## 资源配置调优
  1. 最优并行度设置
    1. source端：调整并行度等于数据源的分区数，例如kafka的topic分区数。如果相等了，消费速度还是跟不是数据产生速度，那么就考虑调大kafka的topic的分区数，并且调整并行度和其相等。不要让flink的并行度大于kafka的topic的分区数，这样会产生有的并行度空闲，浪费资源。
    2. Transform端：第一次keyBy之前的算子，例如map,filter,flatmap处理速度快的算子，和source保持一致就行。第一次keyBy之后的算子，可以根据压测的方法来根据keyBy算子上游的数据发送量和该算子处理数据的能力得到最优并行度。总QPS/单个并行度处理能力=并行度，最后再乘1.2，留一些富余的资源。压测的方式很简单，先在kafka中积压数据，之后开启Flink任务，出现反压，就是处理瓶颈。相当于水库先积水，一下子泄洪。数据可以是自己造的模拟数据，也可以是生产中的部分数据。
    3. sink端：sink端是数据流向下游的地方，可以根据sink端的数据量及下游的服务抗压能力进行评估。如果sink端是kafka，可以设为kafka对应topic的分区数。
  2. Checkpoint的设置
    1. 设置checkpoint的间隔时间，一般设置为分钟级，如果状态大的任务设置时间为5到10分钟
    2. 调整两次checkpoint的间隔时间，这个参数的含义是：例如checkpoint间隔时间设置为1分钟，如果checkpoint耗时40s,那么20s后就会进行下一次checkpoint，但是如果设置了两次checkpoint的间隔时间为30s, 那么不会20s之后执行，会30s之后执行。
    3. 设置 checkpoint 语义，可以设置为 EXACTLY_ONCE，表示既不重复消费也不丢数据AT_LEAST_ONCE，表示至少消费一次，可能会重复消费
    4. 设置checkpoint的超时时间，默认为10分钟，可以设置设置小一些
    5. 允许失败的次数
## 反压处理
  1. 产生场景
    通过在短时间内的负载高峰导致系统的接收数据的速率远高于它处理数据的速度。在垃圾回收的时间内消息快速的堆积。在大促，节日等场景下数据量会急速增大。如果反压不及时处理会导致资源耗尽和系统崩溃。
  2. 反压机制
  反压机制是指系统能自动检测出数据阻塞的Operator, 然后自适应的数据源发送数据的速率，从而维持整个系统的稳定。
  3. 检测方法
    1. 使用web UI的Job中的backpressure页面，其中会进行反压测试，会采样判断卡在申请缓存的次数，来确定是否出现反压，根据比例。
    2. 使用Metrics对task的inputChannel进行监控，判断是否满了。
  
## 数据倾斜
  1. 检测是否出现数据倾斜
  每个task会有很多subtask,通过Flink的Web UI 界面可以很准确的检测到分发给每个subtask的数据量，这样          就可以判断出哪个subtask出现了数据倾斜，通常数据倾斜也会引起反压。
  2. keyBy后的聚合操作存在数据倾斜
    使用两阶段聚合，在keyBy之前使用flatmap或者其他转换算子进行预聚合，然后发送到下游
  3. keyBy前出现数据倾斜
    一般是原本的数据存在数据不均匀的情况，例如Kafka的分区数据量不同，这种情况需要让 Flink 任务强制进行shuffle。使用shuffle、rebalance 或 rescale算子即可将数据均匀分配，从而解决数据倾斜的问题。
  4. keyBy之后的窗口聚合操作存在数据倾斜
    添加随机数前缀或者后缀的方法，首先给数据key加上随机数前缀，然后进行keyBy，开窗聚合.最后去掉前缀或者后缀，进行keyBy然后聚合。
  Flink的特点
  1. 高吞吐，低延迟，每秒可以处理百万级数据，可以达到毫秒级延迟
  2. 保证结果的准确性，Flink提供了事件事件和处理事件语义，使用watermark机制使乱序流和基于事件时间都可以保证准确性结果。
  3. 使用checkpoint和state来保证了精确一次。
  Flink vs Spark
  1. Spark以批处理为根本，离线的是一个大批次，实时数据就是一个一个无限的小批次。因此SparkStreaming并不是真正意义上的流，而是微批次处理。
  Flink是基于流处理为基础，认为实时数据是标准的没有界限的流
  2. Spark的底层数据模型是RDD，SparkStreaming底层接口Dstrean, Dstream实际上也就是一组组小批次的RDD集合，在计算的时候将DAG图根据宽窄依赖划分为不同的Stage，然后根据分区数划分为不同的任务进行执行。
    Flink基本数据模型是数据流DataFlow, 以事件（Event) 为序列。标准的流式处理流程，一个节点处理完就发送到下一个节点进行处理。
