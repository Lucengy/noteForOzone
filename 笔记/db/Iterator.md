## 1. 前言

All data in the database is logically arranged in sorted order. 原文有如下描述，就是在创建iterator时，会创建一个统一的 视图，后面迭代的返回值是根据这个视图来的，意味着在迭代的过程中，新put的数据并不会被返回，新delete的数据也并不会被删除

```
A consistent-point-in-time view of the database is created when the Iterator is created. Thus, all keys returned via the Iterator are from a consistent view of the database.
```

## 2. 验证删除

```java
public class Demo1 {
    public static void main(String[] args) {
        testWrite();
        testReadUseIterator();
    }
    
    public static void testWrite(){
        final KeyValuePair kv1 = new KeyValuePair("aaa", "beijing");
        final KeyValuePair kv2 = new KeyValuePair("bbb", "shanghai");
        final KeyValuePair kv3 = new KeyValuePair("aab", "shenzhen");
        final KeyValuePair kv4 = new KeyValuePair("baa", "guangzhou");
        final List<KeyValuePair> kvs = Arrays.asList(kv1, kv2, kv3, kv4);
        testPut(kvs);
    }
    
    public static void testPut(List<KeyValuePair> data) {
        System.out.println("***** before testPut *****");
        try(final Options options = new Options().setCreateIfMissing(true);
            RocksDB db = RocksDB.open(options, dbDirectory)){
            data.forEach(kvPair -> {
                try {
                    db.put(kvPair.getKey().getBytes(StandardCharsets.UTF_8), kvPair.getValue().getBytes(StandardCharsets.UTF_8));
                } catch (RocksDBException e) {
                    throw new RuntimeException(e);
                }
            });
        } catch (RocksDBException e) {
            throw new RuntimeException(e);
        }
        System.out.println("***** end testPut *****");
    }
    
    public static void testReadUseIterator() {
        System.out.println("***** before testReadUseIterator *****");
        try(final Options options = new Options().setCreateIfMissing(true);
            RocksDB db = RocksDB.open(options, dbDirectory)){
            RocksIterator iterator = db.newIterator(new ReadOptions());
            for(iterator.seekToFirst(); iterator.isValid(); iterator.next()) {
                System.out.println(new String(iterator.key(), StandardCharsets.UTF_8));
            }
        } catch (RocksDBException e) {
            throw new RuntimeException(e);
        }
        System.out.println("***** end testReadUseIterator *****");
    }
    
    @Data
    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    static class KeyValuePair {
        String key;
        String value;
    }
}
```

先写入四条kv数据，可以看到结果输出为

![1739259884798](C:\Users\v587\AppData\Roaming\Typora\typora-user-images\1739259884798.png)

将testReadUseIterator的过程加入sleep，将testDelete()方法放到单独的线程中去执行

```java
public class Demo1 {
    static final String dbDirectory;
    static RocksDB db;
    static {
        dbDirectory = "D:\\software\\idea_workstation\\flume\\db";
        RocksDB.loadLibrary();
    }
    
    public static void main(String[] args) throws InterruptedException {
        initDB();
        Thread t1 = new Thread(Demo1::testReadUseIterator);
        Thread t2 = new Thread(() -> {
            List<KeyValuePair> list = Arrays.asList(new KeyValuePair("aab", ""), new KeyValuePair("bbb", ""));;
            testDelete(list);
        });

        t1.start();
        TimeUnit.SECONDS.sleep(1);
        t2.start();
        t1.join();
        t2.join();
        System.out.println("----- 华丽的分割线 -----");
        testReadUseIterator();
    }
    
    public static void initDB(){
        System.out.println("***** before initDB *****");
        try(final Options options = new Options().setCreateIfMissing(true)) {
            db = RocksDB.open(options, dbDirectory);
        } catch (RocksDBException e) {
            throw new RuntimeException(e);
        }
        System.out.println("***** end initDB *****");
    }
    
    public static void testReadUseIterator() {
        System.out.println("***** before testReadUseIterator *****");

        RocksIterator iterator = db.newIterator(new ReadOptions());
        for(iterator.seekToFirst(); iterator.isValid(); iterator.next()) {
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println(new String(iterator.key(), StandardCharsets.UTF_8));
        }

        System.out.println("***** end testReadUseIterator *****");
    }
    
    public static void testDelete(List<KeyValuePair> data) {
        System.out.println("***** before testDelete *****");

        try{
            for (KeyValuePair datum : data) {
                db.delete(datum.getKey().getBytes(StandardCharsets.UTF_8));
            }
        } catch (RocksDBException e) {
            throw new RuntimeException(e);
        }

        System.out.println("***** end testDelete *****");
    }
}
```

可以看到，在先拿到Iterator之后调用delete()方法，并没有影响到已有的Iterator，等待线程执行完毕后，再次调用Iterator，就无法get到已经删除的两个key值

![1739260949663](C:\Users\v587\AppData\Roaming\Typora\typora-user-images\1739260949663.png)

## 3. Error Handling

```
Iterator::status() returns the error of the iterating. The errors include I/O errors, checksum mismatch, unsupported operations, internal errors, or other errors.
```

主要是通过iterator.isValid()以及iterator.status()方法来进行判断

```
If there is no error, the status is Status::OK(). If the status is not OK, the iterator will be invalidated too. In another word, if Iterator::Valid() is true, status() is guaranteed to be OK() so it's safe to proceed other operations without checking status():
```

如果没有error出现的话，那么status()方法返回ok。如果Status not Ok，那么iterator.isValid()方法也会返回false

```
On the other hand, if Iterator::Valid() is false, there are two possibilities: (1) We reached the end of the data. In this case, status() is OK(); (2) there is an error. In this case status() is not OK(). It is always a good practice to check status() if the iterator is invalidated.
```

反过来讲，如果valid()方法返回false，那么会有两种情况

1. 迭代器已经迭代完成，此时Status()方法是ok的
2. 出现了error，此时status()方法是NOk的

## 4. Iterator资源释放和refreshing

Java好像并没有提供fresh()或者refresh()这个接口功能

Iterator本身是不需要太多的内存的，但是因为它提供了统一视图这个概念，就意味着它要将一些资源pin到内存中，所以用完iterator要及时的释放，这样的话可以及时的回收资源

除此之外，Iterator在创建之后，如果重复使用，意味着它的数据是过期的，C++的接口中提供了refresh()这个接口方法

## 5. Read-ahead

