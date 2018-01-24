# Request请求
## BaseRequest类
所有请求的基类
定义各种属性，及其get和set方法
### 1.CacheMode属性
get属性被CacheCall[CacheCall](https://github.com/ainiyiwan/OkGo2.x/blob/master/CacheCall.md)调用
```java
public CacheMode getCacheMode() {
    return cacheMode;
}
```
### 2.AbsCallback属性
get属性被CacheCall[AbsCallback](https://github.com/ainiyiwan/OkGo2.x/blob/master/AbsCallback.md)调用
```java
public AbsCallback getCallback() {
    return mCallback;
}
```
### 3.Converter属性
get属性被CacheCall[Converter](https://github.com/ainiyiwan/OkGo2.x/blob/master/Converter.md)调用
```java
public Converter getConverter() {
    return mConverter;
}
```
### 4.泛型的设计
```java
public abstract class BaseRequest<R extends BaseRequest>{}
```
>其中泛型 R 主要用于属性设置方法后，返回对应的子类型，以便于实现链式调用

不懂，那么先学会用吧，后面好好研究泛型。
```java
public R cacheMode(CacheMode cacheMode) {
    this.cacheMode = cacheMode;
    return (R) this;
}
```
看完应用的，有点明白了，以后基类的设计，可以参照这个。

#### 不同类型方法的重载设计
```java
@SuppressWarnings("unchecked")
public R params(Map<String, String> params, boolean... isReplace) {
    this.params.put(params, isReplace);
    return (R) this;
}

@SuppressWarnings("unchecked")
public R params(String key, String value, boolean... isReplace) {
    params.put(key, value, isReplace);
    return (R) this;
}
```
### 5.抽象方法，供子类实现
```java
/** 根据不同的请求方式和参数，生成不同的RequestBody */
    public abstract RequestBody generateRequestBody();
```
### 6.通用方法的包装（RequestBody）
```java
/** 对请求body进行包装，用于回调上传进度 */
    public RequestBody wrapRequestBody(RequestBody requestBody) {
        ProgressRequestBody progressRequestBody = new ProgressRequestBody(requestBody);
        progressRequestBody.setListener(new ProgressRequestBody.Listener() {
            @Override
            public void onRequestProgress(final long bytesWritten, final long contentLength, final long networkSpeed) {
                OkGo.getInstance().getDelivery().post(new Runnable() {
                    @Override
                    public void run() {
                        if (mCallback != null) mCallback.upProgress(bytesWritten, contentLength, bytesWritten * 1.0f / contentLength, networkSpeed);
                    }
                });
            }
        });
        return progressRequestBody;
    }
```
#### 6.1 ProgressRequestBody封装
典型的[委派设计模式](http://blog.csdn.net/ergouge/article/details/7421256)

典型的回调设计,基本四步，设计回调接口，设置回调listener，回调数据，实现回调接口执行自己的操作。
```java
 /** 回调接口 */
    public interface Listener {
        void onRequestProgress(long bytesWritten, long contentLength, long networkSpeed);
    }
    
    protected Listener listener;     //进度回调接口
    
    public void setListener(Listener listener) {
        this.listener = listener;
    }
    
    if (listener != null) listener.onRequestProgress(bytesWritten, contentLength, networkSpeed);
    
    progressRequestBody.setListener(new ProgressRequestBody.Listener() {
                @Override
                public void onRequestProgress(final long bytesWritten, final long contentLength, final long networkSpeed) {
                    OkGo.getInstance().getDelivery().post(new Runnable() {
                        @Override
                        public void run() {
                            if (mCallback != null) mCallback.upProgress(bytesWritten, contentLength, bytesWritten * 1.0f / contentLength, networkSpeed);
                        }
                    });
                }
            });
```
#### 6.2 回调主线程
说一点，这里所有分析出来的东西都是可以直接重用的。
```java
OkGo.getInstance().getDelivery().post(new Runnable() {
    @Override
    public void run() {
        if (mCallback != null) mCallback.upProgress(bytesWritten, contentLength, bytesWritten * 1.0f / contentLength, networkSpeed);
    }
});

public Handler getDelivery() {
   return mDelivery;
}

private Handler mDelivery;                                  //用于在主线程执行的调度器
mDelivery = new Handler(Looper.getMainLooper());
```

#### 6.3 OkHttp的使用
其实这里牵扯到一个OkHttp的使用问题，[这篇教程](http://blog.csdn.net/iispring/article/details/51661195)写得很详细，当然，最简单明了的当然还是OkHttp官网的[教程](https://square.github.io/okhttp/)

get请求
```java
OkHttpClient client = new OkHttpClient();

String run(String url) throws IOException {
  Request request = new Request.Builder()
      .url(url)
      .build();

  Response response = client.newCall(request).execute();
  return response.body().string();
}
```
post请求
```java
public static final MediaType JSON
    = MediaType.parse("application/json; charset=utf-8");

OkHttpClient client = new OkHttpClient();

String post(String url, String json) throws IOException {
  RequestBody body = RequestBody.create(JSON, json);
  Request request = new Request.Builder()
      .url(url)
      .post(body)
      .build();
  Response response = client.newCall(request).execute();
  return response.body().string();
}
```
### 7.通用方法包装（Call）
```java
/** 获取同步call对象 */
public okhttp3.Call getCall() {
    //构建请求体，返回call对象
    RequestBody requestBody = generateRequestBody();
    mRequest = generateRequest(wrapRequestBody(requestBody));
    return generateCall(mRequest);
}

/** 根据当前的请求参数，生成对应的 Call 任务 */
public okhttp3.Call generateCall(Request request) {
    mRequest = request;
    if (readTimeOut <= 0 && writeTimeOut <= 0 && connectTimeout <= 0 && interceptors.size() == 0) {
        return OkGo.getInstance().getOkHttpClient().newCall(request);
    } else {
        OkHttpClient.Builder newClientBuilder = OkGo.getInstance().getOkHttpClient().newBuilder();
        if (readTimeOut > 0) newClientBuilder.readTimeout(readTimeOut, TimeUnit.MILLISECONDS);
        if (writeTimeOut > 0) newClientBuilder.writeTimeout(writeTimeOut, TimeUnit.MILLISECONDS);
        if (connectTimeout > 0) newClientBuilder.connectTimeout(connectTimeout, TimeUnit.MILLISECONDS);
        if (interceptors.size() > 0) {
            for (Interceptor interceptor : interceptors) {
                newClientBuilder.addInterceptor(interceptor);
            }
        }
        return newClientBuilder.build().newCall(request);
    }
}
```
### 8.同步请求（execute()）
```java
/** 普通调用，阻塞方法，同步请求执行 */
public Response execute() throws IOException {
    return getCall().execute();
}
```
图片来自OkGo2.x的文档
![](https://github.com/ainiyiwan/OkGo2.x/blob/master/picture/execute.jpg)
