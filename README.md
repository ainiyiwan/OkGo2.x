# OkGo2.x
OkGo2.x学习记录：https://github.com/jeasonlzy/okhttp-OkGo/tree/v2.1.4 

项目使用总结见[这里](https://github.com/ainiyiwan/OkGo2.x/blob/master/use/use.md)

## 1.调用init()方法，保存一个context，具体用处？

## 2.初始化的时候调用getInstance()，获取OkGo的单例
```java
    public static OkGo getInstance() {
        return OkGoHolder.holder;
    }

    private static class OkGoHolder {
        private static OkGo holder = new OkGo();
    }
```
这个属于静态内部类的单例设计，看来现在很流行，以后可以用这个。
获取单例的同时，设置默认值，设计框架的时候需要设计这些默认值
```java
    private OkGo() {
        okHttpClientBuilder = new OkHttpClient.Builder();
        okHttpClientBuilder.hostnameVerifier(HttpsUtils.UnSafeHostnameVerifier);
        okHttpClientBuilder.connectTimeout(DEFAULT_MILLISECONDS, TimeUnit.MILLISECONDS);
        okHttpClientBuilder.readTimeout(DEFAULT_MILLISECONDS, TimeUnit.MILLISECONDS);
        okHttpClientBuilder.writeTimeout(DEFAULT_MILLISECONDS, TimeUnit.MILLISECONDS);
        mDelivery = new Handler(Looper.getMainLooper());
    }
```
## 3.调用debug方法
```java
// 打开该调试开关,打印级别INFO,并不是异常,是为了显眼,不需要就不要加入该行
// 最后的true表示是否打印okgo的内部异常，一般打开方便调试错误
.debug("OkGo", Level.INFO, true)
```
初始化自定义的HttpLoggingInterceptor,并且设置给Okhttp,并且判断是否打印OkGo异常
```java
    /**
     * 调试模式,第三个参数表示所有catch住的log是否需要打印
     * 一般来说,这些异常是由于不标准的数据格式,或者特殊需要主动产生的,并不是框架错误,如果不想每次打印,这里可以关闭异常显示
     */
    public OkGo debug(String tag, Level level, boolean isPrintException) {
        HttpLoggingInterceptor loggingInterceptor = new HttpLoggingInterceptor(tag);
        loggingInterceptor.setPrintLevel(HttpLoggingInterceptor.Level.BODY);
        loggingInterceptor.setColorLevel(level);
        okHttpClientBuilder.addInterceptor(loggingInterceptor);
        OkLogger.debug(isPrintException);
        return this;
    }
```
### 3.1 HttpLoggingInterceptor的巧妙设计
日志工具的核心就是log这个方法，所以这个方法一定要封装起来，后面方便替换
```java
    public void log(String message) {
        logger.log(colorLevel, message);
    }
```
相关属性通过get和set设置
```java
    public void setPrintLevel(Level level) {
        printLevel = level;
    }

    public void setColorLevel(java.util.logging.Level level) {
        colorLevel = level;
    }
```
### 3.2OkLogger的巧妙设计
首先是默认值的设置，这对于一个工具来说是尤为重要的，因为我们的用户，可能不会按照我们的想法去初始化，这样设计就避免了崩溃的可能性
这个的tag是默认写死的，如果要看OkGo的异常信息，需要输入filter，OkGo
```java
    private static boolean isLogEnable = true;

    public static String tag = "OkGo";

    public static void debug(boolean isEnable) {
        debug(tag, isEnable);
    }

    public static void debug(String logTag, boolean isEnable) {
        tag = logTag;
        isLogEnable = isEnable;
    }
```
觉得这里还可以优化一下
```java
    public static void debug(){
        debug(tag, isLogEnable);
    }
```
这里你差不多理解了自定义view的几个构造函数的意义。

另外对于相应的属性提供，get和set方法肯定是必不可少的
## 4.连接超时时间设置
```java
 //如果使用默认的 60秒,以下三行也不需要传
 .setConnectTimeout(OkGo.DEFAULT_MILLISECONDS)  //全局的连接超时时间
 .setReadTimeOut(OkGo.DEFAULT_MILLISECONDS)     //全局的读取超时时间
 .setWriteTimeOut(OkGo.DEFAULT_MILLISECONDS)    //全局的写入超时时间
```
## 5.缓存设置
```java
//可以全局统一设置缓存模式,默认是不使用缓存,可以不传,具体其他模式看 github 介绍 https://github.com/jeasonlzy/
.setCacheMode(CacheMode.NO_CACHE)
//可以全局统一设置缓存时间,默认永不过期,具体使用方法看 github 介绍
.setCacheTime(CacheEntity.CACHE_NEVER_EXPIRE)
```
### 5.1CacheMode
Google并不推荐Enum的写法，会占用两倍的内存，这里可以使用静态变量替代。
### 5.2CacheEntity
>使用缓存前，必须让缓存的数据javaBean对象实现Serializable接口，否者会报NotSerializableException。 因为缓存的原理是将对象序列化后直接写入 数据库中，如果不实现Serializable接口，会导致对象无法序列化，进而无法写入到数据库中，也就达不到缓存的效果。
####无论对于哪种缓存模式，都可以指定一个cacheKey，建议针对不同需要缓存的页面设置不同的cacheKey，如果相同，会导致数据覆盖。
重要属性的get和set，以及通用方法的封装，这是每一个类都要做的事情。
### 5.3CacheDao,DataBaseDao,CacheHelper
数据库相关操作，封装的不错，自己也想按照这个封装一个方便时使用的数据库，我想，这就是[litepal](https://github.com/LitePalFramework/LitePal)的初衷吧(郭神)
### 5.4Cachemanager
对于这种enum类的写法不太熟悉，后期需要恶补基础，这个类的作用是读取缓存，并且加锁，所以是线程安全的。
```java
public CacheEntity<Object> get(String key) {
    mLock.lock();
    try {
        return cacheDao.get(key);
    } finally {
        mLock.unlock();
    }
}
```
## 6.重试次数
```java
 //可以全局统一设置超时重连次数,默认为三次,那么最差的情况会请求4次(一次原始请求,三次重连请求),不需要可以设置为0
.setRetryCount(3)
```
调用处 BaseRequest
```java
//超时重试次数
retryCount = go.getRetryCount();
```
BaseRequest的retryCount调用处CacheCall
```java
if (e instanceof SocketTimeoutException && currentRetryCount < baseRequest.getRetryCount()) {
    //超时重试处理
    currentRetryCount++;
    okhttp3.Call newCall = baseRequest.generateCall(call.request());
    newCall.enqueue(this);
}
```
既然如此让我们一探[BaseRequest](https://github.com/ainiyiwan/OkGo2.x/blob/master/BaseRequest.md)和[CacheCall](https://github.com/ainiyiwan/OkGo2.x/blob/master/CacheCall.md)的究竟吧

## 7.Cookie管理
```java
//如果不想让框架管理cookie（或者叫session的保持）,以下不需要
//.setCookieStore(new MemoryCookieStore())            //cookie使用内存缓存（app退出后，cookie消失）
.setCookieStore(new PersistentCookieStore())        //cookie持久化存储，如果cookie不过期，则一直有效
```
Cookie的详细介绍见[这里](https://github.com/jeasonlzy/okhttp-OkGo/wiki/Cookie#%E7%A7%91%E6%99%AE%E6%A6%82%E5%BF%B5)
## 8.设置https的证书
```java
.setCertificates()                                  //方法一：信任所有证书,不安全有风险
//  .setCertificates(new SafeTrustManager())            //方法二：自定义信任规则，校验服务端证书
//  .setCertificates(getAssets().open("srca.cer"))      //方法三：使用预埋证书，校验服务端证书（自签名证书）
//  //方法四：使用bks证书和密码管理客户端证书（双向认证），使用预埋证书，校验服务端证书（自签名证书）
//  .setCertificates(getAssets().open("xxx.bks"), "123456", getAssets().open("yyy.cer"))//

  //配置https的域名匹配规则，详细看demo的初始化介绍，不需要就不要加入，使用不当会导致https握手失败
//  .setHostnameVerifier(new SafeHostnameVerifier())
```
关于Https，看鸿洋大神的[这篇](http://blog.csdn.net/lmj623565791/article/details/48129405)文章
## 9.设置公共头和公共参数
```java
 //这两行同上，不需要就不要加入
.addCommonHeaders(headers)  //设置全局公共头
.addCommonParams(params);   //设置全局公共参数
```

## 10.福利
关于OkGO的代码结构请看，uml文件夹下的OkGo2.x.mdj文件，请下载StarUML软件观看，StarUML是一个很好用的UML软件，并且可以一直免费用下去

# 完结
