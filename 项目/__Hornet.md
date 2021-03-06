# Hornet (暂停开发)

Hornet是一个用C++开发针对CDN的轻量级缓存引擎，[github地址](https://github.com/GhostZCH/hornet)。

## 状态

目前已经实现了基础架构，没有经过完整测试。考虑到开发便捷度，正在考虑用go重新开发。

## 背景

在生产环境和测试环境上使用和测试了数种缓存引擎后，发现已有的成品都有一些不能满足需求的地方（见下方表格），于是诞生了开发本程序的想法。

|引擎|优点|缺点|
|--|--|--|
|proxy-cache(nginx)|与nginx一体，方便在nginx内操作|为每一个key分配一个文件，产生大量小文件，不适合大规模使用|
|varnish| 性能好|重启后会丢失已有的缓存信息|
|squid| http支持较好 | 架构老旧，不支持多核，性能较差 |
|traffic server| 性能高，功能全 | 代码量惊人，没有二次开发的可能性|
|redis/memcached| 性能高 | 仅支持内存缓存，缓存量太小|
|其他支持kv的数据库| - | 需要额外设置过期时间或者容量有限，或者性能不高（大多数据库都考虑读写均衡，但是缓存需要的是极高的读性能写性能要求较低）|

## 原则 & 功能

整个程序设计的原则是高效，可靠，轻量级，方便二次开发。每一个功能都经过反复推敲， 力图用最简洁的代码实现最实用的功能，每一个实现方案都要在性能功能和实现复杂度做权衡，拒绝为少量性能功能提升增加大量的代码。丰富的功能往往意味着繁杂的配置和大量多余的功能，本引擎的目标则是提供一个便于二次定制化开发的基础，让每个用户都能在维护少量代码的基础上实现较好的定制化服务。

+ 通过HTTP协议与其他程序通信
+ 仅支持带有epoll，accept4, 等特性的linux系统，（ubuntu 14.04作为标准环境）
+ 支持内存缓存和硬盘缓存，数据在重启后仍然能够正常使用
+ 设计容量可以存储千万级缓存对象，每个缓存对象可设置一个文件夹路径（HASH）和4个Tag
+ 可以删除其中的一个，或根据文件夹和tag删除一批（秒级删除）
+ 不支持回源功能
+ 不检测恶意攻击和非正常使用
+ 用几个文件存储大量的小文件
+ 通过expire选项控制过期时间
+ 当缓存满时自动删除一部分旧数据
+ 在一个正常的服务器上，qps不低于30k（平均数据大小24k）
+ 分别输出access和error日志
+ 支持操作系统的信号量，退出，截断日志
+ PUT一定次数才缓存
+ 提供一个额外的日志头，引擎会将这个头部加入到日志了方便定位问题（一般是url）
+ 返回插入一个头部，用于标示缓存状态，一般是主机名和age

## 应用场景

+ 较高的流量和缓存，包含多个域名，有紧急删除的需求
+ 长期运行，偶尔更新，不需要热启动，通过集群和二级缓存提高可靠性
+ 运行在其他负载均衡系统（nginx, haproxy等）之后
+ 使用专用服务器有足够的内存和CPU资源
+ 暂不支持取源功能

## API

+ 每个tag是一个不超过65535的整数，默认为0。可以用来表示文件类型，子文件夹，子用户等属性, 协议，子域名。删除时65535为通配符，不检测这个设置

* GET /$dir/$id HTTP/1.1

    例如：

        // get www.myweb.com/xxx.jpg
        GET /12345/654123

* PUT /$dir/$id?tag1=$tag1&$tag2=$tag2 HTTP/1.1

	例如：

        // add www.myweb.com/xxx.jpg
        PUT /12345/654123?tag1=1

* DEL /$dir/$id?tag1=$tag1&$tag2=$tag2 HTTP/1.1

	例如：

        // delete www.myweb.com/xxx.jpg
        DEL /12345/654123?tag1=1 HTTP/1.1

        // delete www.myweb.com/*.jpg, id是0表示统配
        DEL /12345/0?tag1=1 HTTP/1.1

        // delete www.myweb.com/*
        DEL /12345/0?tag1=65535 HTTP/1.1

        // delete */*
        DEL /0/0?tag1=65535 HTTP/1.1

## 设计

### 架构

master-worker
+ master: 负责 管理worker, 执行周期行的任务
+ worker: 负责读写数据，处理用户请求

### 文件系统

用N个大文件组成一个类似环形缓冲区的文件系统

+ 由一个meta文件记录缓存对象信息，启动时加载到内存
+ 多个大小固定的文件，每个保存一系列的文件，序号递增
+ 删除文件只删除meta中的记录
+ 超过文件大小限制的单独缓存文件
+ 顺序写入，写满一个文件后，打开新文件
+ 超过最大限制存储空间后删除序号最小的文件，标记这个文件中的所有缓存为删除状态，直达所有缓存对象不在被woker使用，正式删除文件

### 流程

#### 总体流程

![总体流程](https://github.com/GhostZCH/hornet/blob/master/start-end.png?raw=true)

#### 事件处理

![事件处理](https://github.com/GhostZCH/hornet/blob/master/process.png?raw=true)

## Q & A

### 为什么不是LRU
LRU使用最广的方法，但是在CDN实际使用中有一定缺陷，比如遇到爬虫时会造成非热点数据刷掉热点数据。根据数据统计，绝大多数的访问是由及少比例的url产生的，这些对象的访问频率非常高，即使用最简单的FIFO过期方式，也不会对命中率产生太大影响，线上环境一般会部署二级缓存，同时穿透两层缓存的概率很低。但是这种FIFO的方法代码简单很多，不需要每次移动数据或者维护队列。例如：一个资源每分钟访问100次，FIFO的过期过程中大概需要2小时，也就是每隔两小时miss一次，命中率大概是 （100 * 60 * 2 - 1） / 100 * 60 * 2，是不是使用LRU对命中的影响可以忽略不计。

### 为什么不增加回源功能
回源功能往往涉及负载均衡，同时需要读取网站配置。考虑到这部分较为复杂，同时处于缓存程序之前的nginx等已经有完善的功能，没必要再重复造轮子。

### 为什么多个文件不是单个文件多个块
+ 单个文件多个块形式不支持在已经有缓存内容的情况下用户调整缓存块的大小
+ 假设现在第n个块已经写满，准备写第n+1个块，但是测试第n+1个块的数据正在被使用，这时可能要用到读写锁等方式才能保证正常。如果用多个文件则没有这个烦恼，可以直接打开新文件并在meta中把最早的一个文件标记成delete状态，等到use数降到0再删除这个文件。

### 为什么不采整个文件做为一个空间，有空位置就存储的方式
会产生一定程度的碎片空间，也不方便过期旧文件

### 为什么批量删除用tag
目前主流的设计有两种批量删除，有的厂商两种都提供：

+ 按照目录删除，Ａ厂和Ｔ厂都支持这种功能，使用trie树实现
+ 按照正则表达式删除，一些老牌的厂商支持这个功能，这个功能比较鸡肋，对于缓存对象比较多的情况需要消耗几十分钟甚至几个小时才能完成一次,在内存中保存完整的url消耗大量的内存。

根据一些线上使用情况，删除操作使用不是很频繁（与访问量对比），批量删除发生的更少。事实上用户需要的更多是根据前缀和后缀进行删除，比如删除`www.myweb.me/news/*.jpg`完整的正则虽然可以完美的实现功能，但是耗时太长。利用trie树虽然会多删除一些但是可以在很短的时间执行完（几秒或几分钟）。本程序目前使用的是一层目录加tag的方式进行删除（经过测试在1000W对象中删除50%的时间不超过１s）。

第一层目录一般是域名，如www.myweb.me，也可以包含目录信息，如:www.myweb.me/news/sport/。
tag可以灵活使用，建议至少选择一个作为文件扩展名（例如: jpg=1,html=2,..）。也可以有更多灵活的用法例如下面的方式:

+ 表示多层子目录，如/2018/03/01/xxx.jpg -> tag1=2018&tag2=3&tag3=1&tag4=１
+ 标志文件大小范围，如：size < 4k = 1, 4k < size < 32k = 2 
+ 标志图片文件分辨率
+ 参数的hash

利用trie树进行目录删除是一个很好的功能，只是要增加一定量的代码，做为本项目的后续改进方向之一。tags有正则和目录删除不具有的优势，后续将继续保留。

## TODO

- [ ] 架构图
- [ ] 类图

