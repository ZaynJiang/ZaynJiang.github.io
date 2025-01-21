## Pebble

Pebble 是 Cockroach  参考 RocksDB 并用 Go 语言开发的高性能 KV 存储引擎。一直以来 CockroachDB 以 RocksDB 作为底层存储引擎，虽然 RocksDB 是一款非常优秀的 KV 内嵌式数据库，但是在 Cockroach 的使用过程中，也遇到了一些使用上的缺陷及性能问题（参考：[Cockroach 官方文档](http://link.zhihu.com/?target=https%3A//www.cockroachlabs.com/blog/pebble-rocksdb-kv-store/)）。由于存储引擎是数据库中非常核心的组件，为了更好的把控核心技术，Cockroach 打算自研底层存储引擎，于是乎 Pebble 呼之欲出