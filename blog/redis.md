# Redis持久化选项之RDB和AOF全面对比
> 本文由笔者翻译自redis官方的持久化配置详细说明，原文传送门 [Redis Persistence](https://redis.io/topics/persistence) 


在Redis中，有不同的持久化选项：

- **RDB**：RDB持久化按照指定的时间间隔，执行数据集的时间点快照。
- **AOF**：AOF持久化记录了由服务器接收的所有写操作。服务器启动时会回放写操作，以重建原始数据集。记录写操作的格式和Redis协议是相同的，并且采用仅追加方式。当日志太大时，Redis可以在后台重写日志。
- **不进行持久化**：可以完全禁用持久性，如果接受服务器运行时存在数据。
- **RDB + AOF**：可以在同一实例中同时合并AOF和RDB。在这种情况下，Redis重启时，首先使用AOF文件用于重建原始数据集，因为它可以保证是最完整的。

要理解的最重要的事情是RDB与AOF持久化之间的不同取舍。让我们从RDB开始：



## RDB的优势

- RDB使用了某一时间点压缩的单文件表示数据。RDB文件非常适合备份。例如，可能希望在最近24小时每小时存档一次RDB文件，并保存最近30天的RDB快照。这样，在发生灾难时，可以容易地还原不同版本的数据集。
- RDB对于灾难恢复非常有用，它是一个压缩文件，可以传输到远程数据中心或Amazon S3（可能已加密）上。
- RDB最大限度地提高了Redis的性能。原因是Redis父进程需要做的唯一工作就是fork一个子进程，剩下的工作由子进程执行。父进程不会执行磁盘I / O或类似操作。
- 与AOF相比，在大数据集下RDB能更快地重启。



## RDB的缺点

- 如果在Redis停止工作时（如断电），需要尽可能减少数据丢失，那么RDB不够好。用户可以配置不同的*保存点*（例如，在五分钟内至少对数据集进行100次写入，允许有多个保存点）。通常会每隔五分钟或更长时间创建一次RDB快照。因此出于任何原因，如果Redis在没有正确关闭的情况下停止工作，最近几分钟的数据会丢失。
- RDB持久化需要经常使用fork创建使用子进程。如果数据集很大，fork可能会很耗时。并且如果数据集很大且CPU性能不佳，则可能导致Redis停止为客户端服务几毫秒甚至一秒钟。AOF也需要fork，但是可以调整日志重写的频率，而无需在可靠性上进行权衡。



## AOF的优势

- 使用AOF Redis更加可靠：可以采取不同的fsync策略：不写回，每秒写回，每次查询写回。使用fsync的默认策略（每秒写回），性能仍然很好。原因是同步由后台线程执行，当前没有同步进行时，主线程将尽力执行写操作）。但是可能损失一秒的写入操作。
- AOF日志是仅追加的日志，所以不会出现查询。如果断电，也不会出现损坏问题。即使由于某种原因（磁盘已满或其他原因），日志末尾写入了一半的写操作命令，也可以通过redis-check-aof工具修复。
- AOF文件太大时，Redis可以在后台自动重写AOF。AOF日志重写是十分安全的。原因是Redis会将写操作继续追加到旧文件，同时生成一个新文件。新文件包含了创建当前数据集所需的最少操作。一旦准备好第二个文件，Redis会切换到新文件，并开始对新文件进行追加。
- AOF使用易于理解和解析的格式陆续地记录了所有操作。用户可以轻松导出AOF文件。例如，即使不小心使用 FLUSHALL 命令刷新了全部内容，只要在此期间未执行日志重写，仍然可以保存数据集。具体操作是停止 Redis 服务器、删除最近的命令并重启。



## AOF的缺点

- 对于同一数据集，AOF文件通常大于等效的RDB文件。
- 实际中选择同步策略，AOF可能比RDB慢。通常，在将同步策略设置为*每秒写回的情况下，*Redis性能仍然很高。在禁用同步的情况下，即使在高负载下，它会与RDB一样快。在较大写负载的情况下，RDB能够提供在延迟最大化的更多保证。
- 在特定命令中出现过很少见的错误（例如，其中有一个涉及阻塞的命令，例如BRPOPLPUSH），导致产生的AOF重载后无法重现完全相同的数据集。这些错误很少见。相关人员在测试套件中进行了测试，自动创建了随机的复杂数据集，然后重新加载它们以验证是否正常。但是，对于RDB而言，这类错误几乎是不可能的。更明确地说：Redis AOF通过增量更新已有状态来实现持久化，和Mysql类似。而RDB快照一次又一次地创建所有内容，理论上讲更健壮。但是1、Redis每次重写AOF时，都会从依据内存当前实际数据从头开始重新创建AOF，与追加的方式相比（或重写是根据读取旧AOF而不是根据内存中的数据），更有效抵抗错误。2、现实环境中，相关人员从未收到过用户检测到的AOF文件损坏的报告。



## 使用建议

通常的建议是：如果希望获得与PostgreSQL相当的数据安全性，则应同时使用两种持久性方法。

如果很重视数据，在发生灾难的情况下同时可以接受几分钟的数据丢失，则可以只使用RDB。

也有很多用户单独使用AOF，但开发人员不建议这样做。原因是经常创建RDB快照以进行数据库备份，能加快重启速度。同时应注意AOF引擎中存在一些错误。

注意：由于所有这些原因，相关人员将来可能会最终将AOF和RDB统一为一个持久化模型（长期的计划）。





以下各节将说明有关这两个持久性模型的更多详细信息。





## 快照

默认情况下，Redis将数据集的快照保存在磁盘的二进制文件`dump.rdb`上。用户可以配置Redis，使其在N秒的时间内至少有M个更改时保存一次数据集，或者用户可以手动调用[SAVE](https://redis.io/commands/save)或[BGSAVE](https://redis.io/commands/bgsave)命令。

例如，此配置将在每60秒至少更改了1000个键时，自动将备份数据集到磁盘上：

```
save 60 1000
```

这种策略称为*快照*。



### 快照运行原理

每当Redis需要将数据集刷到盘时，就会发生以下情况：

- Redis 进行fork，此时有一个父过程和一个子进程。
- 子进程开始将数据集写入临时RDB文件。
- 当子进程完成新的RDB文件的写入后，它将替换旧的RDB文件。

快照持久化的方式是和COW（copy-on-write）机制相关的。





## AOF文件

快照不是很可靠。如果运行Redis的计算机停止运行，电源线出现故障或不小心`kill `掉实例，则写入Redis的最新数据将丢失。尽管这对于某些应用程序可能不是什么大问题，但有些使用场景要求充分的可靠性。在这些情况下，Redis并不是可行的选择。

*AOF文件*对于Redis而言是二选一，完全可靠的策略。它在1.1版中可用。

可以在配置文件中打开AOF：

```
appendonly yes
```

从现在开始，每次Redis收到更改数据集的命令（例如[SET](https://redis.io/commands/set)）时，它将添加到AOF。当Redis重启时，它将回放AOF以重建状态。



### 日志重写

可以想到，随着写操作被执行，AOF文件越来越大。例如，如果增加一个计数器100次，最终数据集中值有一个最终值的键，而在AOF中却包含100条操作。重建当前状态时，其中99条是不需要的。

因此，Redis支持一个有趣的功能：它可以在后台重建AOF，而不会中断对客户端的服务。每次发出BGREWRITEAOF时，Redis都会编写最短的命令来重建当前内存中的数据集。Redis 2.2中如果使用AOF，则需要不时运行BGREWRITEAOF)。Redis 2.4能够自动触发日志重写（有关更多信息，请参见2.4示例配置文件）。



### AOF文件的可靠性如何？

可以配置Redis 在磁盘上进行同步的次数。共有三个选项：

- `appendfsync always`：每次都`fsync`命令，会非常慢，但十分安全。请注意，在多个客户端或管道的一批命令被执行后，这些命令会追加到AOF，因此这意味着一次写入和一次fsync（在返回响应之前）。
- `appendfsync everysec`：`fsync`每秒。速度足够快（在2.4版本中可能与快照速度一样快）。灾难发生时，可能只会丢失1秒的数据。
- `appendfsync no`：从不`fsync`，只需将数据交给操作系统即可。更快但相对不安全的方法。通常，使用此配置时，Linux将每30秒刷新一次数据，但这取决于内核的实际调整情况。

推荐（也是默认的）策略是`fsync`每秒执行一次。它既快速又安全。`always`策略在实践中非常慢，但是支持组提交。因此如果有多个并行写入，Redis将尝试执行单个`fsync`操作。



### 如果AOF被截断怎么办？

写入AOF文件时，服务器可能崩溃，或者存储AOF文件的卷已满。发生这种情况时，AOF仍包含给定时间点版本数据集的一致数据（默认的AOF fsync策略下，该版本数据可能会过期达一秒钟），但是AOF中的最后一条命令可能会被截断。最新major的Redis版本仍将能够加载AOF，只需丢弃文件中最后一个格式不正确的命令即可。在这种情况下，服务器将呈现如下日志：

```
* Reading RDB preamble from AOF file...
* Reading the remaining AOF tail...
# !!! Warning: short read while loading the AOF file !!!
# !!! Truncating the AOF at offset 439 !!!
# AOF loaded anyway because aof-load-truncated is enabled
```

可以根据需要更改默认配置以强制Redis停止。但是无论文件中的最后一个命令格式是否正确，默认配置都是继续执行，以确保重新启动后的可用性。

旧版本的Redis可能无法恢复，并且可能需要执行以下步骤：

- 备份AOF文件。

- 使用Redis附带的`redis-check-aof`的工具修复原始文件：

  $ redis-check-aof --fix

- （可选）用于`diff -u`检查两个文件之间的区别。

- 指定文件，重新启动服务器。



### 如果AOF损坏了怎么办？

如果AOF文件不仅被截断，而且中间部分被无效字节序列破坏，则情况会更加复杂。Redis将在启动时控告并中止：

```
* Reading the remaining AOF tail...
# Bad file format reading the append only file: make a backup of your AOF file, then use ./redis-check-aof --fix <filename>
```

最好的办法是运行`redis-check-aof`实用程序，初始化时不要带`--fix`选项。然后了解问题，跳转到给定的文件中的偏移，并查看是否可以手动修复文件：AOF文件使用与Redis协议格式相同的格式 ，手动修复非常简单。此外，可以让实用程序修复文件。但是如果损坏恰好是在文件的初始部分，那么使用工具修复时，从无效部分到文件末尾的所有部分都可能会被丢弃，会导致大量数据丢失。



### AOF怎么实现的

日志重写使用与快照相同的写时复制技巧。它是这样工作的：

- Redis [forks](http://linux.die.net/man/2/fork)，现在有一个子进程和一个父进程。
- 子进程开始在临时文件中写新的AOF。
- 父进程将所有新的更改累积在内存缓冲区中（与此同时，它会将新更改写入旧的追加文件中。如果重写失败，数据是安全的）。
- 当子进程完成对文件的重写后，父进程会收到一个信号，并将内存中的缓冲区附加到在子进程生成的文件末尾。
- 现在，Redis原子地将旧文件重命名为新文件，并开始将新数据追加到新文件中。



### 如何将dump.rdb快照切换到AOF？

在Redis 2.0和Redis 2.2中，执行此操作的过程有所不同，因为您可能会认为它在Redis 2.2中更简单，并且根本不需要重新启动。

**Redis> = 2.2**

- 备份最新的dump.rdb文件。
- 将此备份转移到安全的地方。
- 发出以下两个命令：
- redis-cli配置 appendonly yes
- redis-cli配置 save “”
- 确保数据库包含与rdb文件中包含的key数量是一样的。
- 确保将写操作正确地追加到aof文件中。

第一个CONFIG命令启用“aof文件”。为此，**Redis将阻塞**生成初始存储，然后打开文件进行写入，并将开始附加所有接下来的写入查询。

第二个CONFIG命令用于关闭快照持久化。如果希望可以同时启用两种持久化方法，这一步可选。

**重要信息：**通过修改redis.conf的方式打开AOF开关。否则重启服务器时，会丢失之前的配置更改。服务器将使用旧配置重新启动。

**Redis 2.0**

- 备份最新的dump.rdb文件。
- 将此备份转移到安全的地方。
- 停止对数据库的所有写操作！
- 发出`redis-cli BGREWRITEAOF`命令。这将创建aof文件。
- Redis完成生成AOF存储后，停止服务器。
- 编辑redis.conf，启用aof文件持久化。
- 重新启动服务器。
- 确保数据库包含的key与快照包含的数量一致。
- 确保将写操作正确地追加到aof文件中。



## AOF与RDB持久性之间的相互作用

Redis> = 2.4可以确保进行rdb快照操作时，避免触发AOF重写。或者在AOF重写时不允许BGSAVE。这样可以防止两个Redis后台进程同时执行繁重的磁盘I / O。

当进行快照时，用户使用[BGREWRITEAOF](https://redis.io/commands/bgrewriteaof)显式请求aof日志重写操作时，服务器将回复OK状态码，告知用户已计划该操作。当快照完成后将开始重写。

如果同时启用了AOF和RDB持久化，Redis重启时使用AOF文件重建原始数据集，因为它可以保证是最完整的。



## 备份Redis数据

在开始本节之前，请确保：**确保备份数据库**。磁盘损坏，cloud上的实例消失等等：没有备份意味着数据有消失在/ dev / null的巨大风险。

Redis非常便于数据备份，因为可以在数据库运行时复制RDB文件：RDB一旦生成就永远不会被修改，而RDB在生成时会使用一个临时名称，并且当新的快照完成后，使用rename（2）原子地重命名为最终名称。

这意味着在服务器运行时，复制RDB文件是完全安全的。建议如下：

- 在服务器中创建一个cron job，在一个目录中每小时创建一次RDB快照，并在另一个目录中每天创建一次快照。
- 每次运行cron脚本时，请确保调用`find`命令以删除太旧的快照：例如，可以在最近48小时内每小时进行一次快照，并在一个或两个月内每天进行快照。确保使用数据和时间信息命名快照。
- 确保每天至少一次将RDB快照传输到*数据中心**外部*，或至少传输到运行Redis实例*的物理计算机外部*。

如果仅在启用了AOF持久化的情况下运行Redis实例，也可以复制AOF以便创建备份。该文件可能缺少最后一部分，但是Redis仍然可以加载它（请参阅前面有关AOF文件被截断的部分）。



## 灾难恢复

Redis中的灾难恢复与备份基本相同，并且能够在许多不同的外部数据中心中传输这些备份。即使在某些灾难性事件下，运行redis并生成其快照的主数据中心受影响的情况下，也可以通过这种方式保护数据。

由于许多Redis用户都处于启动阶段，因此没有足够的成本开销，可以采取成本不高的最有趣的灾难恢复技术。

- Amazon S3和类似服务是实施灾难恢复系统的好方法。只需的每天或每小时的RDB快照以加密形式传输到S3。可以使用`gpg -c`（在对称加密模式下）对数据进行加密，同时确保将密码存储在许多不同的安全位置（例如，拷贝一份提供给组织中最重要的人员）。建议使用多个存储服务以提高数据安全性。
- 使用SCP（SSH的一部分）将快照传输到远程服务器。可以采取简单和安全的方法：在距离远的地方获得一个小型VPS，在那儿安装ssh，然后生成不带口令的ssh客户端密钥，将其添加到小型VPS的`authorized_keys`文件中，可以自动传输备份。至少在两个不同的提供商中获得VPS，可以获得最佳结果。

需要注意的是，如果未正确实施此系统，则该系统很容易发生故障。至少要确保传输完成后，能够验证文件大小（该大小应与复制的一个文件是匹配的），如果使用的是VPS，也可以验证SHA1摘要。

如果由于某些原因无法传输新备份，则还需要某种独立的警报系统。
