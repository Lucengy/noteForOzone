## 1. 前言

根据对RaftServerImpl中leader的流程分析，我们将重点放在

* StateMachine.startTransaction(RaftClientRequest)
* StateMachine.preAppendTransaction()
* StateMachine.applyTransactionSerial(TransactionContext)
* StateMachine.applyTransaction(TransactionContext)

## 1. startTransaction(RaftClientRequest)

先将代码贴上来，一点点分析

1. Ratis使用Grpc进行RPC调用，其并不了解上层StateMachine使用的数据格式，所以统一将其序列化成字节流，这里先获取请求字节流，将其转换为对应的OMRequest数据类型
2. 通过RequestHandler.validateRequest(OMRequest)方法对请求进行校验，然后构建TransactionContext对象返回，这里需要注意的是Transaction.setStateMachineContext(Object)方法，形参是一个Object类型，作为attached对象放在TransactionContext对象中，后续使用

```java
  public TransactionContext startTransaction(
      RaftClientRequest raftClientRequest) throws IOException {
    ByteString messageContent = raftClientRequest.getMessage().getContent();
    OMRequest omRequest = OMRatisHelper.convertByteStringToOMRequest(
        messageContent);

    Preconditions.checkArgument(raftClientRequest.getRaftGroupId().equals(
        raftGroupId));
    try {
      handler.validateRequest(omRequest);
    } catch (IOException ioe) {
      TransactionContext ctxt = TransactionContext.newBuilder()
          .setClientRequest(raftClientRequest)
          .setStateMachine(this)
          .setServerRole(RaftProtos.RaftPeerRole.LEADER)
          .build();
      ctxt.setException(ioe);
      return ctxt;
    }

    return TransactionContext.newBuilder()
        .setClientRequest(raftClientRequest)
        .setStateMachine(this)
        .setServerRole(RaftProtos.RaftPeerRole.LEADER)
        .setLogData(raftClientRequest.getMessage().getContent())
        .setStateMachineContext(omRequest)
        .build();
  }
```

## x. preAppendTransaction()

```java
public TransactionContext preAppendTransaction(TransactionContext trx)
      throws IOException {
    final OMRequest request = (OMRequest) trx.getStateMachineContext();
    OzoneManagerProtocolProtos.Type cmdType = request.getCmdType();

    OzoneManagerPrepareState prepareState = ozoneManager.getPrepareState();

    if (cmdType == OzoneManagerProtocolProtos.Type.Prepare) {
      // Must authenticate prepare requests here, since we must determine
      // whether or not to apply the prepare gate before proceeding with the
      // prepare request.
      UserGroupInformation userGroupInformation =
          UserGroupInformation.createRemoteUser(
          request.getUserInfo().getUserName());
      if (ozoneManager.getAclsEnabled()
          && !ozoneManager.isAdmin(userGroupInformation)) {
        String message = "Access denied for user " + userGroupInformation
            + ". "
            + "Superuser privilege is required to prepare ozone managers.";
        OMException cause =
            new OMException(message, OMException.ResultCodes.ACCESS_DENIED);
        // Leader should not step down because of this failure.
        throw new StateMachineException(message, cause, false);
      } else {
        prepareState.enablePrepareGate();
      }
    }

    // In prepare mode, only prepare and cancel requests are allowed to go
    // through.
    if (prepareState.requestAllowed(cmdType)) {
      return trx;
    } else {
      String message = "Cannot apply write request " +
          request.getCmdType().name() + " when OM is in prepare mode.";
      OMException cause = new OMException(message,
          OMException.ResultCodes.NOT_SUPPORTED_OPERATION_WHEN_PREPARED);
      // Indicate that the leader should not step down because of this failure.
      throw new StateMachineException(message, cause, false);
    }
  }
```

乍一看，这里也只是做了一部分校验工作，涉及到OzoneManagerPrepareState的部分在HDDS-4569，后续再看，这里无伤大雅

## x. applyTransactionSerial(TransactionContext)

这里没有使用到

## 2. 入口方法， applyTransation(TransactionContext)

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

## 3. RequestAuditor接口 

Audit 审计，用来将OMRequest请求转换为审计对象

## x. 以OMVolumeCreateRequest为例

```java

```

