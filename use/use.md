# 项目使用篇

## 基类设计研究
见[这里](https://github.com/ainiyiwan/OkGo2.x/blob/master/use/base.md)

## 缓存失败原因 Model的内部类model没有序列化
```java
public class NewsModel implements Serializable {

    private static final long serialVersionUID = 6753210234564872868L;

    public PageBean pagebean;
    public int ret_code;

    public static class PageBean implements Serializable {
        private static final long serialVersionUID = -7782857008135312889L;

        public int allNum;
        public int allPages;
        public int currentPage;
        public int maxResult;
        public List<ContentList> contentlist;
    }
}
```

## 项目中所有的网络请求都会有两遍，因为有一遍是我写的

## 关于缓存模式 FIRST_CACHE_THEN_REQUEST 先使用缓存，不管是否存在，仍然请求网络 
这里如果读取缓存成功，但是网络可能请求失败的话，你们读取完缓存手动直接回调onSuccess

**网络请求失败一定会回调onError，在onError中加一个判断，如果缓存成功，就吐司，如果缓存也失败，就展示失败页面，这个尤为重要**
```java
OkGo.get(Urls.NEWS)//
                .params("channelName", fragmentTitle)//
                .params("page", 1)                              //初始化或者下拉刷新,默认加载第一页
                .cacheKey("TabFragment_" + fragmentTitle)       //由于该fragment会被复用,必须保证key唯一,否则数据会发生覆盖
                .cacheMode(CacheMode.FIRST_CACHE_THEN_REQUEST)  //缓存模式先使用缓存,然后使用网络数据
                .execute(new NewsCallback<NewsResponse<NewsModel>>() {
                    @Override
                    public void onSuccess(NewsResponse<NewsModel> newsResponse, Call call, Response response) {
                        NewsModel newsModel = newsResponse.showapi_res_body;
                        currentPage = newsModel.pagebean.currentPage;
                        newsAdapter.setNewData(newsModel.pagebean.contentlist);
                    }

                    @Override
                    public void onCacheSuccess(NewsResponse<NewsModel> newsResponse, Call call) {
                        //一般来说,只需呀第一次初始化界面的时候需要使用缓存刷新界面,以后不需要,所以用一个变量标识
                        if (!isInitCache) {
                            //一般来说,缓存回调成功和网络回调成功做的事情是一样的,所以这里直接回调onSuccess
                            onSuccess(newsResponse, call, null);
                            isInitCache = true;
                        }
                    }

                    @Override
                    public void onCacheError(Call call, Exception e) {
                        //获取缓存失败的回调方法,一般很少用到,需要就复写,不需要不用关心
                    }

                    @Override
                    public void onError(Call call, Response response, Exception e) {
                        super.onError(call, response, e);
                        //网络请求失败的回调,一般会弹个Toast
                        showToast(e.getMessage());
                    }

                    @Override
                    public void onAfter(@Nullable NewsResponse<NewsModel> newsResponse, @Nullable Exception e) {
                        super.onAfter(newsResponse, e);
                        //可能需要移除之前添加的布局
                        newsAdapter.removeAllFooterView();
                        //最后调用结束刷新的方法
                        setRefreshing(false);
                    }
                });
```
