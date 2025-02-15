## 1. 前言

原文是这么描述的

```
This page describes the Atomic Read-Modify-Write operation in RocksDB, known as the "Merge" operation. It is an interface overview, aimed at the client or RocksDB user who has the questions: when and why should I use Merge; and how do I use Merge?
```

它是将read-modify-write封装成了一个原子操作

## 2. 使用场景

使用RocksDB提供一个整数的set remove get add接口，接口定义如下

```java
interface Counters {
    void set(String key, int value);
    void remove(String key);
    int get(String key);
    boolean add(String key, int delta);
}
```

可以知道，除去add操作，其他操作都可以直接映射a single operation in rocksdb. 但是，一个add操作却被映射为两个rocksdb操作

除此之外，如果多线程同时操作同一个Counter，那么就会有线程同步的问题，这时候就必须加锁，对性能又会有损失

如果RocksDB支持add语义呢？这里会引发一个新的问题，就是add中delta的类型，目前我们接口中定义的是一个int类型，但如果是其他类型怎么办呢。

## 3. 接口定义

为了解决上述问题，RocksDB定义了新的抽象类: MergeOperator. 它提供了很多功能，用来告知RocksDB怎么核定incremental update operations(called "merge operands") with base-values(Put/Delete)

