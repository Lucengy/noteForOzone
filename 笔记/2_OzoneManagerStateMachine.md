## 1. 入口方法， applyTransation(TransactionContext)

调用实例方法runCommand(OMRequest, TermIndex)

调用OzoneMangerRequestHandler.handlerWriteRequest(omRequest, TermIndex)方法

```java
 @Override
  public OMClientResponse handleWriteRequest(OMRequest omRequest, TermIndex termIndex) throws IOException {
    injectPause();
    OMClientRequest omClientRequest =
        OzoneManagerRatisUtils.createClientRequest(omRequest, impl); //1
    return captureLatencyNs(
        impl.getPerfMetrics().getValidateAndUpdateCacheLatencyNs(),
        () -> {
          OMClientResponse omClientResponse =
              omClientRequest.validateAndUpdateCache(getOzoneManager(), termIndex); // 2
          Preconditions.checkNotNull(omClientResponse,
              "omClientResponse returned by validateAndUpdateCache cannot be null");
          if (omRequest.getCmdType() != Type.Prepare) {
            ozoneManagerDoubleBuffer.add(omClientResponse, termIndex);
          }
          return omClientResponse;
        });
  }
```

1. 构造对应的OMClientRequest对象
2. 调用构造对象的validateAndUpdateCache(OzoneManager, TermIndex)方法

那么接下来将目光转移到OMClientRequest类中，该类是一个抽象类，实现了RequestAuditor接口

## 2. RequestAuditor接口 

Audit 审计，用来将OMRequest请求转换为审计对象

