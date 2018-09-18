## Pika 笔记
---
*written by Alex Stocks on 2018/09/07，版权所有，无授权不得转载*


### 0 引言
---
愚人所在公司的大部分服务端业务无论是缓存还是存储颇为依赖 Codis，经过数次踩坑，其中一条经验教训是：线上 Redis 数据不要落地。

也就是说，我司的 Codis 集群中的 Redis，无论是 master 还是 slave，都没有打开 rdb 和 aof，所有数据都放在内存中。Codis 以这种方式“平静地”运行了一年，但是大伙终究心里石头无法落地，现状要求运维的同事在线上部署一种能高效运行且数据能落地的 “Codis”。

经交流和调研，今年七月份运维的同事决定采用 v2.3.x Pika 版的 Codis【下文提及的 Pika 不做特殊说明均指代 Pika 版本的 Codis 集群，pika 则指代单个 pika member】。在经过一段时间测试后，结果也令人满意：无论是在 SATA 盘还是 SSD 盘上，写【set，key 长度 16B， value 长度 30B】 qps 最差 60k/s，稳定情况下 80k/s，峰值可达 100k/s。于是 CTO 便拍板决定继续测试【到目前为止运维同事已经各种测试了两个月】，并根据公司以往的传统：使用开源系统，公司内部必须有人通读其代码，且能够解决掉在测试和线上遇到的问题。

最终这个“光荣任务”落在了愚人肩上。本文用来记录我阅读代码并在改进 Pika 【到 2018/09/07 为止主要是开发相关工具】过程中遇到的一些问题。

### 1 数据迁移
---

八月初运维的同事提出了一个需求：把 Pika 数据实时同步到 Codis 集群，即把 Pika 集群作为数据固化层，把 Codis 作为数据缓存层。

#### 1.1 Pika-port V1

刚开始得到这个需求，愚人的实现思路是：

- 1 通过 redis-cli 向 pika 发送 bgsave 命令，然后把 dump 出来的数据解析后发送给 Codis；
- 2 再开发这样一个工具：根据 dump info 文件存储的 filenum & offset 信息解析 binlog，并把解析出来的写指令增量同步给 Codis。

