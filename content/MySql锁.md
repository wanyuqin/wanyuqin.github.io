---
title: Mysql锁
categories: 日常学习
comments: true
---



# MySql锁

## InnoDB中的锁

### 锁的类型

* 共享锁(S Lock)允许事务读一行数据
* 排他锁 (X Lock)允许事务删除或更新一行数据


|      |   X    |   S    |
| :--: | :----: | :----: |
|  X   | 不兼容 | 不兼容 |
|  S   | 不兼容 |  兼容  |

<!-- more -->

### 意向锁

InnoDB支持多粒度的锁定，这种锁定允许事务在行级上的锁和表级上的锁同时存在，为了支持不同粒度上进行加锁，InnoDB支持一种额外的锁的方式，**意向锁**。意向锁会把锁定的对象分成多个层次，意向锁意味着事务希望在更细粒度上进行加锁。

<img src="https://img.ethanleo.top/uPic/image-20220812204346575.png" alt="image-20220812204346575" style="zoom:50%;" />

把上锁对象看成树，如果要对页上的记录r进行上X锁，那么分别需要对数据库A、表、页上意向锁IX，最后记录r上X锁，如果其中任何一个部分导致等待，那么该操作需要等待粗粒度锁的完成。比如说对r记录加x锁之前，已经有一个事务对表1进行了S表锁，那么之后的事务要对记录r在表1上加上IX，必须要等待表锁操作完成



* 意向共享锁(IS Lock)，事务想要获得一张表里的某几行的共享锁
* 意向排他锁(IX Lock)，事务要想获得一张表里的某几行的排他锁

|      |   IS   |   IX   |   S    |   X    |
| :--: | :----: | :----: | :----: | :----: |
|  IS  |  兼容  |  兼容  |  兼容  | 不兼容 |
|  IX  |  兼容  |  兼容  | 不兼容 | 不兼容 |
|  S   |  兼容  | 不兼容 |  兼容  | 不兼容 |
|  X   | 不兼容 | 不兼容 | 不兼容 | 不兼容 |





### 一致性非锁定读

一致性非锁定读是指InnoDB通过行多版本控制的方式来读取当前执行时间数据库中的行数据。如果读取的行数据正在进行DELETE或UPDATE操作，这个时候读取操作不会等待行上锁的释放。相反的InnoDB会去读取行数据的一个快照数据。快照数据是指，改行之前版本的数据，一个行的记录可能会存在多个版本的快照数据，这样的多版本技术就是MVCC(Multi Version Concurrency Control)是通过undo段来完成的，而undo段数据使用来进行事务回滚的,所以不会有额外的开销。

<img src="https://img.ethanleo.top/uPic/image-20220813140223598.png" alt="image-20220813140223598" style="zoom:50%;" />

在事务隔离级别READ COMMITED和REPETABLE READ中，Innodb采用一致性非锁定读，但是在这两个隔离级别下，读取的数据版本并不是一样的。

* REPETABLE READ 读取事务开始时的行数据版本
* READ COMMITED 读取行的最新版本，如果行被锁定了，则读取行的最新版本的快照

<img src="https://img.ethanleo.top/uPic/image-20220813142120309.png" alt="image-20220813142120309" style="zoom:50%;" />



## 一致性锁读

默认情况下InnoDB对于REPERABLE READ的隔离级别下时一致性非锁读，但是有很多时候，我们需要对数据进行显式的加锁，以保证数据逻辑的一致性，对于SELECT的读操作，MySQL也支持两种加锁的方式

* SELECT ..... FOR UPDATE
* SELECT ..... LOCK IN SHARE MODE

FOR UPDATE的操作会个数据加上一个X锁，这个时候其他事务不能对锁定的数据再次加锁，加锁的操作会被阻塞，而LOCK IN SHARE MODE是给数据加上s锁，其他事务可以对数据加上s锁，但是要给数据加X锁时，一样会被阻塞。对于这两种加锁的方式，会在事务提交之后将锁释放，所以要使用这两种方式加锁，务必加上BEGIN、START TRANCTION或SET AUTOCOMMIT=0



## 锁的算法

InnoDB有三种锁的算法

* Record Lock 单行记录上的锁
* Gap Lock 间隙锁，锁定一个范围，但是不包含记录本身
* Next-key Lock  Gap Lock+Record Lock 锁定一个范围，并锁定记录本身

