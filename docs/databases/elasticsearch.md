# elasticsearch

## es底层结构

因为es底层全文检索库是lucene，它本身就是一个功能齐全的搜索引擎。

“分片”是Lucene的一个索引，副本分片的用途：1.主节点故障时的故障转移；2.增加的读取吞吐量；分片有主分片和副本，默认情况下，每个索引会创建5个分片。
每个分片包含多个segment“分段”，其中分段是倒排索引，默认每秒都会生成一个段文件，可通过'GET /test/_segments'查看索引的段；段一旦创建则不会再被修改，多个段会在段合并的阶段合并在一起，段合并需要消耗大量的i/o;

为了防止elasticsearch宕机造成数据丢失保证可靠存储，es会将每次写入数据同时写到translog日志中。translog还用于提供实时CRUD。 当尝试按ID检索，更新或删除文档时，它会首先检查translog中是否有任何最近的更改，然后再尝试从相关段中检索文档。 这意味着它始终可以实时访问最新的已知文档版本；

写入磁盘的倒排索引永远不会改变；good：无需锁，不用担心多进程操作更改问题；bad：更新了词典词库后，老的索引不能生效。如果要使其可搜索，则必须重建整个索引；

## Elasticsearch写入步骤

1. 新document首先写入内存Buffer缓存中。
2. 每隔一段时间，执行“commitpoint”操作，buffer写入新Segment中。
3. 新segment写入文件系统缓存 filesystem cache。
4. 文件系统缓存中的index segment被fsync强制刷到磁盘上，确保物理写入。
此时，新egment被打开供search操作。
5. 清空内存buffer，可以接收新的文档写入。

refresh：es为了保证‘实时性’，会做refresh操作，将index-buffer中数据生成的segment写到文件系统；相比于Lucene的提交操作，ES的refresh是相对轻量级的操作；默认1s，可通过此进行设置：PUT /myindex {"settings": {"refresh_interval": "30s"}}

flush：新创建的数据会先进入到index buffer之后，与此同时会将操作记录在translog之中，当发生refresh时translog中的操作记录并不会被清除，而是当数据从文件系统缓存中被写入磁盘之后才会将translog中清空。从filesystem cache写入磁盘的过程就是flush。

如果数据量并不那么大的话，过多的分片会影响查询时的效率，因为每个分片都带有一部分数据，需要分散查询，最后整理结果，一般使用节点数<=主分片数*（副本+1）；<br>路由，允许查询时指定路由，相同路由的，必定在相同的分片上；频繁的修改会导致大量的小段，小的段必定会导致查询变慢；
