## 1. 前言

客户端这一侧的OzoneBucket和OzoneVolume类的祖宗类就是WithMetadate

```java
public class WithMetadata {
    protected Map<String, String> metadata = new HashMap<>();
    
    public Map>String, String> getMetadata() {
        return metadata;
    }
    
    public void setMetadata(Map<String, String> metadata) {
        this.metadata = metadata;
    }
}
```

