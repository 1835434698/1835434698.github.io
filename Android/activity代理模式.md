# activity代理模式
代理模式就是将插件的activity作为一个普通的java类，用一个代理activiy来调用维护其生命周期。
## 1、定义启动activity的action。

public static final String PROXY_VIEW_ACTION = "com.hostapp.demo.VIEW";

```java
    <activity android:name=".ProxyActivity">
        <intent-filter>
            <action android:name="com.hostapp.demo.VIEW" />

            <category android:name="android.intent.category.DEFAULT" />
        </intent-filter>
    </activity>
```
因为是代理，所以用action方式启动
## 2、定义interface IRemoteActivity

```java

public interface IRemoteActivity {
public void onStart();
public void onRestart();
public void onActivityResult(int requestCode, int resultCode, Intent data);
public void onResume();
public void onPause();
public void onStop();
public void onDestroy();
public void onCreate(Bundle savedInstanceState);
public void setProxy(Activity proxyActivity, String dexPath);

/**

获取当前Activity的启动模式
*/
public int getLaunchMode();
}
```

目的是自定义插件activity的生命周期。
## 3、定义BasePluginActivityactivity 插件activity需要继承该类主要

```java
public class BasePluginActivity extends ContextThemeWrapper implements IRemoteActivity {

    private static final String TAG = "Client-BaseActivity";
    
    /**
     * 等同于mProxyActivity，可以当作Context来使用，会根据需要来决定是否指向this<br/>
     * 可以当作this来使用
     */
    protected Activity that;
    protected String dexPath;

//为了无侵入试使用。处理插件中new Intent(this, Test.class)时上下文获取包名为null崩溃问题，继承ContextThemeWrapper，复写getPackageName()方法
   ** public String getPackageName(){**
        **Log.d(TAG, "111111 getPackageName  = "+that.getPackageName());**
        **return that.getPackageName();**
    **}**

    private int launchMode = LaunchMode.STANDARD;
    
    public void setLunchMode(int launchMode) {
        this.launchMode = launchMode;
    }
    
    @Override
    public int getLaunchMode() {
        return launchMode;
    }


    @Override
    public void setProxy(Activity proxyActivity, String dexPath) {
        that = proxyActivity;
        this.dexPath = dexPath;
    }
    
    @Override
    public void onCreate(Bundle savedInstanceState) {

//        super.onCreate(savedInstanceState);
    }

    @Override
    public void onActivityResult(int requestCode, int resultCode, android.content.Intent data) {
    }
    
    @Override
    public void onStart() {
    }
    
    @Override
    public void onRestart() {
    }
    
    @Override
    public void onResume() {
    }
    
    @Override
    public void onPause() {
    }
    
    @Override
    public void onStop() {
    }
    
    @Override
    public void onDestroy() {
    }

//    @Override
    public void setContentView(int layoutResID) {
        that.setContentView(layoutResID);
    }

//    @Override
    public View findViewById(int id) {
        return that.findViewById(id);
    }

//    @Override
    public void startActivity(Intent data) {
        Log.d(TAG, "path startActivity ");
        Intent intent = replaceIntent(data);
        that.startActivity(intent);
    }

//    @Override
    public void startActivityForResult(Intent data, int requestCode) {
        Intent intent = replaceIntent(data);
        that.startActivityForResult(intent, requestCode);
    }

    private Intent replaceIntent(Intent data) {
        Log.d(TAG, "111");
    
        if (!TextUtils.isEmpty(data.getAction())){//判断是否是插件化的action方式启动。（走代理）见方式1
            if (TextUtils.isEmpty(data.getStringExtra(AppConstants.EXTRA_DEX_PATH))) {//判断是否设置了要加载的插件dex路径。默认是当前的，即不设置就用当前。
                data.putExtra(AppConstants.EXTRA_DEX_PATH, dexPath);
            }
            return data;
        }else {
            if (!TextUtils.isEmpty(data.getComponent().getClassName()) && data.getComponent().getClassName().contains(data.getComponent().getPackageName())){//跨组建的正常的intent跳转（不走代理）见方式2
                return data;
            }else {
                Intent intent = new Intent(AppConstants.PROXY_VIEW_ACTION);//一般用于插件内的跳转（走代理）见方式3
                intent.putExtra(AppConstants.EXTRA_DEX_PATH, dexPath);
                Log.d(TAG, "path getClassName = " + data.getComponent().getClassName());
                intent.putExtras(data);
                intent.putExtra(AppConstants.EXTRA_CLASS, data.getComponent().getClassName());
                String userName = intent.getStringExtra("userName");
                Log.d(TAG, " userName = "+userName);
                return intent;
            }
    
        }
    }
    
    @Override
    public Resources getResources() {
        return that.getResources();
    }

//    @Override
    public void finish() {
        that.finish();
    }

//    @Override
    public Intent replaceIntent() {
        return that.getIntent();
    }

    public void onBackPressed() {
        that.onBackPressed();
    }
    
    public final void setResult(int resultCode, Intent data) {
        that.setResult(resultCode, data);
    }

}


见方式1

                Intent intent = new Intent(AppConstants.PROXY_VIEW_ACTION);
                intent.putExtra(AppConstants.EXTRA_DEX_PATH, plugin1DexPath);
                intent.putExtra(AppConstants.EXTRA_CLASS, "com.allin.plugin1.SecondActivity");
                startActivity(intent);
    
                Intent intent = new Intent(AppConstants.PROXY_VIEW_ACTION);

//                intent.putExtra(AppConstants.EXTRA_DEX_PATH, dexPath);
                intent.putExtra(AppConstants.EXTRA_CLASS, "com.allin.plugin2.ActivityA");
                intent.putExtra("username", "baobao");
                startActivity(intent);
见方式2
                Intent intent = new Intent();
                intent.putExtra("userName", "tzy");
                ComponentName componentName = new ComponentName("com.hostapp.demo", "com.hostapp.demo.MainActivity");
                intent.setComponent(componentName);
                startActivity(intent);
见方式3
                Intent intent = new Intent(MainActivity.this, SecondActivity.class);
                intent.putExtra("userName", "tzy1");
                startActivity(intent);
```