Record Lock 总是会去锁定索引记录，如果InnoDB在建立表的时候没有任何索引，那么它会隐式的使用主键来锁定。Next-key Lock 是结合了Record Lock和Gap Lock的一种算法，在Next-Key Lock算法下，InnoDB对于行的查询都是用的这种算法。假如有一个索引是10，11，13和20的值，那么可能锁定的范围有：

```
(-∞,10]
(10,11]
(11,13]
(13,20]
(20,+∞)
```

假如事务T1通过Next-Key Lock锁定了范围是

(10,11]、 (11,13]

当插入新的记录12的时候，锁的范围就变成

(10,11]、(11,12]、(12,13]

然而，当查询的索引含有唯一性的时候，InnoDb 会进行优化，将next-key Lock降级为Record Lock，即锁住索引本身而不是一个范围。但是当查询的是一个辅助索引的时候，还是会对范围进行锁定

```sql
CREATE TABLE t2  (a INT,b INT,PRIMARY KEY(a),KEY(b));
INSERT INTO t2 SELECT 1,1;
INSERT INTO t2 SELECT 3,1;
INSERT INTO t2 SELECT 5,3;
INSERT INTO t2 SELECT 7,6;
INSERT INTO t2 SELECT 10,8;
```

表t2的字段b作为辅助索引，如果此时在session1中执行：

```sql
BEGIN;
SELECT * FROM t2 WHERE b =3 FOR UPDATE;
```

这个时候就会采用next-key Lock 进行锁定，而且由于有两个索引的存在，就需要分别进行锁定。对于聚集索引，仅对a=5进行Record Lock，对于辅助索引，加上的是Next-key Lock，锁定的范围是(1，3]，于此同时，innodb还会对辅助索引下一个键值进行加上Gap Lock

也就是说还有一个(3,6)的范围的锁，若在新的会话中执行下列语句，都会被阻塞

```sql
BEGIN;
SELECT * FROM t2 where a=5 LOCK IN SHARE MODE;
INSERT INTO t2 SELECT 4,2;
INSERT INTO t2 SELECT 6,5;
```

第一个语句，由于a=5的记录已经加上了X锁，所以需要等待。第二句，因为插入的b=2在next-key Lock(1,3]的范围内，也需要等待

，第三局，虽然a=6和b=5不在record Lock 和 next-key Lock的范围内，但是b=5在gap Lock的范围内，所以依旧需要等待。

回话session1已经锁定了b=3的记录，假如gap Lock没有锁定(3,6)的范围，那么用户可以继续插入b=3，这个时候，会出现幻读的情况，导致session1两次查询得到的结果不一样。

当然，我们也可以显示的关闭gap Lock

* 将事务级别设置为READ COMMITED
* 将参数 innodb_lock_unsafe_for_binlog设置为1 

在innodb中,执行INSERT操作也会检查记录的下一条记录是否被锁定，如果被锁定则不会允许查询。对于上面的例子，由于会话1已经锁定了b=3的记录，即已经锁定了(1,3]的范围，那么在执行下列语句的时候就会被阻塞

```sql
INSERT INTO t2 SELECT 2,2
```

### 解决幻读问题

幻读是指在同一事务下，连续执行两次SQL可能导致结果不同，第二次的SQL结果可能会返回之前不存在的行。在默认的事务级别REPETABLE READ下，InnoDB会采用next-key lock来解决这样的问题

## 锁问题

### 脏读

脏数据是指未提交的数据，如果一个事务读取到了另一个未提交的事务的数据。脏读就是可以读取到脏数据，也就是一个事务可以读取到另一个未提交事务的数据，这样就违反了事务的隔离性

| TIME | 会话A                   | 会话B                              |
| :--: | :---------------------- | :--------------------------------- |
|  1   |                         | BEGIN;                             |
|  2   | BEGIN                   | SELECT * FROM t;<br />a:1          |
|  3   | INSERT INTO t SELECT 2; |                                    |
|  4   |                         | SELECT * FROM t;<br />a:1<br />a:2 |



### 不可重复读

不可重复读是指，一个事务中多次读取同一数据集合，在这个事务还没有结束的时候，另一个事务也对此访问了同样的数据集合，并且进行了一些DML操作，那么第一个事务两次读到的数据可能是不一样的，这样的情况就是不可重复读。他与脏读读区别在于，脏读读取到了另一个事务未提交的数据，不可重复读是读到了另一个已经提交了读数据。这样的话就违反了事务的一致性。

