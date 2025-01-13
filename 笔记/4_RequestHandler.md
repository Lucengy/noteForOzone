## 1. 前言

```
Handler to handleRequest the OmRequests
```

用来处理OmRequest的handler接口，主要看其接口方法

* OMResponse handleOMRequest(OMRequest request)
* OMResponse validateRequest(OMRequest omRequest) throws IOException
* OMClientResponse handleWriteRequest(OMRequest omRequest, TermIndex termIndex) throws IOExcepiton
* void updateDoubleBuffer(OzoneManagerDoubleBuffer ozoneManagerDoubleBuffer)

