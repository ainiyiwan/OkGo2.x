# 研究基类设计

## 好女不二嫁，好码不二写
## talk is cheap, show me the code
## 这些都是可以直接拿来用的

## BaseActivity类研究

### 1.抽象类
```java
public abstract class BaseActivity extends AppCompatActivity{}
```
### 2.通用方法封装
```java
@SuppressWarnings("unchecked")
public <T extends View> T findView(int id) {
    return (T) findViewById(id);
}
```
### 3.项目通用操作封装
```java
 @Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    initSystemBarTint();
}
```
### 4.通用初始化操作
```java
/** 设置状态栏颜色 */
protected void initSystemBarTint() {
```
### 5.各种方法重载及其通用初始化操作
```java
    @Override
    public void setContentView(@LayoutRes int layoutResID) {
        super.setContentView(layoutResID);
        ButterKnife.bind(this);
    }

    @Override
    public void setContentView(View view) {
        super.setContentView(view);
        ButterKnife.bind(this);
    }

    @Override
    public void setContentView(View view, ViewGroup.LayoutParams params) {
        super.setContentView(view, params);
        ButterKnife.bind(this);
    }
```
### 6.能抽取通用方法，坚决抽取，绝对不写第二次
```java
/** 获取主题色 */
public int getColorPrimary() {
    TypedValue typedValue = new TypedValue();
    getTheme().resolveAttribute(R.attr.colorPrimary, typedValue, true);
    return typedValue.data;
}
```
**好文推荐**

**[优秀程序员眼中的整洁代码](http://mp.weixin.qq.com/s/4Z4vRmWz4PocECMNAAtd3A)**
### 7.通用方法抽取
```java
/** 初始化 Toolbar */
public void initToolBar(Toolbar toolbar, boolean homeAsUpEnabled, String title) {
    toolbar.setTitle(title);
    setSupportActionBar(toolbar);
    getSupportActionBar().setDisplayHomeAsUpEnabled(homeAsUpEnabled);
}
```
### 8.通用操作封装
```java
public void showLoading() {
    if (dialog != null && dialog.isShowing()) return;
    dialog = new ProgressDialog(this);
    dialog.requestWindowFeature(Window.FEATURE_NO_TITLE);
    dialog.setCanceledOnTouchOutside(false);
    dialog.setProgressStyle(ProgressDialog.STYLE_SPINNER);
    dialog.setMessage("请求网络中...");
    dialog.show();
}
```

## 基中基 BaseDetailActivity类研究
### 1.通用界面封装？
这个意义不大，因为项目中一般用不到
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    actionBar = getSupportActionBar();
    if (actionBar != null) {
        actionBar.setDisplayShowTitleEnabled(true);
        actionBar.setDisplayHomeAsUpEnabled(true);
        actionBar.setDisplayShowHomeEnabled(true);
    }
    getDelegate().setContentView(R.layout.activity_base);
    Window window = getWindow();
    requestState = (TextView) window.findViewById(R.id.requestState);
    requestHeaders = (TextView) window.findViewById(R.id.requestHeaders);
    responseData = (TextView) window.findViewById(R.id.responseData);
    responseHeader = (TextView) window.findViewById(R.id.responseHeader);
    rootContent = (FrameLayout) window.findViewById(R.id.content);
    onActivityCreate(savedInstanceState);
}
```
### 2.重头戏 子类必须实现的重要方法 抽象方法
```java
protected abstract void onActivityCreate(Bundle savedInstanceState);
```
### 3.通用方法及通用操作封装

## 总结
要想自己去封装一个基类，比如Activity，你首先要做的就是对于Activity特别熟悉，包括生命周期，重要方法等，对RecyclerView的adapter封装也一样，
你需要对于这个类特别熟悉，那些可以抽取出来，那些必须要自己实现，这些说到底都是你对于这个东西特别熟悉，自然可以做到，所以真的是
**基础决定高度**