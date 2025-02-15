## 1. BatchOperation接口

神奇的接口，只是实现了AutoCloseable接口，并重写了close()方法，不再抛出异常

```java
public interface BatchOperation extends Autocloseable {
    @Override
    void close();
}
```

## 2. RDBBatchOperation实现类

### 1. 内部类Bytes

封装了三个实例变量，分别是

1. byte[] array
2. CodecBuffer buffer
3. int hash

那么Bytes的组成只能时byte[]或者CodecBuffer其中之一，可以看其构造器，存在两个构造器，分别用来初始虎啊arry和buffer，赋值一个，另外一个设置为null

```java
static final class Bytes {   
    private final byte[] array;    
    private final CodecBuffer buffer;   
    private final int hash;        
    Bytes(CodecBuffer buffer) {        
        this.array = null;        
        this.buffer = Objects.requireNonNUll(buffer, "buffer == null");        
        this.hash = buffer.asReadOnlyByteBuffer().hashCode();   
    }        
    Bytes(byte[] arr) {        
        this.array = arr;        
        this.buffer = null;        
        this.has = ByteBuffer.wrap(array).hashCode();    
    }
}
```

### 2. 内部类Op

Enum类，只有一个选项DELETE

```java
private enum Op {
    DELETE;
}
```

### 3. 内部类OpCache

持有一个内部类FamilyCache，用来缓存一个列族的WriteBatch操作，WriteBatch是外部类RDBBatcheOperation的实例变量。FamilyCache持有一个FamilyColumn对象，用来指定缓存的列族，持有一个Map对象，泛型分别为Map\<Bytes, Object>，根据JavaDoc，通过Object的类型来获取该操作是put还是delete

```
A (dbKey -> dbValue) map, where the dbKey is Bytes and the dbValue type is Object
When ebValue is a byte[]/ByteBuffer, it represents a put-op
Otherwise, it represents a delete-op(dbValue is Op.DELETE)
```

### 4. RDBBatchOperation本身

这里有两个点需要关注。

首先是WriteBatch对象，根据RocksDB中的描述，WriteBatch对象是对操作的一个缓存，将对应的delete/put操作先缓存在本地，那么何时提交呢？

```C++
  #include "rocksdb/write_batch.h"
  ...
  std::string value;
  rocksdb::Status s = db->Get(rocksdb::ReadOptions(), key1, &value);
  if (s.ok()) {
    rocksdb::WriteBatch batch;
    batch.Delete(key1);
    batch.Put(key2, value);
    s = db->Write(rocksdb::WriteOptions(), &batch);
  }
```

根据RDBBatchOperation的commit()实例方法来看，WriteBatch的提交是通过调用RocksDB.write(WriteOptions, WriteBatch)方法来实现的

```java
public void commit(RocksDatabase db) throws IOException {
    debug(() -> String.format("%s: commit %s",
        name, opCache.getCommitString()));
    try (Closeable ignored = opCache.prepareBatchWrite()) {
      db.batchWrite(writeBatch);
    }
  }

  public void commit(RocksDatabase db, ManagedWriteOptions writeOptions)
      throws IOException {
    debug(() -> String.format("%s: commit-with-writeOptions %s",
        name, opCache.getCommitString()));
    try (Closeable ignored = opCache.prepareBatchWrite()) {
      db.batchWrite(writeBatch, writeOptions);
    }
  }
```

其次，需要理解的点是为什么有OpCache类，根据HDDS-8128，主要是因为需要在WriteBatch中，可能存在着对一个key进行多次的操作，尤其以多次修改后删除key这种情况会极大的浪费资源，所以对writeBatch的操作先进行缓存，如果后续对同一个key再次有put/delete操作时，先在本地进行修改，再提交给RocksDB

```
we add a cache to RDBBatchOperation for deduplication. Within a batch, the put-ops and delete-ops of the same key can be safely deduplicated. Only the last op has to be applied to the db. All the previous ops can be discarded.
```