## 4、代理activity
```java
public class ProxyActivity extends BaseHostActivity {

    private static final String TAG = "ProxyActivity";
    
    private String mClass;
    
    private IRemoteActivity mRemoteActivity;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mDexPath = getIntent().getStringExtra(AppConstants.EXTRA_DEX_PATH);
        mClass = getIntent().getStringExtra(AppConstants.EXTRA_CLASS);
    
        loadClassLoader();//加载dex文件获取class
        loadResources();//加载资源文件
    
        launchTargetActivity(mClass);//反射获取目标activity并启动。
    }
    
    void launchTargetActivity(final String className) {
        try {
            //反射出插件的Activity对象
            Class<?> localClass = dexClassLoader.loadClass(className);
            Constructor<?> localConstructor = localClass.getConstructor(new Class[] {});
            Object instance = localConstructor.newInstance(new Object[] {});
    
            mRemoteActivity = (IRemoteActivity) instance;
            CJBackStack.getInstance().launch(mRemoteActivity);
    
            //执行插件Activity的setProxy方法，建立双向引用
            mRemoteActivity.setProxy(this, mDexPath);
    
            Bundle bundle = new Bundle();
            mRemoteActivity.onCreate(bundle);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

 }

public class BaseHostActivity extends Activity {
    private AssetManager mAssetManager;
    private Resources mResources;
    private Theme mTheme;

    protected String mDexPath;
    protected ClassLoader dexClassLoader;
    
    protected void loadClassLoader() {
        File dexOutputDir = this.getDir("dex", Context.MODE_PRIVATE);
        final String dexOutputPath = dexOutputDir.getAbsolutePath();
        dexClassLoader = new DexClassLoader(mDexPath, dexOutputPath, null, getClassLoader());
    }
    protected void loadResources() {
        try {
            AssetManager assetManager = AssetManager.class.newInstance();
            Method addAssetPath = assetManager.getClass().getMethod("addAssetPath", String.class);
            addAssetPath.invoke(assetManager, mDexPath);
            mAssetManager = assetManager;
        } catch (Exception e) {
            e.printStackTrace();
        }
        Resources superRes = super.getResources();
        mResources = new Resources(mAssetManager, superRes.getDisplayMetrics(),
                superRes.getConfiguration());
        mTheme = mResources.newTheme();
        mTheme.setTo(super.getTheme());
    }
    
    @Override
    public AssetManager getAssets() {
        return mAssetManager == null ? super.getAssets() : mAssetManager;
    }
    
    @Override
    public Resources getResources() {
        return mResources == null ? super.getResources() : mResources;
    }
    
    @Override
    public Theme getTheme() {
        return mTheme == null ? super.getTheme() : mTheme;
    }

}
```



## 小结
借鉴与《Android插件化开发指南》_包建强，在其基础上做了部分改动，减少了对目标项目的改动。