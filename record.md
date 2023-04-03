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