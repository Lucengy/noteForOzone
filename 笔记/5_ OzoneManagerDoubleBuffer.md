## 1. 前言

## 2. 内部类

### 1. Entry

是对TermIndex和OMClientResponse的封装

```java
  private static class Entry {
    private final TermIndex termIndex;
    private final OMClientResponse response;

    Entry(TermIndex termIndex, OMClientResponse response) {
      this.termIndex = termIndex;
      this.response = response;
    }

    TermIndex getTermIndex() {
      return termIndex;
    }

    OMClientResponse getResponse() {
      return response;
    }
  }
```

### 2. FlushNotifier

也有一个Entry内部类

```java
static class Entry {
      private final CompletableFuture<Integer> future = new CompletableFuture<>();
      private int count;

      private CompletableFuture<Integer> await() {
        count++;
        return future;
      }

      private int complete() {
        Preconditions.assertTrue(future.complete(count));
        return future.join();
      }
 }
```

