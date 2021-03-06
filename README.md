# kvstore

##背景

这是课程CS162的项目，分布式KV存储, 由C语言编写

##技术要点
> 网络请求服务：采用线程池加阻塞IO完成 <br>
> 缓存系统：采用LRU的淘汰策略 <br>
> 磁盘存储：每一项数据存储在一个文件中，存储数据的二进制格式 <br>
> 一致性算法: 采用二阶段提交协议 <br>
> 负载均衡算法: 一致性哈希，多副本存储 <br>

###模块设计
kvstore主要结构为主从结构，包括Master和Slave节点，每一个存储节点包括网络请求模块，数据存储模块，包括内存中的缓存和磁盘中的数据。Master节点
包括缓存，但不进行磁盘存储，同时进行请求转发。Slave节点包括缓存和磁盘存储。
####网络请求
网络请求模块采用的是线程池加阻塞IO，算是比较低效的部分。可以使用事件循环和回调提高网络请求效率。请求由4字节的消息长度加消息内容组成。
####缓存
缓存使用的是LRU淘汰策略。
####存储
kvstore直接存储的二进制数据，每个Key值一个文件，采用链接编号处理哈希冲突。这里的处理应该是很低效的，文件数过多。其实可以将数据集中写在几个
文件中，同时维护Key和数据在文件中的位置。（？）

####负载均衡
在分布式系统中，为了避免单点问题，数据项一般在系统中存在多个数据备份，如何存放同一数据以及如何存放不同数据都是需要考虑的问题。
数据分布存在两个对应关系，Key和逻辑位置的关系，逻辑位置和实际位置的关系。
哈希分布又称Round-Robin分布，假设有N个数据节点，数据存储位置由Hash(Key) % N 决定，该方法对于存取都比较方便，但分布式系统一般都是可扩展的，
在添加一个节点时，数据存储节点为Hash(Key) % (N + 1), 大部分数据需要进行迁移，扩展不是很方便，这里的问题主要是将逻辑位置等同于物理位置，
没有进行分离；
虚拟桶，引入虚拟桶的概念，Key根据Hash值分配虚拟桶，每个物理节点含有不同的虚拟桶，维护一个物理节点和虚拟桶的对应关系，进行数据访问，先找到对应
的虚拟桶，再找到相应的物理节点。那么当系统进行扩展加入新节点时，只需要为新节点分配对应的虚拟桶即可，只会影响该桶的数据及其相关的节点，
其中的虚拟桶和节点的对应关系可以通过ZooKeeper等系统维护，比较灵活。
一致性哈希，主要的思想就是哈希环，哈希环的地址由0~2^32-1组成，每个节点计算Hash(node)分布在哈希环上，假设数据节点的Hash值分别为K1，K2，K3，
那么每个节点的值范围为k3~K1，k1~k2，k2~k3，k3~k1，那么每次新加入数据节点，只会影响一个节点的数据。存在的一个问题就是每个节点的值范围有可能不同
，导致每个节点的负载不平衡，同时也不容易更加节点的性能等进行数据分配。这里引入虚拟节点的概念，引入K个虚拟节点均匀分布在哈希环上，每个数据节点
对应M个虚拟节点，这个对应关系同时可以使用Zookeeper等进行维护，同时可以根据数据节点的性能，负载等进行实时调整。同时在进行数据节点扩展时，可以为
新的数据节点分配虚拟节点，影响的也是相关的虚拟节点上的数据。
####一致性算法
使用二阶段提交协议。主要包括三个阶段，Master节点接收到Put和Delete命令时，会执行一致性协议保证各个slave节点数据的一致性。这里以Delete为例，
第一阶段：Master接收到Delete指令后，向Key所在数据节点发送VOTE_COMMIT询问，同时等待数据节点的消息返回；
第二阶段：Master等待所有Key所在数据节点的回复，存在COMMIT和ABORT两种情况；
第三阶段: Master在收到相关数据节点的投票后，确认是否COMMIT或者ABORT，并通知数据节点；
二阶段提交协议有几个比较明显的优缺点，最大的优点就是简单；缺点也比较明显，Master单点，如果Master节点进行备份，就会引入同步问题；在第二阶段，
Master要等待所有的节点回复之后才能确定是否提交操作，等待时间由回复最慢的那个节点决定。
