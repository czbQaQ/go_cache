# <center> go_cache
&nbsp;&nbsp;本项目跟随[GeeCache](https://geektutu.com/post/geecache.html)实现了一个模仿groupcache的简单分布式缓存系统，整体项目包括以下几部分：  
* LRU缓存淘汰策略：  
&nbsp;&nbsp;最近最少使用（LRU）认为，如果数据最近被访问过，那么将来被访问的概率也会更高。LRU算法的实现是维护一个队列，如果某条记录被访问了，则移动到队尾，那么对手则是最近最少访问的数据，淘汰该条记录即可。LRU的底层数据结构包含绿色的字典，存储键和值的映射关系；红色的是双向链表实现的队列，将所有的的值放的双向链表中。  
![](./image/lru.jpg#pic_center "lru结构图")  
+ 单机并发缓存  
&nbsp;&nbsp;为保证并发的正确性，在读写缓存的时候增加互斥锁。  
+  Http通信  
&nbsp;&nbsp;从不同的节点获取缓存使用http进行通信传输。主要实现http.Handler结构的ServeHTTP函数。  
+ 一致性哈希  
&nbsp;&nbsp;为了保证在每一次访问时对同一个key获取同一个节点的缓存，使用哈希的方式计算获取节点。同时为了防止出现节点新增或减少大规模调整key对应的节点而采用一致性哈希。一致性哈希通俗来时是一个圆环，计算节点和虚拟节点的哈希值放置在环上，如果请求的key到来，计算key的哈希值，顺时针选取节点。当出现新增或减少节点时，对应到新的节点需要重新从数据源获取值（面试被问到过）。  
![新增节点](./image/add_peer.jpg#pic_center "新增peer8，只有key27需要调整到perr8，其余不变，peer8从数据源获取key27对应的值")  
+ 防止缓存击穿  
  > 缓存在同一时刻全部失效，造成瞬时DB请求量大、压力骤增，引起雪崩。缓存雪崩通常因为缓存服务器宕机、缓存的 key 设置了相同的过期时间等引起。  

  > 缓存击穿：一个存在的key，在缓存过期的一刻，同时有大量的请求，这些请求会击穿到DB，造成瞬时DB请求量大，压力骤增。

  > 查询一个不存在的数据，因为不存在则不会写到缓存中，所以每次都会去请求 DB，如果瞬间流量过大，穿透到 DB，导致宕机。  
  
  &nbsp;&nbsp;为了防止上述缓存击穿的情况出现。对每一个key增加一个call，如果这个call是空的那么久请求同时使用sync.WaitGroup计数器加一，等待结束了计数器再减一，然后删除这个key，在这个过程中如果其他的请求来了判断这个key是否存在，如果存在就等待。之后直接返回值。
