## 1. 前言

相较于Ratis，Ozone的protobuf文件定义的就比较混乱

从OmClientProtocol.proto文件出发，在文件的最后，存在rpc定义

```protobuf
service OzoneManagerService {
    // A client-to-OM RPC to send client requests to OM Ratis server
    rpc submitRequest(OMRequest)
          returns(OMResponse);
}
```

其实现类为OzoneManagerServiceGrpc.java

```java
public class OzoneManagerServiceGrpc extends OzoneManagerServiceImplBase {
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
}
```

omTranslator类型为OzoneManagerProtocolServerSideTranslatorPB，主要是调用OzoneProtocolMessageDispatcher的processRequest()方法以及四针的processRequest()方法，处理逻辑在自身的internalProcessReuqest(OMRequest)方法中

```java
private OMResponse internalProcessRequest(OMRequest request) throws
      ServiceException {
    OMClientRequest omClientRequest = null;
    boolean s3Auth = false;

    try {
      if (request.hasS3Authentication()) {
        OzoneManager.setS3Auth(request.getS3Authentication());
        try {
          s3Auth = true;
          // If request has S3Authentication, validate S3 credentials.
          // If current OM is leader and then proceed with the request.
          S3SecurityUtil.validateS3Credential(request, ozoneManager);
        } catch (IOException ex) {
          return createErrorResponse(request, ex);
        }
      }

      if (!isRatisEnabled()) {
        return submitRequestDirectlyToOM(request);
      }

      if (OmUtils.isReadOnly(request)) {
        return submitReadRequestToOM(request);
      }

      // To validate credentials we have already verified leader status.
      // This will skip of checking leader status again if request has S3Auth.
      if (!s3Auth) {
        OzoneManagerRatisUtils.checkLeaderStatus(ozoneManager);
      }
      OMRequest requestToSubmit;
      try {
        omClientRequest = createClientRequest(request, ozoneManager);
        // TODO: Note: Due to HDDS-6055, createClientRequest() could now
        //  return null, which triggered the findbugs warning.
        //  Added the assertion.
        assert (omClientRequest != null);
        OMClientRequest finalOmClientRequest = omClientRequest;
        requestToSubmit = preExecute(finalOmClientRequest);
      } catch (IOException ex) {
        if (omClientRequest != null) {
          omClientRequest.handleRequestFailure(ozoneManager);
        }
        return createErrorResponse(request, ex);
      }

      OMResponse response = submitRequestToRatis(requestToSubmit);
      if (!response.getSuccess()) {
        omClientRequest.handleRequestFailure(ozoneManager);
      }
      return response;
    } finally {
      OzoneManager.setS3Auth(null);
    }
  }
```

然后调用submitRequestToRatis(OMRequest request)方法。然后问题就回到了OzoneManagerStateMachine中

根据Ratis的流程

- leader收到logEntry
- StateMachine.startTransaction(RaftClientRequest)
- StateMachine.preAppendTransaction()
- leader将logEntry放到自己的RaftLog中
- leader将logEntry发给各follower
- 针对已经committed的信息，leader调用StateMachine.applyTransacionSerial(TransactionContext)和StateMachine.applyTransaction(TransactionContext)方法

方法的入口变成了OzoneManagerStateMachine.startTransaction(RaftClientRequest)方法。这个方法只是简单的对request进行校验，核心的方法在OzoneManagerStateMachine.applyTransaction(TransactionContext)方法中。这里有一个小细节，就是在startTransaction()方法中是可以在TransactionContext对象中attach一个Object的对象，Ozone attache的对象是OMRequest，在applyTransaction中取得OMRequest这个对象进行操作。具体的逻辑实现在runCommand(OMRequest, TermIndex)方法中，该方法调用了RequestHandler.handleWriteRequest(OMRequest, TermIndex)方法，实现类为OzoneManagerRequestHandler

