Memcached与Redis异同比较
1. 浅谈
  redis与memcached都有缓存能力，其中两者都支持key-value的内存存储，而redis同时还具备数据的持久化备份，以及主从模式的数据备份、而memcached支持的数据各更加多样化，如多媒体数据缓存。那么他们的异同到底有多少呢。
2、简单粗暴表格：
|各个方面|Redis|Memcached|
|-|-|-|
|IO|单线程IO复用模型|多线程，非阻塞网络模型|
|数据类型|K-V模型，多媒体|K-V模型，list,set等|
|内存管理机制|K-V模型，多媒体|K-V模型，list,set等|
