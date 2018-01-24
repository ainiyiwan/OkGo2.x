# Request请求
## BaseRequest类
所有请求的基类
定义各种属性，及其get和set方法
###1.CacheMode属性
get属性被CacheCall[CacheCall](https://github.com/ainiyiwan/OkGo2.x/blob/master/CacheCall.md)调用
```java
public CacheMode getCacheMode() {
    return cacheMode;
}
```
