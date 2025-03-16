## 1. 前言

```
Handler to handleRequest the OmRequests
```

用来处理OmRequest的handler接口，主要看其接口方法

* OMResponse handleOMRequest(OMRequest request)
* OMResponse validateRequest(OMRequest omRequest) throws IOException
* OMClientResponse handleWriteRequest(OMRequest omRequest, TermIndex termIndex) throws IOExcepiton
* void updateDoubleBuffer(OzoneManagerDoubleBuffer ozoneManagerDoubleBuffer)

## 2. 实现类OzoneManagerReqeustHandler

从OzoneManagerStateMachine中过来，将目光先放到hadnlerWriteRequest()方法中

```java
@Override
public OMClientResponse handleWriteRequest(OMRequest omRequest, TermIndex termIndex) throws IOException {
    injectPause();
    OMClientRequest omClientRequest =
        OzoneManagerRatisUtils.createClientRequest(omRequest, impl);
    return captureLatencyNs(
        impl.getPerfMetrics().getValidateAndUpdateCacheLatencyNs(),
        () -> {
            OMClientResponse omClientResponse =
                omClientRequest.validateAndUpdateCache(getOzoneManager(), termIndex);
            Preconditions.checkNotNull(omClientResponse,
                                       "omClientResponse returned by validateAndUpdateCache cannot be null");
            if (omRequest.getCmdType() != Type.Prepare) {
                ozoneManagerDoubleBuffer.add(omClientResponse, termIndex);
            }
            return omClientResponse;
        });
}
```

这个调用对端是OMClientRequest.getValidateAndUPdateCacheLatencyNs()方法，因为我们跟踪的是createVolume()方法，那么我们接下来看的是OMVolumeCreateRequest子类的validateAndUpdateCache()方法，最终调用的是OMVolumeRequest.createVolume()方法

```java
protected static void createVolume(
    final OMMetadataManager omMetadataManager, OmVolumeArgs omVolumeArgs,
    PersistedUserVolumeInfo volumeList, String dbVolumeKey,
    String dbUserKey, long transactionLogIndex) {
    // Update cache: Update user and volume cache.
    omMetadataManager.getUserTable().addCacheEntry(new CacheKey<>(dbUserKey),
                                                   CacheValue.get(transactionLogIndex, volumeList));

    omMetadataManager.getVolumeTable().addCacheEntry(
        new CacheKey<>(dbVolumeKey),
        CacheValue.get(transactionLogIndex, omVolumeArgs));
}
```

这里是添加了两个cache，一个是userTable，一个是volumeTable。盲猜一手，这里的tableName实际是RocksDB中的ColumnFamily。没错，getTable(String name)最终调用的是db.getColumnFamily(name)方法     

```java
@Override
public <K, V> Table<K, V> getTable(String name,
                                   Class<K> keyType, Class<V> valueType,
                                   TableCache.CacheType cacheType) throws IOException {
    return new TypedTable<>(getTable(name), codecRegistry, keyType,
                            valueType, cacheType, threadNamePrefix);
}
```

```java
@Override
public RDBTable getTable(String name) throws IOException {
    final ColumnFamily handle = db.getColumnFamily(name);
    if (handle == null) {
        throw new IOException("No such table in this DB. TableName : " + name);
    }
    return new RDBTable(this.db, handle, rdbMetrics);      
}
```

