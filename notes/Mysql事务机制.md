# Mysql事务机制测试

## 事务隔离级别

![](.\image\1.png)

事务的隔离性是通过锁来实现，上一篇也提到事务的隔离级别，这篇简单回顾一下。

四种隔离级别，按READ-UNCOMMITTED、READ-COMMITTED、REPEATABLE-READ、SERIALIZABLE顺序，隔离级别是从低到高，InnoDB默认是REPEATABLE-READ级别，此级别在其余数据库中是会引起幻读问题，InnoDB采用Next-Key Lock锁算法避免了此问题，什么是幻读问题，请参考上一篇文章。

隔离级别越低，则事务请求的锁和保持锁的时间就越短。

### READ-UNCOMMITTED

READ-UNCOMMITTED 中文叫未提交读，即一个事务读到了另一个未提交事务修改过的数据，整个过程如下图:



![img](.\image\2.png)



如上图，SessionA和SessionB分别开启一个事务，SessionB中的事务先将id为1的记录的name列更新为'lisi'，然后Session 中的事务再去查询这条id为1的记录，那么在未提交读的隔离级别下，查询结果由'zhangsan'变成了'lisi'，也就是说某个事务读到了另一个未提交事务修改过的记录。但是如果SessionB中的事务稍后进行了回滚，那么SessionA中的事务相当于读到了一个不存在的数据，这种现象也称为脏读。

可见READ-UNCOMMITTED是非常不安全。

### READ COMMITTED

READ COMMITTED 中文叫已提交读，或者叫不可重复读。即一个事务能读到另一个已经提交事务修改后的数据，如果其他事务均对该数据进行修改并提交，该事务也能查询到最新值。如下图:



![img](.\image\3.png)



在第4步 SessionB 修改后，如果未提交，SessionA是读不到，但SessionB一旦提交后，SessionA即可读到SessionB修改的内容。

从某种程度上已提交读是违反事务的隔离性的。

### REPEATABLE READ

REPEATABLE READ 中文叫可重复读，即事务能读到另一个已经提交的事务修改过的数据，但是第一次读过某条记录后，即使后面其他事务修改了该记录的值并且提交，该事务之后再读该条记录时，读到的仍是第一次读到的值，而不是每次都读到不同的数据。如下图:



![img](.\image\4.png)



InnoDB默认是这种隔离级别，SessionB无论怎么修改id=1的值，SessionA读到依然是自己开启事务第一次读到的内容。

### SERIALIZABLE

SERIALIZABLE 叫串行化， 上面三种隔离级别可以进行 读-读 或者 读-写、写-读三种并发操作，而SERIALIZABLE不允许读-写，写-读的并发操作。 如下图:



![img](.\image\5.png)



SessionB 对 id=1 进行修改的时候，SessionA 读取id=1则需要等待 SessionB 提交事务。可以理解SessionB在更新的时候加了X锁。

## 测试环境

- 应用环境：springboot+mybatis
- 数据库：mysql5.7.32
- 事务隔离级别：REPEATABLE-READ （可重复读写）

## 测试内容

### 代码中实际开启事务的位置

测试结果：当我们在事务中使用 **@Transactional** 注解时，表示当前方法以事务的方式执行，但是实际开启事务的位置其实是在**第一条sql语句**执行的时候才开启事务的，而并非在**方法执行**一开始的时候开启事务。

详细测试：

<!--部分逻辑未写-->

```
@Transactional
public PageInfoData<UserQueryData> findUsers(SysUserParams sysUserParams, PageInfoParams pageInfoParams) {
    try {
        Thread.sleep(10000);
    }catch (Exception e){
        e.printStackTrace();
    }
    SysUser sysUser = sysUserMapper.selectById(JwtUtil.getUserId());
    Page<UserQueryData> sysUserPage = sysUserMapper.selectUserPage(sysUserParams, sysUser.getTopGroupId(), pageHelper);
    return new PageInfoData(sysUserPage);
}
```

```
@Transactional
public SysUser updateUser(Long userId, SysUserUpdateParams sysUserUpdateParams) {
    SysUser sysUser = sysUserMapper.selectById(userId);
    //未贴出赋值代码
    sysUserMapper.updateById(sysUser);
    return sysUser;
}
```

我在查询用户的方法中，添加Thread.sleep()，达到并发执行的目的。

首先执行findUsers代码对应的接口，然后执行updateUser的接口，updateUser应该会很快执行完毕，而findUsers会等待很久。当findUsers执行完毕后，我们会在结果中看到，其中包含updateUser修改之后的数据。这表明findUsers的事务是在updateUser之后执行的，所以当方法执行后后事务并没有生效。



我将findUsers的代码修改了一下

```
@Transactional
public PageInfoData<UserQueryData> findUsers(SysUserParams sysUserParams, PageInfoParams pageInfoParams) {
    SysUser sysUser = sysUserMapper.selectById(JwtUtil.getUserId());
    try {
        Thread.sleep(10000);
    }catch (Exception e){
        e.printStackTrace();
    }
    Page<UserQueryData> sysUserPage = sysUserMapper.selectUserPage(sysUserParams, sysUser.getTopGroupId(), pageHelper);
    return new PageInfoData(sysUserPage);
}
```

按照之前的方式执行了代码，结果findUsers只查询到修改之前的数据，并没有查询到修改之后的数据，这表明findUsers与updateUser的事务时并发执行的，所以说实际事务生效是在第一条sql执行后。



部分资料来源：https://juejin.cn/post/6844903827611582471

## 并发控制 与 MVCC

MVCC (multiple-version-concurrency-control）它是个**行级锁**的变种， 在**普通读情况下避免了加锁操作，因此开销更低**。虽然实现不同，但通常都是实现**非阻塞读**，对于**写操作只锁定必要的行。**

https://segmentfault.com/a/1190000018658828