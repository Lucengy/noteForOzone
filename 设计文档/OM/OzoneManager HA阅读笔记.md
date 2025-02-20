## 1. 前言

proto文件存在于hadoop-ozone/interface-client下，以OmInterServiceProtocol.proto为例，ozone中，使用的是proto2格式，Ratis在定义message和service使用单独的proto文件，但是ozone将其混合在同一个文件，这里的service定义为

```protobuf
service OzoneManagerService {
    // A client-to-OM RPC to send client requests to OM Ratis server
    rpc submitRequest(OMRequest)
          returns(OMResponse);
}
```

那么入口RPC在OzoneManagerServiceImplBase的实现类，即OzoneManagerServiceGrpc中

```java
 @Override
  public void submitRequest(OMRequest request,
                            io.grpc.stub.StreamObserver<OMResponse>
                                responseObserver) {
    LOG.debug("OzoneManagerServiceGrpc: OzoneManagerServiceImplBase " +
        "processing s3g client submit request - for command {}",
        request.getCmdType().name());
    AtomicInteger callCount = new AtomicInteger(0);

    org.apache.hadoop.ipc.Server.getCurCall().set(new Server.Call(1,
        callCount.incrementAndGet(),
        null,
        null,
        RPC.RpcKind.RPC_PROTOCOL_BUFFER,
        getClientId()));
    // TODO: currently require setting the Server class for each request
    // with thread context (Server.Call()) that includes retries
    // and importantly random ClientId.  This is currently necessary for
    // Om Ratis Server to create createWriteRaftClientRequest.
    // Look to remove Server class requirement for issuing ratis transactions
    // for OMRequests.  Test through successful ratis-enabled OMRequest
    // handling without dependency on hadoop IPC based Server.
    try {
      OMResponse omResponse = this.omTranslator.
          submitRequest(NULL_RPC_CONTROLLER, request);
      responseObserver.onNext(omResponse);
    } catch (Throwable e) {
      LOG.error("Failed to submit request", e);
      IOException ex = new IOException(e.getCause());
      responseObserver.onError(
          Status.INTERNAL.withDescription(ex.getMessage())
              .asRuntimeException());
      return;
    }
    responseObserver.onCompleted();
  }
```

所以将入口的类放在了OzoneManagerProtocolServerSideTranslatorPB，调用了processRequest(OMRequest)--> internalProcessRequest(OMRequest)->submitRequestToRatis(OMRequest) --> OzoneManagerRatisServer.submitRequest(OMRequest) --> submitRequestToRatis(RaftClientRequest) --> submitRequestToRatisImpl(RaftClientRequest) --> RaftServerImpl.submitClientRequestAsync(RaftClientRequest)方法

接下来的逻辑交给了Ratis处理，需要将目光转向OzoneManagerStateMachine

## 2. 有关Cahce design

### cache淘汰策略

```
A cache entry can be purged once the corresponding write transaction has been written to RocksDB.
```