| TIME | 会话A                             | 会话B                   |
| :--: | --------------------------------- | ----------------------- |
|  1   | BEGIN;                            | BEGIN;                  |
|  2   | SELECT * FROM T<br />a:1          |                         |
|  3   |                                   | INSERT INTO t SELECT 2; |
|  4   |                                   | COMMIT;                 |
|  5   | SELECT * FROM t<br />a:1<br />a:2 |                         |

一般来说，不可重复读是可以接受的，因为读取到的是已经提交的数据。在InnoDB中，通过使用Next-key Lock来解决不可重复读的现象，在Next-Key Lock的算法下，锁住的不仅是扫描到的索引，而且还会锁住索引覆盖的范围(gap),在这个范围内的插入都是不允许的，因此就避免了不可重复读的情况。

### 丢失更新

丢失更新简单来说就是一个事务的更新会被另一个事务的更新给覆盖。从而导致数据的不一致。

1. 事务1将记录r更新为v1，事务1未提交
2. 事务2将记录r更新为v2，事务2未提交
3. 事务1提交
4. 事务2提交

但是目前的数据库隔离级别都不会导致这样的情况发生，因为对数据进行更新需要加锁，所以上述的例子中事务2的操作会被阻塞

## 阻塞

以为锁之间不同的兼容关系，会出现一个事务需要等待另一个事务的释放其占用的资源，这就是阻塞。阻塞在一定程度上保证了数据的安全性，可以保证事务并发且正常运行。

在innodb中可以通过设置innodb_lock_wait_timeout来设置锁的超时等待时间，默认是50秒。可以在MySql运行的过程中设置。

`SET @@innodb_lock_wait_timeout=60;` innodb_rollback_on_timeout用来设置等待锁超时时，是否对数据进行回滚，默认是off，表示不回滚，该参数是静态的，不能在Mysql运行的时候进设置。

## 死锁

死锁是指两个或两个以上的事务，因争夺锁资源而造成的一种相互等待的情况。最好的解决方法就是设置锁等待超时时间，在超时之后，事务进行回滚，并且重新开启事务。

### 死锁的式例

**示例一**

A等待B，B等待A

| 时间 | A                                    | B                                                            |
| :--: | ------------------------------------ | ------------------------------------------------------------ |
|  1   | BEGIN;                               |                                                              |
|  2   | SELECT * FROM t WHERE a=1 FOR UPDATE | BEGIN;                                                       |
|  3   |                                      | SELECT * FROM t WHERE a=2 FOR UPDATE;                        |
|  4   | SELECT * FROM t WHERE a=2 FOR UPDATE |                                                              |
|  5   |                                      | SELECT * FROM t WHERE a=1 FOR UPDATE;<br />Dead Lock when trying to get lock;try restarting transction |

**示例二**

当前记录持有了带插入记录的下一个记录的X锁，但是在等待队列中存在一个S锁的请求，可能会导致死锁

```sql
CREATE TABLE t3 (a INT PRIMARY KEY );
INSERT INTO t3 SELECT 1;
INSERT INTO t3 SELECT 2;
INSERT INTO t3 SELECT 4;
INSERT INTO t3 SELECT 5;
```

| TIME | A                                                            | B                                                |
| ---- | ------------------------------------------------------------ | ------------------------------------------------ |
| 1    | BEGIN;                                                       |                                                  |
| 2    |                                                              | BEGIN;                                           |
| 3    | SELECT * FROM t3 where a =4 FOR UPDATE;                      |                                                  |
| 4    |                                                              | SELECT * FROM t3 WHERE a <=4 LOCK IN SHARE MODE; |
| 5    | INSERT INTO t3 SELECT 3;<br />1213 - Deadlock found when trying to get lock; try restarting transaction, Time: 0.001000s |                                                  |
| 6    |                                                              | 事务正常执行                                     |

问题的产生是由于B在请求记录4的S锁时发生了等待，但在之前请求1，2的锁都已经成功了，若能插入a=3的记录，那么会话B在获取记录4之后还得向后获取记录3的记录，这样并不是很合理，因此InnoDB在这里选择了死锁。