## 1. 前言

接口，用来存储Ozone的元数据信息。JavaDoc描述的信息很详细

```
/**
 * Interface for key-value store that stores ozone metadata. Ozone metadata is
 * stored as key value pairs, both key and value are arbitrary byte arrays. Each
 * Table Stores a certain kind of keys and values. This allows a DB to have
 * different kind of tables.
 */
```

## 2. 内部接口KeyValue

一点点看，先看KeyValue接口，单纯的POJO类

```java
interface KeyValue<KEY, VALUE> {
    KEY getKey() throws IOException;
    VALUE getValue() throws IOException;
}
```

同时Table提供了获取KeyValue的静态方法，返回一个匿名内部类对象，通过静态方法的实参获得

```java
static <K, V> KeyValue<K, V> newKeyValue(K key, V value) {
    return new KeyValue<K, V>() {
        @Override
        public K getKey() {
            return key;
        }
        
        @Override
        public V getValue() {
            return value;
        }
        
        @Override
        public String toString(){
            
        }
    }
}
```

## 3. 接口TableIterator

继承自Iterator，不单单有hasNext() next()等方法，同时定义了自己的接口API。这里需要注意的是TableIterator中继承自Iterator中的泛型为T类型，即Value类型，那么Iterator遍历的是value而不是key

```java
public interface TableIterator<KEY, T> extends Iterator<T>, Closeable {
    void seekToFirst();
    
    void seekToLast();
    
    T seek(KEY key) throws IOExcepiton;
    
    void removeFromDB() throws IOException;
}
```

## 4. 内部接口KeyvalueIterator

继承自TableIterator，主要是限定了两个泛型为外部类定义的KEY和KeyValue<KEY, VALUE>类型

```java
interface KeyValueIterator<KEY, VALUE> extends TableIterator<KEY, KeyValue<KEY, VALUE>>{
    
}
```

## 5. 类CacheKey

主要是一个POJO类，单纯的持有一个KEY对象

```java
public class CacheKey<KEY> implements Comparable<CacheKey<KEY>> {
    private final KEY key;
    
    @Override
    public int compareTo(CacheKey<KEY> other) {
        if(Objects.equals(key, other.key)) {
            return 0;
        } else {
            return key.toString().compareTo(other.key.toString());
        }
    }
}
```

