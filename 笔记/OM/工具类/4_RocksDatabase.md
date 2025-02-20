## 1. 前言

## 2. 内部类Columnfamily

实际上就是对ColumnFamilyHandle的封装，同时持有一个ColumnName的字符串表示和byte[]表示

```java
public final class ColumnFamily {
    private final byte[] nameBytes;
    private final String name;
    private final ColumnFamilyHandle handle;
}
```

构造器的形参是ColumnFamilyHandle，amazing，要注意调用方的实参是怎么获取的。这里的构造器是使用private进行修饰的，那么就肯定存在static方法，要么在ColumnFamily中，要么在外部类RocksDatabase中，用来构造ColumnFamily实例。大难是在RocksDatabase.toColumNFamilyMap()方法中。

```java
private ColumnFamily(ColumnFamilyHandle handle) throws RocksDBException {
    this.nameBytes = handle.getName();
    this.name = byte2String(nameBytes);
    this.handle = handle;
}
```

接下来就是封装了put和delete的操作，但是没有封装get的操作

