## 1. 前言

先放在这里，并不知道要干什么

```java
/** Declare a single {@link #copyObject()} method. */
@FunctionalInterface
public interface CopyObject<T> {
    /**
   * Copy this object.
   * When this object is immutable,
   * the implementation of this method may safely return this object.
   *
   * @return a copy of this object.  When this object is immutable,
   *         the returned object can possibly be the same as this object.
   */
    T copyObject();
}
```