根据这个思路，借鉴[参考文档1](https://github.com/Qihoo360/pika/wiki/%E4%BD%BF%E7%94%A8binlog%E8%BF%81%E7%A7%BB%E6%95%B0%E6%8D%AE%E5%B7%A5%E5%85%B7)开始实现V1 版本的工具【模仿 redis-port，愚人命名为 pika-port】。但在开发到最后一步时遇到这个问题：pika 以 mmap 方式向磁盘写入 binlog，redis-port 只需要读 binlog，而一般存储系统的读速度最低 5 倍于写速递，当 redis-port 追上 pika 的最新 binlog 文件数据后， 很可能读到截断的脏数据！

因当时刚开始读 pika 代码，遇到这个无法解决坎后便只能放弃这个方案了【后来把pika/src/pika_binlog_sender_thread.cc 详细读懂后已经找到了解决方法，但此时 pika-port V2版本已经开发完毕】。

V1 虽然半途而废，但是开发过程中遇到的两个问题比较有意思，V2 版开发时也需要处理，所以记录如下：

+ pika 的 binlog record 在每个 redis 写命令后面追加了四个额外信息，分别是：Pika Magic [kPikaBinlogMagic]、server_id【用于双 master 同步时做去重】、binlog info【主要是执行命令的时间】以及 send hub 信息，需要过滤掉；

  >  代码详见 include/pika_command.h:Cmd::AppendAffiliatedInfo，修改后的 redis 命令 `set A 1` 格式为 `*7\r\n$3\r\nset\r\n$1\r\nA\r\n$1\r\n1\r\n$14\r\n__PIKA_X#$SKGI\r\n$1\r\n1\r\n$16\r\nj[m\r\n$1\r\n1\r\n`
  >  
  > 这些补充信息在跨机房数据同步的情况下也很有用，详细内容见[参考文档7](http://kernelmaker.github.io/pika-muli-idc)

+ pika 内部有一个特殊的 set 用于记录当前 migrate 信息，set key 前缀是 `_internal:slotkey:4migrate:`，这个在进行数据同步时也需要过滤掉；

#### 1.2 Pika-port V2

V2 版本的 pika-port 相当于是 pika 和 Codis / Redis 之间的 proxy，实现流程是：

+ 1 pika-port 启动时候伪装从 pika 的 slave，向 pika 发送 trysync 指令，如`trysync 10.33.80.155 20847 0 0`，10.33.80.155:20847 为 pika-port 的启动监听地址，后两个参数分别为 filenum 和 offset，同时监听 +1000 地址;
+ 2 pika-port 收到 pika 发来的 wait ack 后，监听 +3000 端口，启动 rsync deamon 等待全量数据同步；
+ 3 pika-port 循环检测 dump info 文件是否存在，当检测到 info 文件存在时，意味着全量同步数据完成，此时阻塞所有流程并把收到的全量数据发送给 Codis；
+ 4 pika-port 根据 info 文件提供的 filenum 和 offset 再次向 pika 发送 trysync 指令，并根据 pika 回复的 ack 获取到 sid 作为此次连接的标识；
+ 5 pika-port 监听 +2000 端口，启动心跳发送线程，首先向 pika 发送 `spci sid`指令，然后每个 1s 向 pika 发送 `ping`指令，并等待 pika 回复的 `pong` ack；
+ 6 pika-port 启动一个 Codis 连接线程池【本质是一个线程池，每个线程启动一个 Codis 连接】；
+ 6 pika 收到心跳后，向 pika-port 发送 `auth sid` 指令成功后，就循环解析 binlog 并把数据增量同步给 pika-port；
+ 7 pika-port 收到 pika 实时同步过来的单个 redis 写指令，过滤掉其中非法指令，再删除合法质量中结尾四个辅助信息，然后根据写指令中的 key 进行 hash 计算后交给线程池中某个线程，此线程将会以阻塞方式将此数据同步给 Codis 直至成功。

整个流程需要对 pika 的主从复制流程非常熟悉，关于主从复制流程可以详细阅读[参考文档2](https://www.jianshu.com/p/01bd76eb7a93)。目前 pika-port 已经开发完毕，支持 v2.3.6 版本的 pika数据实时迁移到 Codis/Redis。

在开发过程中遇到了一些坑，有的是自己对 pika 理解不透彻，有的是 pika 自身一些缺陷，下面详细分小节记录之，以备将来作参考之用。

##### 1.2.1 rsync 启动失败

Pika-port 与 pika 之间全量数据同步是通过 rsync 进行的，如果 pika-port 启动 rsync 失败【譬如rsync 监听端口被占用】，pika-port 所借鉴的 [PikaTrysyncThread::ThreadMain](https://github.com/qihoo360/pika/blob/master/src/pika_trysync_thread.cc#L259) 仅仅记录一个错误日志，然后继续相关流程。

合理的处理方法当然是启动 rsync daemon 失败退出即可，然官方相关处理流程如是，且出现这种错误概率极低，愚人处理方法就是暂时不处理这种 corner case。

##### 1.2.2 非法命令过滤

Pika-port 会对 pika 发来的 redis 写指令进行非法性检查，过滤掉 command 为 auth 以及 key 为 `_internal:slotkey:4migrate:`前缀的非法指令。

在开发过程中，对非法指令的过滤是 [MasterConn::DealMessage](https://github.com/divebomb/pika/blob/master/tools/pika_port/master_conn.cc) 处理的，过滤功能开发到是很简单，但是在开发测试过程中遇到这样一个坑：一旦 pika-port 遇到一个非法指令过滤掉后，pika 与 pika-port 之间的连接就断开发并疯狂重新建立连接。

经过对 [RedisConn::ProcessInputBuffer](https://github.com/PikaLabs/pink/blob/master/pink/src/redis_conn.cc) 详细分析后才发现问题所在： [MasterConn::DealMessage](https://github.com/divebomb/pika/blob/master/tools/pika_port/master_conn.cc) 遇到非法字符串后返回了一个负值作为错误标识，而 [RedisConn::ProcessInputBuffer](https://github.com/PikaLabs/pink/blob/master/pink/src/redis_conn.cc) 调用这个函数后如果检测到结果是负值，就认为处理出错，最终会导致连接被关闭。

最终的解决方法当然是把返回结果改为 0 就可以了。

##### 1.2.3 主从复制过程丢失数据

Pika-port V2开发完毕后测试过程中，遇到这样一个 corner case：通过 redis-cli 向 pika 写入 A 指令【譬如 set A 1】，在 60s 之后再次向 pika 写入 B 指令【譬如 set B 2】，然后立即写入 C 指令【譬如 set C 3】，最后 Codis/Redis 中只有 A 和 C 指令的数据，把 B 质量的数据丢了！

通过 tcpdump 在 pika 和 pika-port 之间进行抓包，分别得到如下两个关键结果【由于花费了半天时间不断重复测试以分析网络流程，所以两幅图时间先后有些错乱，不必较真】：

![](../pic/pika_tcp_fin_reset.jpg)

                                                       ***图1: pika-port fin reset***

![](../pic/pika_tcp_reset_3handshake.jpg)

                                                       ***图2: pika与pika-port 3 handshake***

图1 是在 pika 向 pika-port 写入 B 指令时的网络流程，通过分析 图1 并结合相关代码分析，可以得到这样一个流程：

* 在 60s 的时间间隔内 pika 未向 pika-port 同步数据，导致 pika-port 的超时检查函数 [HolyThread::DoCronTask](https://github.com/PikaLabs/pink/blob/master/pink/src/holy_thread.cc#L189)认为连接超时，便向 pika 发送 fin1 包后，把连接关闭了；
* pika 向 pika-port 写入 B 指令时，pika-port 向 pika 回复了 reset 信号。

图2 则是 pika 向 pika-port 写入 C 指令的网络流程，同样分析后得到其流程是：

* pika 收到 pika-port 发来的 reset 信号并未处理，继续向 pika-port 发送 C 指令；
* pika PikaBinlogSenderThread 此时方能判断出连接已经被 pika-port 关闭(https://github.com/qihoo360/pika/blob/master/src/pika_binlog_sender_thread.cc#L309)，然后关闭连接并重新建立与 pika-port 之间的数据同步连接，并重复发送 C 指令，此次成功。

从 [PikaTrysyncThread::ThreadMain](https://github.com/qihoo360/pika/blob/master/src/pika_binlog_sender_thread.cc#L241) 整个流程可以得出这样一个结论：pika 调用 write api 向 pika-port 写 B 指令的时候，并没有进行读操作以判断当前是否收到了 pika-port 发来的 rst 包，只是调用 write api 向 pika-port 进行了写，并根据其返回值为0就认为写成功了，进而理所当然的认为对端也能收到 B 指令。

可能有些对 tcp 四次挥手逻辑不甚明了的人对这个过程有些不甚了了，根本原因是 tcp 是双向连接，pika-port 只是关闭了 pika-port --> pika 这个方向的连接，而 pika --> pika-port 这个方向的单向连接还是存在的，只不过 pika-port 依赖的 pink 网络库在关闭一个单向连接时调用了 close 函数，导致结果是：pika-port 关闭了 pika-port --> pika 这个方向的连接的同时不再接收 pika --> pika-port 这个方向由 pika 发来的 B 指令数据！

解决问题的根本就在于正确处理 RST 信号，linux manpage 对 RST 信号的处理解释如下：


```
What happens if the client ignores the error return from readline and writes more data to the server? This can happen, for example, if the client needs to perform two writes to the server before reading anything back, with the first write eliciting the RST.

The rule that applies is: When a process writes to a socket that has received an RST, the SIGPIPE signal is sent to the process. The default action of this signal is to terminate the process, so the process must catch the signal to avoid being involuntarily terminated.

If the process either catches the signal and returns from the signal handler, or ignores the signal, the write operation returns EPIPE.
```

上面很清晰的说明：写 B 指令时如果不读取 RST 相关错误信令，写 C 指令时 write 会返回 broken pipe 错误。所以正确的处理方法应该是：在进行 write 之前进行一次 read，以判断对端是否已经发来 fin 包；或者在 write 之后进行 read 以判断对端是否发来 rst 包。

考虑到 [PikaTrysyncThread::ThreadMain](https://github.com/qihoo360/pika/blob/master/src/pika_binlog_sender_thread.cc#L241) 向 pika-port 发送数据的方式是 one way 的，pika-port 自身不会给 pika 回复任何消息，所以第二种方法成本略高。再考虑到这种情况是因为两个写指令之间写时间间隔太长所致，更进一步地处理方法是：每次调用 write 之后记录本次 write 执行的时间，下一次调用 write 时把系统当前时间与上一次 write 的时间进行比较，如果时间间隔超过某个阈值【譬如 1s】，则需要先进行读操作，判断出 pika-port --> pika 方向的连接正常，再调用 write 进行 pika --> pika-port 方向的数据写操作。

根据这个方案的相关改进代码写完，并已向 pika 官方提交了 [pr](https://github.com/PikaLabs/pink/pull/30)，有待 merge。

在测试过程中，发现 pika 自身的 master 和 slave 进行数据复制时，并不会出现数据丢失的错误。经过加 log 分析，愚人在今日[2018/09/08] 下午 15:50pm 发现原因所在：pika slave 并不会对 pika master 之间的数据复制连接进行超时判断，仅仅依靠 tcp 自身的 KeepAlive 特性对连接进行保活【个人认为这种处理方法是不理智的】。至于代码层次原因，详见下图：

![](../pic/pika_keepalive_check.png)

Pika-port 调用了上图[第一个构造函数](https://github.com/pikalabs/pink/blob/master/pink/src/holy_thread.cc#L16)，直接导致  HolyThread::keepalive_time_ 参数被赋值 60，进而导致[HolyThread::DoCronTask](https://github.com/pikalabs/pink/blob/master/pink/src/holy_thread.cc#L186) 超时检查逻辑被激活，然后 pika-port 与 pika 之间连接被 pika-port 关闭。

而 pika 自身则是调用上图的[第二个构造函数](https://github.com/pikalabs/pink/blob/master/pink/src/holy_thread.cc#L25)，直接导致  HolyThread::keepalive_time_ 参数在被 gcc 编译时候被赋值 0，然后 pika slave 就不会去对它与 pika master之间连接作任何超时检查，所以也就不会出现丢数据的问题！

恰当的处理方法当然是重构两个构造函数，让其行为一致，然而作为著名项目的已有代码，相关改动牵一发而动全身，最终处理方法是我在 [pr](https://github.com/PikaLabs/pink/pull/31)【对网络fd进行读写须用 recv，如果用 pread 则会收到 ESPIPE 错误】 中对相关函数所在的头文件中加上注释以进行[调用提醒](https://github.com/divebomb/pink/blob/master/pink/include/server_thread.h#L195)。

至于为何要依赖 tcp 自身的 keepalive 机制而不是在逻辑层对 tcp 连接进行超时判断，pika 开发者陈宗志给出了一个 [blog](http://baotiao.github.io/tech/2015/09/25/tcp-keepalive/) 进行解释，仁者见仁智者见智，这个就不再次探讨了。

在处理这个问题时，与胡伟、[郑树新](https://github.com/zhengshuxin)、[bert](https://github.com/loveyacper)、[hulk](https://github.com/git-hulk)等一帮老友进行了相关探讨，受益匪浅，在此一并致谢！

### 2 数据备份
---

Pika 官方 wiki [[参考文档4](https://github.com/qihoo360/pika/wiki/pika-%E5%BF%AB%E7%85%A7%E5%BC%8F%E5%A4%87%E4%BB%BD%E6%96%B9%E6%A1%88)] 有对其数据备份过程的图文描述，此文就就不再进行转述。

Ardb 作者对 Pika 的评价是  “直接修改了rocksdb代码实现某些功能。这种做法也是双刃剑，改动太多的话，社区的一些修改是很难merge进来的”【详见[参考文档5](http://yinqiwen.github.io/)】。与比较几个主流的基于 RocksDB 实现的 KV 存储引擎（如 TiKV/SSDB/ARDB/CockroachDB）作比较，Pika 确实对 RocksDB 的代码侵入比较严重。RocksDB 默认的备份引擎 BackupEngine 通过 `BackupEngine::Open` 和 `BackupEngine::CreateNewBackup` 即实现了数据的备份【关于RocksDB 的 Backup 接口详见 [参考文档6](http://alexstocks.github.io/html/rocksdb.html) 6.8节】，而 Pika 为了效率起见重新实现了一个 `nemo::BackupEngine`，以进行异步备份。

Pika 的存储引擎 nemo 依赖于其对 RocksDB 的封装引擎 nemo-rocksdb，下面结合[参考文档4](https://github.com/qihoo360/pika/wiki/pika-%E5%BF%AB%E7%85%A7%E5%BC%8F%E5%A4%87%E4%BB%BD%E6%96%B9%E6%A1%88) 从代码层面对备份流程进行详细分析。

<font size=“2” color=blue>***注：本章描述的备份流程基于 pika 的 nemo 引擎，基本与最新的 blackwidow 引擎的备份流程无差。***</font>


#### 2.1 DBNemoCheckpoint
---

nemo:DBNemoCheckpoint 提供了执行实际备份任务的 checkpoint 接口，其实际实现是 nemo:DBNemoCheckpointImpl，其主要接口如下：

```c++
class DBNemoCheckpointImpl : public DBNemoCheckpoint {
  // 如果备份目录和源数据目录在同一个磁盘上，则对 SST 文件进行硬链接，
  // 对 manifest 文件和 wal 文件进行直接拷贝
  virtual Status CreateCheckpoint(const std::string& checkpoint_dir) override;
  // 先阻止文件删除【rocksdb:DB::DisableFileDeletions】，然后获取 rocksdb:DB 快照，如 db 所有文件名称、
  // manifest 文件大小、SequenceNumber 以及同步点(filenum & offset)
  //
  // nemo:BackupEngine 把这些信息组织为BackupContent
  virtual Status GetCheckpointFiles(std::vector<std::string> &live_files,
      VectorLogPtr &live_wal_files, uint64_t &manifest_file_size,
      uint64_t &sequence_number) override;

  // 根据上面获取到的 快照内容 进行文件复制操作
  virtual Status CreateCheckpointWithFiles(const std::string& checkpoint_dir,
      std::vector<std::string> &live_files, VectorLogPtr &live_wal_files,
      uint64_t manifest_file_size, uint64_t sequence_number) override;
}
```

`CreateCheckpoint` 接口可以认为是同步操作，它通过调用 `GetCheckpointFiles` 和 `CreateCheckpointWithFiles` 实现数据备份。

`DBNemoCheckpointImpl::GetCheckpointFiles` 先执行 “组织文件删除”，然后再获取 快照内容。

`DBNemoCheckpointImpl::CreateCheckpointWithFiles(checkpoint_dir, BackupContent)` 详细流程:

- 1 如果 checkpoint 目录 @checkpoint_dir 存在，则退出；
- 2 创建 临时目录 “@checkpoint_dir + .tmp”；
- 3 根据 live file 的名称获取文件的类型，根据类型不同分别进行复制；
	* 3.1 如果 type 是 SST 则进行 hard link，hark link 失败再尝试进行 Copy；
	* 3.2 如果 type 是其他类型则直接进行 Copy，如果 type 是 kDescriptorFile（manifest 文件）还需要指定文件的大小；
- 4 单独创建一个 CURRENT 文件，其内容是 manifest 文件的名称；
- 5 备份 WAL 文件；
	* 5.1 如果文件的类型是归档 WAL，则拒绝备份；
	* 5.2 通过 LogFile::StartSequence() 获取 WAL 初始 SequenceNumber，如果这个 SequenceNumber 小于备份开始时的系统 SequenceNumber，则拒绝备份；
	* 5.3 如果 WAL 文件是最后文件集合的最后一个，则 Copy 文件，且只复制文件在备份开始时的文件 size，以防止复制过多的操作指令；
	* 5.3 如果备份文件和原始文件在同一个文件系统上，则进行 hard link，否则进行 Copy；
- 6 允许文件删除；
- 7 把临时目录 “@checkpoint_dir + .tmp” 重命名为 @checkpoint_dir，并执行 fsync 操作，把数据刷到磁盘。

注：BackupCentent 中别的文件如 CURRENT、SST、Manifest 都是文件名称，唯独 WAL 文件传递了相关的句柄 [LogFile](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/transaction_log.h#L32)。

#### 2.2 BackupEngine
---

基于 DBNemoCheckpoint，nemo:BackupEngine 提供了一个异步备份五种类型数据文件的接口，其定义如下：

```c++
    // Arguments which will used by BackupSave Thread
    // p_engine for BackupEngine handler
    // backup_dir
    // key_type kv, hash, list, set or zset
    struct BackupSaveArgs {
        void *p_engine;
        const std::string backup_dir;
        const std::string key_type;
        Status res;
    };

    struct BackupContent {
        std::vector<std::string> live_files;
        rocksdb::VectorLogPtr live_wal_files;
        uint64_t manifest_file_size = 0;
        uint64_t sequence_number = 0;
    };

    class BackupEngine {
        public:
            ~BackupEngine();
            // 调用 BackupEngine::NewCheckpoint 为五种数据类型分别创建响应的 DBNemoCheckpoint 放入 engines_，
            // 同时创建 BackupEngine 对象
            static Status Open(nemo::Nemo *db, BackupEngine** backup_engine_ptr);
            // 调用 DBNemoCheckpointImpl::GetCheckpointFiles 获取五种类型需要备份的 快照内容 存入 backup_content_
            Status SetBackupContent();
            // 创建五个线程，分别调用 CreateNewBackupSpecify 进行数据备份
            Status CreateNewBackup(const std::string &dir);

            void StopBackup();
            // 调用 DBNemoCheckpointImpl::CreateCheckpointWithFiles 执行具体的备份任务
            // 这个函数之所以类型是 public 的，是为了在 线程函数ThreadFuncSaveSpecify 中能够调用之
            Status CreateNewBackupSpecify(const std::string &dir, const std::string &type);
        private:
            BackupEngine() {}

            std::map<std::string, rocksdb::DBNemoCheckpoint*> engines_; // 保存每个类型的 checkpoint 对象
            std::map<std::string, BackupContent> backup_content_; // 保存每个类型需要复制的 快照内容
            std::map<std::string, pthread_t> backup_pthread_ts_; // 保存每个类型执行备份任务的线程对象

            // 调用 rocksdb::DBNemoCheckpoint::Create 创建 checkpoint 对象
            Status NewCheckpoint(rocksdb::DBNemo *tdb, const std::string &type);
            // 获取每个类型的数据目录
            std::string GetSaveDirByType(const std::string _dir, const std::string& _type) const {
                std::string backup_dir = _dir.empty() ? DEFAULT_BK_PATH : _dir;
                return backup_dir + ((backup_dir.back() != '/') ? "/" : "") + _type;
            }
            Status WaitBackupPthread();
    };
```

`nemo::BackupEngine` 对外的主要接口是 Open、SetBackupContent、CreateNewBackup 和 StopBackup，分别用于 创建 BackupEngine 对象、获取快照内容、执行备份任务和停止备份任务。

#### 2.3 Bgsave
---

`PikaServer::Bgsave` 是 redis 命令 bgsave 的响应函数，通过调用 `nemo::BackupEngine` 相关接口执行备份任务，下面先分别介绍其先关的函数接口。

#### 2.3.1 PikaServer::InitBgsaveEnv 
---

这个函数用于创建数据备份目录，其流程为：

- 1 获取当前时间，以 `%Y%m%d%H%M%S` 格式序列化为字符串；
- 2 创建目录 `pika.conf:dump-path/%Y%m%d`，如果目录已经存在，则删除之；
- 3 删除目录 `pika.conf:dump-path/_FAILED`。

注意上面第二步的备份目录，之所以最终目录只有年月日信息，是因为最终只用了前 8 个字符串作为目录名称。

#### 2.3.2 PikaServer::InitBgsaveEngine

这个函数用于创建 BackupEngine 对象并进行获取五种数据类型的快照内容，其流程为：

- 1 调用 `nemo::BackupEngine::Open` 创建 nemo::BackupEngine 对象；
- 2 通过 `PikaServer::rwlock_::WLock` 进行数据写入 RocksDB::DB 阻止；
- 3 获取当前 Binlog 的 filenum 和 offset；
- 4 调用 `nemo::BackupEngine:: SetBackupContent` 获取快照内容；
- 5 通过 `PikaServer::rwlock_::UnLock` 取消数据写入 RocksDB::DB 阻止。

`PikaClientConn::DoCmd` 在执行写命令的时候，会先调用 `g_pika_server->RWLockReader()` 尝试加上读锁，如果正在执行 Bgsave 则此处就会阻塞等待。 

#### 2.3.3 PikaServer::RunBgsaveEngine

这个函数用于执行具体的备份任务，其流程为：

- 1 调用 `PikaServer::InitBgsaveEnv` 初始化 BGSave 需要的目录环境；
- 2 调用 `PikaServer:: InitBgsaveEngine` 创建 nemo::BackupEngine 对象和获取快照内容；
- 3 调用 `nemo::BackupEngine::CreateNewBackup` 执行备份任务。

#### 2.3.4 PikaServer::DoBgsave

这个函数是 Bgsave 线程的执行体，其流程为：

- 1 调用 `PikaServer::RunBgsaveEngine` 执行数据备份；
- 2 把执行备份任务时长、本机 hostinfo、binlog filenum 和 binlog offset 写入 `pika.conf:dump-path/%Y%m%d/info` 文件；
- 3 如果备份失败，则把 `pika.conf:dump-path/%Y%m%d` 重命名为 `pika.conf:dump-path/%Y%m%d_FAILED`；
- 4 把 `bgsave_info_.bgsaving` 置为 false。

#### 2.3.4 PikaServer::Bgsave

作为命令 bgsave 的响应函数，其流程非常简单：

- 1 如果 `bgsave_info_.bgsaving` 值为 true，则退出，否则把其值置为 true；
- 2 启动 `PikaServer::bgsave_thread_`，通过调用 `PikaServer::DoBgsave` 函数完成备份任务。

## 参考文档

- 1 [使用binlog迁移数据工具](https://github.com/Qihoo360/pika/wiki/%E4%BD%BF%E7%94%A8binlog%E8%BF%81%E7%A7%BB%E6%95%B0%E6%8D%AE%E5%B7%A5%E5%85%B7)
- 2 [pika主从复制原理之工作流程](https://www.jianshu.com/p/01bd76eb7a93)
- 3 [pika主从复制原理之binlog](https://www.jianshu.com/p/d969b6f6ae42)
- 4 [Pika 快照式备份方案](https://github.com/qihoo360/pika/wiki/pika-%E5%BF%AB%E7%85%A7%E5%BC%8F%E5%A4%87%E4%BB%BD%E6%96%B9%E6%A1%88)
- 5 [杂感(2016-06)](http://yinqiwen.github.io/)
- 6 [RocksDB 笔记](http://alexstocks.github.io/html/rocksdb.html)
- 7 [pika 跨机房同步设计](http://kernelmaker.github.io/pika-muli-idc)

## 扒粪者-于雨氏

> 2018/09/07，于雨氏，初作此文于西二旗。
> 
> 2018/09/15，于雨氏，于西二旗添加第二节 “数据备份”。
