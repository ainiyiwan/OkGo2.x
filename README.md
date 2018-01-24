# OkGo2.x
OkGo2.x学习记录：https://github.com/jeasonlzy/okhttp-OkGo/tree/v2.1.4 

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