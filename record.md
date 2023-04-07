## 杂项
- svn账：putianyang 密：^@bB2Fxf

## mysql相关
- update涉及到两次日志记录，一次是server层的binlog，另一次是引擎层的redo-log；其中redo-log的记录涉及到两阶段提交：prepare和commit，主要为了避免和binlog的不一致
- innodb事务的可重复读涉及undo-log的回滚操作，尽量避免开启长事务
- hash索引适用于等值查询，范围查询因为无序涉及全表遍历
- 主键长度越小越好，因为二级索引会更小；只有一个索引的表可以设为主键，其余一般为自增主键
- mysql5.6以后不符合最左匹配原则的部分会采用索引下推优化减少回表次数
- 创建索引时的注意点：
  1. 非空字段：应该指定列为NOTNULL，除非你想存储NULL。在mysql中，含有空值的列很难进行查询优化，因为它们使得索引、索引的统计信息以及比较运算更加复杂。你应该用0、一个特殊的值或者一个空串代替空值
  2. 取值离散大的字段：（变量各个取值之间的差异程度）的列放到联合索引的前面，可以通过count()函数查看字段的差异值，返回值越大说明字段的唯一值越多字段的离散程度高
  3. 索引字段越小越好：数据库的数据存储以页为单位一页存储的数据越多一次IO操作获取的数据越大效率越高

## redis相关
- 分布式锁
```
# 原子性的设置超时
set lock:xx true ex 5
process....
del lock:xx
# 一般不用于执行长时间的任务，以免超过超时时间
# 使用带tag的加锁释放锁，释放锁时需要使用lua保证原子性
```

## 引擎相关
- LogicCLassModule加载logical_class.xml构建逻辑对象
- 玩家进入场景时序
![](/pic/player_1.jpg)


## AI模块
- bevTreeId关联NPC对象和对应一套行为树json文件，AIModule加载时读取对应所有的json文件并转换为各个节点
- AI转换目标的类型
![](/pic/ai_target_type.jpg)


```c++
1. npc--->onEntry()--->检查场景是否开启AI--->createNpcAI()
```

## 战斗和副本相关
- 造成伤害的方式：最终都调用`DoDamage`方法
    + 直接伤害(DamageTarget) 
    + 按攻击公式计算后造成伤害(DamageTargetWithAttack) 
    + 分担伤害(ShareDamage)
```c++
// 伤害事件
FightModule::DoDamage()=>FightModule::NoticeDamage()=>CloneSceneModule::OnCommandDamageTarget()
//
```
## C++
- __thread只支持POD对象，且只能静态初始化；pthread_key_t可以存储复杂对象

## 多线程相关
- 如果需要用到多个锁，保证按照同一顺序添加以免死锁；如果需要锁住相同类型的多个对象，为了保证始终按照同一顺序加锁，可以比较mutex地址
- 作为成员的mutex不能保护析构函数，构造函数中不应该传出this指针
- 线程安全的observer模式：使用智能指针`shared_ptr`和`weak_ptr`而不是加锁；`weak_ptr`的提升操作是线程安全的；但`shared_ptr`只是保证了引用计数的线程安全而不会保证智能指针本身读写的安全，只能由多个线程读，但不能多个线程同时读写
- 单线程eventloop有一个明显的缺点，对优先级不敏感，优先级高的连接不能得到及时处理；对于这种情况可以采用多线程eventloop来避免
- 多线程对于IO或者计算很容易到达瓶颈的场景无能为力，在这两种情形下，选择单线程
- 防止死锁

![](/pic/prevent_dead_lock.png)

## 设计模式
- 组合、关联和聚合: 组合表示部分与整体，部分的生命周期归整体管理；关联表示一种使用，形式上看A持有B的指针或者引用；聚合也是部分与整体，不过部分的生命周期不由整体控制

## 日志模块
- AsyncLogging类是异步日志系统的核心，它负责将日志信息写入磁盘文件。AsyncLogging类的构造函数中，接收三个参数：basename表示日志文件的基础名称，rollSize表示日志文件达到一定大小时需要切换到新的日志文件，flushInterval表示日志信息的刷新间隔。在构造函数中，会初始化一些成员变量，包括flushInterval_、running_、basename_、rollSize_、thread_、latch_、currentBuffer_、nextBuffer_和buffers_。其中，buffers_是一个BufferVector类型的容器，用于存储待写入磁盘文件的缓冲区。
- AsyncLogging类中的append函数用于向异步日志系统中添加日志信息。每次添加日志信息时，会先判断当前缓冲区是否有足够的空间存储该日志信息，如果有，则直接将日志信息添加到当前缓冲区中；否则，将当前缓冲区存储到buffers_容器中，并将nextBuffer_指向当前缓冲区，然后创建一个新的缓冲区currentBuffer_，将日志信息添加到该缓冲区中，最后唤醒异步日志系统的线程。
- AsyncLogging类中的threadFunc函数是异步日志系统的线程函数。在该函数中，会先初始化一些成员变量，包括newBuffer1、newBuffer2和buffersToWrite。newBuffer1和newBuffer2分别表示两个缓冲区，用于轮流存储日志信息；buffersToWrite是一个BufferVector类型的容器，用于存储待写入磁盘文件的缓冲区。
- 在threadFunc函数的主循环中，会先判断buffers_容器是否为空，如果为空，则等待一段时间后再次检查；否则，将当前缓冲区currentBuffer_存储到buffers_容器中，并将newBuffer1指向当前缓冲区，同时将buffers_容器中的缓冲区存储到buffersToWrite容器中，并将nextBuffer_指向newBuffer2。接着，遍历buffersToWrite容器中的缓冲区，将缓冲区中的日志信息写入磁盘文件中，如果缓冲区的数量超过了25个，则只保留前两个缓冲区，并输出一条日志信息表示有日志信息被丢弃。最后，将newBuffer1和newBuffer2指向buffersToWrite容器中的两个缓冲区，清空buffersToWrite容器，并将磁盘文件刷新到磁盘中。
- LogFile负责滚动日志文件，滚动时机包括当前文件已写入的字节数以及append调用次数。每次append会检测当前调用次数累计是否达到设置的阈值，并且如果达到下个时间区域（将time函数的返回分割成均等的时间区域）就会滚动文件；如果还未达到下一个时间区域，则flush一次。