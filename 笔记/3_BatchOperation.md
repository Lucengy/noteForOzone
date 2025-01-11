## 1. BatchOperation

batchOperation单纯的继承了AutoCloseable接口，并覆盖其close()方法，不再抛出异常

```java
public interface BatchOperation extends AutoCloseable {

  @Override
  void close();
}
```

## 2. RDBBatchOperation

### 内部类Op

```java
private enum Op {
    DELETE
}
```

### 内部类Bytes

```java
static final class Bytes {
    private final byte[] array;
    private CodecBuffer buffer;
    private final int hash;
}
```

### 内部类OpCache



