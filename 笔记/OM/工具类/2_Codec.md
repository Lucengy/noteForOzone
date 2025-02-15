## 1.Codec

根据JavaDoc，Codec是一个接口，主要是定义了序列化和反序列化的api。有两种

1. (反)序列化成byte[]
2. (反)序列化成CodecBuffer对象

```
Codec interface to serialize/deserialize objjects to/from bytes.
A codec implementation must support the byte[] methods and may optionally support the CodecBuffer methods
```

接口是如何定义的呢，先看byte[]相关的api

```java
public interface Codec<T> {
    //空的byte[]数组
    byte[] EMPTY_BYTE_ARRAY = {};
    
    /**
   * 这里是将对象序列化成byte[]
   * Convert object to raw persisted format.
   * @param object The original java object. Should not be null.
   */
  byte[] toPersistedFormat(T object) throws IOException;
    
  /**
   * 这里是从byte[]反序列化成原始对象
   * Convert object from raw persisted format.
   *
   * @param rawData Byte array from the key/value store. Should not be null.
   */
  T fromPersistedFormat(byte[] rawData) throws IOException;
}
```

再看CodecBuffer相关的api，因为JavaDoc中也明确讲了，Codec接口的实现类并不一定要实现CodecBuffer的相关方法，所以CodecBuffer的相关api都使用default进行了修饰

```java
public interface Codec<T> {
  /**
   * 实现类并不一定会实现CodecBuffer相关方法，所以先进行判断
   * Does this {@link Codec} support the {@link CodecBuffer} methods?
   * If this method returns true, this class must implement both
   * {@link #toCodecBuffer(Object, CodecBuffer.Allocator)} and
   * {@link #fromCodecBuffer(CodecBuffer)}.
   *
   * @return ture iff this class supports the {@link CodecBuffer} methods.
   */
  default boolean supportCodecBuffer() {
    return false;
  }
    
  /**
   * @return an upper bound, which should be obtained without serialization,
   *         of the serialized size of the given object.
   */
  default int getSerializedSizeUpperBound(T object) {
    throw new UnsupportedOperationException();
  }
    
  /**
   * Serialize the given object to bytes.
   *
   * @param object The object to be serialized.
   * @param allocator To allocate a buffer.
   * @return a buffer storing the serialized bytes.
   */
  default CodecBuffer toCodecBuffer(@Nonnull T object,
      CodecBuffer.Allocator allocator) throws IOException {
    throw new UnsupportedOperationException();
  }
    
  /**
   * Serialize the given object to bytes.
   *
   * @param object The object to be serialized.
   * @return a direct buffer storing the serialized bytes.
   */
  default CodecBuffer toDirectCodecBuffer(@Nonnull T object)
      throws IOException {
    return toCodecBuffer(object, CodecBuffer.Allocator.getDirect());
  }

  /**
   * Serialize the given object to bytes.
   *
   * @param object The object to be serialized.
   * @return a heap buffer storing the serialized bytes.
   */
  default CodecBuffer toHeapCodecBuffer(@Nonnull T object)
      throws IOException {
    return toCodecBuffer(object, CodecBuffer.Allocator.getHeap());
  }

  /**
   * Deserialize an object from the given buffer.
   *
   * @param buffer Storing the serialized bytes of an object.
   * @return the deserialized object.
   */
  default T fromCodecBuffer(@Nonnull CodecBuffer buffer) throws IOException {
    throw new UnsupportedOperationException();
  }
}
```

除去byte[]和CodecBuffer的相关(反)序列化方法，还存在一个copyObject方法，当Object是不可变时，即使用final进行修饰时，可以返回Object本身，否则的话应该要返回一个深拷贝的副本

```java
  /**
   * Copy the given object.
   * When the given object is immutable,
   * the implementation of this method may safely return the given object.
   *
   * @param object The object to be copied.
   * @return a copy of the given object.  When the given object is immutable,
   *         the returned object can possibly be the same as the given object.
   */
  T copyObject(T object);
```

## 2. CodecBuffer

说白了就是对ByteBuf的一层封装