# CacheCall 带缓存的请求

### 1. 定义Okhttp请求所需的各种参数
```java
private volatile boolean canceled;
private boolean executed;
private BaseRequest baseRequest;
private okhttp3.Call rawCall;
private CacheEntity<T> cacheEntity;
private AbsCallback<T> mCallback;
```
### 2.异步请求(重点) public void execute(AbsCallback<T> callback)
这里需要回调[AbsCallback](https://github.com/ainiyiwan/OkGo2.x/blob/master/AbsCallback.md)
#### 2.1防止并发产生
```java
synchronized (this) {
    if (executed) throw new IllegalStateException("Already executed.");
    executed = true;
}
```
这里通过synchronized同步，防止并发的产生
#### 2.2 mCallback.onBefore(baseRequest);
请求执行前UI线程调用
#### 2.3请求之前获取缓存信息，添加缓存头和其他的公共头
```java
if (baseRequest.getCacheKey() == null)
    baseRequest.setCacheKey(HttpUtils.createUrlFromParams(baseRequest.getBaseUrl(), baseRequest.getParams().urlParamsMap));
if (baseRequest.getCacheMode() == null) baseRequest.setCacheMode(CacheMode.NO_CACHE);
```
#### 2.4无缓存模式,不需要进入缓存逻辑
```java
final CacheMode cacheMode = baseRequest.getCacheMode();
if (cacheMode != CacheMode.NO_CACHE) {
    //noinspection unchecked
    cacheEntity = (CacheEntity<T>) CacheManager.INSTANCE.get(baseRequest.getCacheKey());
    //检查缓存的有效时间,判断缓存是否已经过期
    if (cacheEntity != null && cacheEntity.checkExpire(cacheMode, baseRequest.getCacheTime(), System.currentTimeMillis())) {
        cacheEntity.setExpire(true);
    }
    HeaderParser.addCacheHeaders(baseRequest, cacheEntity, cacheMode);
}
```
#### 2.5构建请求
```java
RequestBody requestBody = baseRequest.generateRequestBody();
final Request request = baseRequest.generateRequest(baseRequest.wrapRequestBody(requestBody));
rawCall = baseRequest.generateCall(request);
```
#### 2.6如果缓存不存在才请求网络，否则使用缓存
```java
 if (cacheMode == CacheMode.IF_NONE_CACHE_REQUEST) {
    //如果没有缓存，或者缓存过期,就请求网络，否者直接使用缓存
    if (cacheEntity != null && !cacheEntity.isExpire()) {
        T data = cacheEntity.getData();
        HttpHeaders headers = cacheEntity.getResponseHeaders();
        if (data == null || headers == null) {
            //由于没有序列化等原因,可能导致数据为空
            sendFailResultCallback(true, rawCall, null, OkGoException.INSTANCE("没有获取到缓存,或者缓存已经过期!"));
        } else {
            sendSuccessResultCallback(true, data, rawCall, null);
            return;//获取缓存成功,不请求网络
        }
    } else {
        sendFailResultCallback(true, rawCall, null, OkGoException.INSTANCE("没有获取到缓存,或者缓存已经过期!"));
    }
}
```
#### 2.7先使用缓存，不管是否存在，仍然请求网络
```java
else if (cacheMode == CacheMode.FIRST_CACHE_THEN_REQUEST) {
    //先使用缓存，不管是否存在，仍然请求网络
    if (cacheEntity != null && !cacheEntity.isExpire()) {
        T data = cacheEntity.getData();
        HttpHeaders headers = cacheEntity.getResponseHeaders();
        if (data == null || headers == null) {
            //由于没有序列化等原因,可能导致数据为空
            sendFailResultCallback(true, rawCall, null, OkGoException.INSTANCE("没有获取到缓存,或者缓存已经过期!"));
        } else {
            sendSuccessResultCallback(true, data, rawCall, null);
        }
    } else {
        sendFailResultCallback(true, rawCall, null, OkGoException.INSTANCE("没有获取到缓存,或者缓存已经过期!"));
    }
}
```
#### 2.8请求是否取消
```java
if (canceled) {
    rawCall.cancel();
}

private volatile boolean canceled;
```
通过volatile关键字保证canceled的可见性

#### 2.9执行OkHttp的异步请求call.enqueue
```java
 @Override
public void onFailure(okhttp3.Call call, IOException e) {
    if (e instanceof SocketTimeoutException && currentRetryCount < baseRequest.getRetryCount()) {
        //超时重试处理
        currentRetryCount++;
        okhttp3.Call newCall = baseRequest.generateCall(call.request());
        newCall.enqueue(this);
    } else {
        mCallback.parseError(call, e);
        //请求失败，一般为url地址错误，网络错误等,并且过滤用户主动取消的网络请求
        if (!call.isCanceled()) {
            sendFailResultCallback(false, call, null, e);
        }
    }
}
```
如果失败，则重试次数+1，如果重试次数小于指定次数，继续请求。
#### 2.10请求成功
```java
int responseCode = response.code();
//304缓存数据
if (responseCode == 304 && cacheMode == CacheMode.DEFAULT) {
    if (cacheEntity == null) {
        sendFailResultCallback(true, call, response, OkGoException.INSTANCE("服务器响应码304，但是客户端没有缓存！"));
    } else {
        T data = cacheEntity.getData();
        HttpHeaders headers = cacheEntity.getResponseHeaders();
        if (data == null || headers == null) {
            //由于没有序列化等原因,可能导致数据为空
            sendFailResultCallback(true, call, response, OkGoException.INSTANCE("没有获取到缓存,或者缓存已经过期!"));
        } else {
            sendSuccessResultCallback(true, data, call, response);
        }
    }
    return;
}
//响应失败，一般为服务器内部错误，或者找不到页面等
if (responseCode == 404 || responseCode >= 500) {
    sendFailResultCallback(false, call, response, OkGoException.INSTANCE("服务器数据异常!"));
    return;
}
```
相应状态的处理
```java
try {
    Response<T> parseResponse = parseResponse(response);
    T data = parseResponse.body();
    //网络请求成功，保存缓存数据
    handleCache(response.headers(), data);
    //网络请求成功回调
    sendSuccessResultCallback(false, data, call, response);
} catch (Exception e) {
    //一般为服务器响应成功，但是数据解析错误
    sendFailResultCallback(false, call, response, e);
}
```
如果请求成功，更新缓存数据。


### 最后
下面的方法都很直观，就不分析了。