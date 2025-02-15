## 1. 前言

接口类，是OzoneManager用来处理客户端请求的类

```java
public interface RequestHandler{
    /**
   * Handle the read requests, and returns OmResponse.
   * @param request
   * @return OmResponse
   */
  OMResponse handleReadRequest(OMRequest request);
    
    /**
   * Validates that the incoming OM request has required parameters.
   * TODO: Add more validation checks before writing the request to Ratis log.
   *
   * @param omRequest client request to OM
   * @throws OMException thrown if required parameters are set to null.
   */
  void validateRequest(OMRequest omRequest) throws OMException;
    
  /**
   * Handle write requests. In HA this will be called from
   * OzoneManagerStateMachine applyTransaction method. In non-HA this will be
   * called from {@link OzoneManagerProtocolServerSideTranslatorPB} for write
   * requests.
   *
   * @param omRequest
   * @param termIndex - ratis transaction log (term, index)
   * @return OMClientResponse
   */
  OMClientResponse handleWriteRequest(OMRequest omRequest, TermIndex termIndex) throws IOException;
    
    /**
   * Update the OzoneManagerDoubleBuffer. This will be called when
   * stateMachine is unpaused and set with new doublebuffer object.
   * @param ozoneManagerDoubleBuffer
   */
  void updateDoubleBuffer(OzoneManagerDoubleBuffer ozoneManagerDoubleBuffer);
} 
```

