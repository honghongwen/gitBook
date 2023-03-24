# Spring Transaction注解

### 1. 四特性
##### A
> Atomicity，事务不可分割，转账时：A扣款B收款，这个操作要视为一个操作，最小操作单元。

##### C
> Consistency，数据要保持一致性。200元的转账，A少了200，那B一定是多了200。

##### I
> Isolation，不应受到其他事务干扰，不能受到A转给C的干扰，比如不能说A转给C100，但是A账户还从500剩400。应该先减去B的200，再减去C的100。还剩200。

##### D
> Durability，一旦结束，就应该持久到数据库。



#### 2. 隔离级别
##### RU
> Read Uncommitted，未提交读，会引起脏读。指能读到别的事务未提交的数据。比如A事务修改但是最后回滚，但是B事务读取到了修改的数据。(造成脏读)

![image.png](https://cdn.nlark.com/yuque/0/2022/png/454950/1665651010541-105bfe42-a921-4467-8aa1-36239f982254.png#clientId=u280eeff6-497b-4&from=paste&height=454&id=u5110cb80&name=image.png&originHeight=454&originWidth=862&originalType=binary&ratio=1&rotation=0&showTitle=false&size=20105&status=done&style=none&taskId=u82e9c3d1-9af6-4508-8d98-25d8f4060bb&title=&width=862)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/454950/1665651028257-a35be5fd-d2bd-4acb-8a32-342d180c562d.png#clientId=u280eeff6-497b-4&from=paste&height=441&id=ufd2340b9&name=image.png&originHeight=441&originWidth=892&originalType=binary&ratio=1&rotation=0&showTitle=false&size=21354&status=done&style=none&taskId=u0b275e9b-6e4d-45e7-a775-f6ad4be6c9c&title=&width=892)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/454950/1665651061173-50b448fd-f09c-402a-be93-2b1b278f0745.png#clientId=u280eeff6-497b-4&from=paste&height=277&id=u4a7b3efa&name=image.png&originHeight=277&originWidth=576&originalType=binary&ratio=1&rotation=0&showTitle=false&size=15128&status=done&style=none&taskId=u3ee98f7a-1ed8-4c74-8764-5f483e02d1d&title=&width=576)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/454950/1665651072391-a8222290-aad7-49c9-8ba7-2ffb603f0ca5.png#clientId=u280eeff6-497b-4&from=paste&height=247&id=ud3c0fd97&name=image.png&originHeight=247&originWidth=911&originalType=binary&ratio=1&rotation=0&showTitle=false&size=14199&status=done&style=none&taskId=ueaa71d16-265c-44eb-a4b4-24ff541b4d6&title=&width=911)
##### RC
> Read Committed，提交读，oracle的默认级别。指A事务中只要B提交的数据就能被读到，两次结果不同。比如A事务下单失败，因为抢占CPU失败了，进行重试。但是B事务此时把库存扣完了。A从新读取发现库存已经被扣完。或者A发现查询卡里有2000块，进行消费，但是B已经消费完了2000，此时A再次读取发现卡里没钱了，消费失败。（造成不可重复读）

![image.png](https://cdn.nlark.com/yuque/0/2022/png/454950/1665651233939-8c1ae7c7-85c8-467c-b4e3-13248382f768.png#clientId=u280eeff6-497b-4&from=paste&height=359&id=uf990c050&name=image.png&originHeight=359&originWidth=646&originalType=binary&ratio=1&rotation=0&showTitle=false&size=14852&status=done&style=none&taskId=u3d03b01d-a3f5-41f2-b033-1f589255fdd&title=&width=646)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/454950/1665651244901-c9db730f-4cbd-429d-b5e7-a059134449d2.png#clientId=u280eeff6-497b-4&from=paste&height=294&id=u5bccaec9&name=image.png&originHeight=294&originWidth=733&originalType=binary&ratio=1&rotation=0&showTitle=false&size=18082&status=done&style=none&taskId=u3149f877-df24-46a1-8c7d-8ce01bd9df3&title=&width=733)
commit之后
![image.png](https://cdn.nlark.com/yuque/0/2022/png/454950/1665651266457-1f649234-10df-480a-b0e2-53cea2670a15.png#clientId=u280eeff6-497b-4&from=paste&height=293&id=u0a35a0b0&name=image.png&originHeight=293&originWidth=1004&originalType=binary&ratio=1&rotation=0&showTitle=false&size=15791&status=done&style=none&taskId=uf70eb0db-23f5-459d-8a8e-38400baadbe&title=&width=1004)
##### RR
> Repeatable Read，重复读，mysql默认级别。解决了不可重复读。
> 幻读指的是在一个事务内，同一 SELECT 语句在不同时间执行，得到不同的结果集时，就会发生所谓的幻读问题。
> 比如select * from order where amount < 10;

！mysql如何在RR级别解决部分幻读的问题的
> select 分为快照读（MVCC multi version concurrency control）和实时读(innodb)，快照读通过并发多版本控制解决。实时读通过间隙锁解决。


MVCC（multi version concurrency control)是如何解决幻读的，首先，针对每行数据，都会有多个版本，而且每个版本上会有两个隐藏字段，当前版本和删除版本。
对于insert，会将当前版本设置为当前事务id
对于update，会将新行数据的当前版本设置为当前事务id，并且将老行数据的删除版本设置为当前事务id
对于delete，会将当前删除版本设置为当前事务id
对于select，只读取删除版本为空且当前版本小于等于当前事务id的数据
这是种无锁实现。
正常的**SELECT**语句，后面不加**FOR UPDATE**和**LOCK IN SHARE MODE**的，就是用的MVCC去读。
[参考](https://www.yasinshaw.com/articles/125)

当前读的幻读解决办法为间隙锁，锁的大致结构如下:

- 行锁(Next-Key Locks)
   - 记录锁 （找到确切记录）
      - 共享锁(select ... lock in share mode)
      - 排它锁 (select ... for update)
   - 间隙锁（范围内没有确切记录，为左开右闭，[0,100）(100, 正无穷)这种，间隙锁之间共享，不阻塞）
      - 插入意向锁（之间不阻塞，但和间隙锁阻塞）

所以会造成死锁，即两个for update，两个insert，都被对方的间隙锁锁住了。mysql会监控到并回滚其中一个事务。
所有行锁都针对有索引情况，没索引会锁表。
##### S
> Serializable，序列化。每个事务按顺序执行，读写阻塞。写写阻塞。


#### 传播机制

- 支持当前事务
   - propagation_required，当前存在，则使用。否则新建一个。
   - propagation_supports，当前存在，则使用，否则不使用。
   - propagation_mandatory，当前存在，则使用，否则抛异常。
- 不支持当前事务
   - propagation_requires_new，总是新建事务，如果当前有，则将当前的挂起。
   - propagation_not_supports，总是不支持，如果当前有，则将当前的挂起。
   - propagation_never，总是不支持，如果当前有，则抛异常。
- 嵌套
   - propagation_nested，当前存在，则创建一个嵌套事务运行，如果当前没有。相当于required。
- 最常用 三种
   - required，上游回滚，下游也会，同理下游回滚，上游也会
   - requires_new，上游回滚，下游不会。下游抛异常，上游依然回滚。但如果下游单纯回滚，上游也不会被影响。
   - nested，上游回滚，下游也会。下游回滚，上游不会。
