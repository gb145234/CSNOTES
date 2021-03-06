# 05 内存快照：宕机后，Redis如何实现快速恢复？

Redis AOF日志方法的好处是每次执行只需要记录操作命令，需要持久化的数据量不大。一般而言，只要你采用的不是 always 的持久化策略，就不会对性能造成太大影响。

但是，也正因为记录的是操作命令，而不是实际的数据，所以，用 AOF 方法进行故障恢复的时候，需要逐一把操作日志都执行一遍。如果操作日志非常多，Redis 就会恢复得很缓慢，影响到正常使用。这当然不是理想的结果。那么，还有没有既可以保证可靠性，还能在宕机时实现**快速恢复**的其他方法呢？

内存快照。所谓内存快照，就是指内存中的数据在某一个时刻的状态记录。redis把内存中的数据在某一时刻的状态以文件的形式写到磁盘上，也就是快照，这个快照文件就称为 RDB 文件，RDB 是 Redis DataBase 的缩写。

和 AOF 相比，RDB 记录的是**某一时刻的数据**，并不是操作，所以，在做数据恢复时，我们可以**直接把 RDB 文件读入内存**，很快地完成恢复。

## 给哪些内存数据做快照

Redis RDB快照是全量快照，会将所有数据保存到磁盘中。

Redis 提供了两个命令来生成 RDB 文件，分别是 save 和 bgsave。

- save：在主线程中执行，会导致阻塞；
- bgsave：创建一个子进程，专门用于写入 RDB 文件，避免了主线程的阻塞，这也是 Redis RDB 文件生成的默认配置。

## 快照时数据能修改吗

虽然bgsave能避免主线程阻塞，但是避免阻塞和正常处理写操作并不是一回事。此时，主线程的确没有阻塞，可以正常接收请求，但是，为了保证快照完整性，它只能处理读操作，因为不能修改正在执行快照的数据。

这时 Redis 就会借助操作系统提供的写时复制技术（Copy-On-Write, COW），在执行快照的同时，正常处理写操作。 

此时，如果主线程对这些数据也都是读操作（例如图中的键值对 A），那么，主线程和 bgsave 子进程相互不影响。但是，如果主线程要修改一块数据（例如图中的键值对 C），那么，这块数据就会被主线程复制一份，生成该数据的副本（键值对 C’）。然后，主线程在这个数据副本上进行修改。同时，bgsave 子进程可以继续把原来的数据（键值对 C）写入 RDB 文件。

![img](https://typora-gao-pic.oss-cn-beijing.aliyuncs.com/a2e5a3571e200cb771ed8a1cd14d5558.jpg)

这既保证了快照的完整性，也允许主线程同时对数据进行修改，避免了对正常业务的影响。

## 可以每秒做一次快照吗

如果每次快照的时间间隔过长，一旦发生宕机会导致丢失过多数据，可靠性很低

如果频繁执行全量快照，则会导致两方面问题：

1.  频繁将全量数据写入磁盘，会加大磁盘写入压力
2. 频繁fork子进程，fork过程会阻塞主线程：fork 这个创建过程本身会阻塞主线程，而且主线程的内存越大，页表越大，拷贝页表的时间越长，阻塞时间越长。如果频繁 fork 出 bgsave 子进程，这就会频繁阻塞主线程了（所以，在 Redis 中如果有一个 bgsave 在运行，就不会再启动第二个 bgsave 子进程）

此时，我们可以做增量快照，增量快照就是指做了一次全量快照后，后续的快照只对修改的数据进行快照记录，这样可以避免每次全量快照的开销。

Redis 4.0 中提出了一个混合使用 AOF 日志和内存快照的方法。简单来说，内存快照以一定的频率执行，在两次快照之间，使用 AOF 日志记录这期间的所有命令操作。这样一来，快照不用很频繁地执行，这就避免了频繁 fork 对主线程的影响。而且，AOF 日志也只用记录两次快照间的操作，也就是说，不需要记录所有操作了，因此，就不会出现文件过大的情况了，也可以避免重写开销。

如下图所示，T1 和 T2 时刻的修改，用 AOF 日志记录，等到第二次做全量快照时，就可以清空 AOF 日志，因为此时的修改都已经记录到快照中了，

![img](https://typora-gao-pic.oss-cn-beijing.aliyuncs.com/e4c5846616c19fe03dbf528437beb320.jpg)

前一节提到AOF日志记录所有操作记录，但有了RDB快照能力后，AOF就不用记录所有操作了，只需要**记录增量记录**即可，记录量就小了。若要恢复数据，可用RDB文件再加上AOF日志就可以全量恢复数据了。在速度上，因为RDB是二进制数据流，可以快速恢复出redis数据，然后在此基础上小量的执行AOF操作命令，相比于只用AOF来恢复全量数据的操作，也不会太多影响到恢复速度。

最后，关于 AOF 和 RDB 的选择问题，有三点建议：

- 数据不能丢失时，内存快照和 AOF 的混合使用是一个很好的选择；
- 如果允许分钟级别的数据丢失，可以只使用 RDB；
- 如果只用 AOF，优先使用 everysec 的配置选项，因为它在可靠性和性能之间取了一个平衡。

