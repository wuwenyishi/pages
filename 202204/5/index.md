# MySQL 为什么需要两阶段提交？

<iframe align="center" width="100%" height="2000"  src="https://xuemingde.com/JavaNotes/index.html"  frameborder="no" border="0" marginwidth="0"  marginheight="0" scrolling="no"></iframe>





## 1. 什么是两阶段提交

### 1.1 binlog 与 redolog

#### binlog

binlog 我们中文一般称作归档日志，如果大家看过松哥之前发的 MySQL 主从搭建，应该对这个日志有印象，当我们搭建 MySQL 主从的时候就离不开 binlog（传送门：[MySQL8 主从复制踩坑指南](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2Fy8PR1emyBjvk124b5ByjXg)）。

binlog 是 MySQL Server 层的日志，而不是存储引擎自带的日志，它记录了所有的 DDL 和 DML(不包含数据查询语句)语句，而且是以事件形式记录，还包含语句所执行的消耗的时间等，需要注意的是：

- binlog 是一种逻辑日志，他里边所记录的是一条 SQL 语句的原始逻辑，例如给某一个字段 +1，注意这个区别于 redo log 的物理日志（在某个数据页上做了什么修改）。
- binlog 文件写满后，会自动切换到下一个日志文件继续写，而不会覆盖以前的日志，这个也区别于 redo log，redo log 是循环写入的，即后面写入的可能会覆盖前面写入的。
- 一般来说，我们在配置 binlog 的时候，可以指定 binlog 文件的有效期，这样在到期后，日志文件会自动删除，这样避免占用较多存储空间。

根据 MySQL 官方文档的介绍，开启 binlog 之后，大概会有 1% 的性能损耗，不过这还是可以接受的，一般来说，binlog 有两个重要的使用场景：

- MySQL 主从复制时：在主机上开启 binlog，主机将 binlog 同步给从机，从机通过 binlog 来同步数据，进而实现主机和从机的数据同步。
- MySQL 数据恢复，通过使用 mysqlbinlog 工具再结合 binlog 文件，可以将数据恢复到过去的某一时刻。

#### redo log

前面我们说的 binlog 是 MySQL 自己提供的，在 MySQL 的 server 层，而 redo log 则不是 MySQL 提供的，是存储引擎 InnoDB 自己提供的。所以在 MySQL 中就存在两类日志 binlog 和 redo log，存在两类日志既有历史原因（InnoDB 最早不是 MySQL 官方存储引擎）也有技术原因，这个咱们以后再细聊。

我们都知道，事务的四大特性里面有一个是持久性，即只要事务提交成功，那么对数据库做的修改就被永久保存下来了，写到磁盘中了，怎么做到的呢？其实我们很容易想到是在每次事务提交的时候，将该事务涉及修改的数据页全部刷新到磁盘中，一旦写到磁盘中，就不怕数据丢失了。

但是要是每次都这么搞，数据库就不知道慢到哪里去了！因为 Innodb 是以页为单位进行磁盘交互的，而一个事务很可能只修改一个数据页里面的几个字节，这个时候将完整的数据页刷到磁盘的话，不仅效率低，也浪费资源。效率低是因为这些数据页在物理上并不连续，将数据页刷到磁盘会涉及到随机 IO。

有鉴于此，MySQL 设计了 redo log，在 redo log 中只记录事务对数据页做了哪些修改。那有人说，写 redo log 不就是磁盘 IO 吗？而写数据到磁盘也是磁盘 IO，既然都是磁盘 IO，那干嘛不把直接把数据写到磁盘呢？还费这事！

此言差矣。

写 redo log 跟写数据有一个很大的差异，那就是 **redo log 是顺序 IO，而写数据涉及到随机 IO**，写数据需要寻址，找到对应的位置，然后更新/添加/删除，而写 redo log 则是在一个固定的位置循环写入，是顺序 IO，所以速度要高于写数据。

redo log 本身又分为：

- 日志缓冲（redo log buffer)，该部分日志是易失性的。
- 重做日志(redo log file)，这是磁盘上的日志文件，该部分日志是持久的。

MySQL 每执行一条 DML 语句，先将记录写入 `redo log buffer`，后续在某个时间点再一次性将多个操作记录写到 `redo log file`，这种先写日志再写磁盘的技术就是 MySQL 里经常说到的 `WAL(Write-Ahead Logging)` 技术（预写日志）。

### 1.2 两阶段提交

在 MySQL 中，两阶段提交的主角就是 binlog 和 redolog，我们来看一个两阶段提交的流程图：

![img](https://cdn.jsdelivr.net/gh/wuwenyishi/shared@image/2022/04/02/2320-EBGook.awebp)

从上图中可以看出，在最后提交事务的时候，有 3 个步骤：

1. 写入 redo log，处于 prepare 状态。
2. 写 binlog。
3. 修改 redo log 状态变为 commit。

由于 `redo log` 的提交分为 `prepare` 和 `commit` 两个阶段，所以称之为两阶段提交。

## 2. 为什么需要两阶段提交

如果没有两阶段提交，那么 binlog 和 redolog 的提交，无非就是两种形式：

1. 先写 binlog 再写 redolog。
2. 先写 redolog 再写 binlog。

这两种情况我们分别来看。

假设我们要向表中插入一条记录 R，如果是先写 binlog 再写 redolog，那么假设 binlog 写完后崩溃了，此时 redolog 还没写。那么重启恢复的时候就会出问题：binlog 中已经有 R 的记录了，当从机从主机同步数据的时候或者我们使用 binlog 恢复数据的时候，就会同步到 R 这条记录；但是 redolog 中没有关于 R 的记录，所以崩溃恢复之后，插入 R 记录的这个事务是无效的，即数据库中没有该行记录，这就造成了数据不一致。

相反，假设我们要向表中插入一条记录 R，如果是先写 redolog 再写 binlog，那么假设 redolog 写完后崩溃了，此时 binlog 还没写。那么重启恢复的时候也会出问题：redolog 中已经有 R 的记录了，所以崩溃恢复之后，插入 R 记录的这个事务是有效的，通过该记录将数据恢复到数据库中；但是 binlog 中还没有关于 R 的记录，所以当从机从主机同步数据的时候或者我们使用 binlog 恢复数据的时候，就不会同步到 R 这条记录，这就造成了数据不一致。

那么按照前面说的两阶段提交就能解决问题吗？

我们来看如下三种情况：

**情况一：**一阶段提交之后崩溃了，即`写入 redo log，处于 prepare 状态` 的时候崩溃了，此时：

由于 binlog 还没写，redo log 处于 prepare 状态还没提交，所以崩溃恢复的时候，这个事务会回滚，此时 binlog 还没写，所以也不会传到备库。

**情况二：**假设写完 binlog 之后崩溃了，此时：

redolog 中的日志是不完整的，处于 prepare 状态，还没有提交，那么恢复的时候，首先检查 binlog 中的事务是否存在并且完整，如果存在且完整，则直接提交事务，如果不存在或者不完整，则回滚事务。

**情况三：**假设 redolog 处于 commit 状态的时候崩溃了，那么重启后的处理方案同情况二。

由此可见，两阶段提交能够确保数据的一致性。

## 3. 小结

好啦，今天和小伙伴们简单聊了一下 MySQL 中的两阶段提交，有问题欢迎留言讨论。


作者：江南一点雨
链接：https://juejin.cn/post/7080366887695024141
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。