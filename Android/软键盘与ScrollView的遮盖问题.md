# 软键盘与ScrollView的遮盖问题

## 一、背景介绍

神一般的UI设计出神一般的交互，如下图，一个底部抽屉，抽屉有最低高度（不能完全消失）和最高高度，抽屉内部有滚动的内容（ScrollView），里面有edittext输入框。要求一、弹起键盘的时候文本框完全显示；要求二、抽屉最低高度时，后面布局可操作、可点击。

<img src="/Users/allin1477/Documents/ws/AndroidStudio/mine/1835434698.github.io/Android/Data/typora-user-images/image-20210423173736034.png" alt="image-20210423173736034" style="zoom: 33%;" />

## 二、解决思路。

1、自定义一个ViewGroup设置软键盘弹起模式android:windowSoftInputMode="adjustResize|stateHidden" 、adjustPan等均达到令人无法接受的地步。因为有ScrollView，ScrollView完全显示的时候与ScrollView不完全显示的时候软键盘弹起情况还不一样。呵呵，放弃。

2、使用BottomSheetDialog设置软键盘弹起模式android:windowSoftInputMode，勉强可以接受，软键盘问题，但是下面一层view完全不能操作、点击，因为这就是一个dialog，有自己的windows处理了所有的touchu事件。不能接受。

3、还是好好使用自定义ViewGroup吧。迅速思考，依稀记得有一个windowSoftInputMode是软键盘可以悬浮而不会顶起任何布局。这样就可以完全自己控制布局文件的位置了。信心。

## 三、动手解决

1、这里只介绍成功的3，就不介绍失败的1、2了。

2、果断设置android:windowSoftInputMode="adjustNothing"，运行。完美，布局完全没有被顶起。接下来只需要获取到软键盘弹起的监听设置ScrollView的android:layout_marginBottom 就可以了。但是尝试发现，what，addOnLayoutChangeListener监听怎么无效了？换一种监听，使用addOnGlobalLayoutListener已然无效。what？what？what？

3、打开边界布局，弹起软键盘，收起软键盘，应该是这种模式下软键盘有了自己的层级，导致监听无效了。好吧，仿佛看到了绝望。

4、继续找软键盘弹起的回调，茫茫大海，总有一个可以使用的回调吧。度娘一下吧。有人借助用popwind获取到了软键盘的回调，果断下载源码运行，尝试。有回调，看源码，嗯，可行。

5、拷贝有用的类到项目中，OK了



```java

    private KeyboardHeightProvider keyboardHeightProvider;

	onCreate(){
        keyboardHeightProvider = new KeyboardHeightProvider((Activity) mContext);
    // 一定要这么写。
        llRootView.post(() -> keyboardHeightProvider.start());
  }

    public void onPause() {
        Log.i(TAG, "onPause");
        keyboardHeightProvider.setKeyboardHeightObserver(null);
    }
    /**
     * {@inheritDoc}
     */
    public void onResume() {
        Log.i(TAG, "onResume");
        keyboardHeightProvider.setKeyboardHeightObserver(this);
    }
    public void onDestroy() {
        Log.i(TAG, "onDestroy");
        keyboardHeightProvider.close();
    }

    @Override
    public void onKeyboardHeightChanged(int height, int orientation) {
        String or = orientation == Configuration.ORIENTATION_PORTRAIT ? "portrait" : "landscape";
        Log.i(TAG, "onKeyboardHeightChanged in pixels: " + height + " " + or);

        if(height>0) {
            LogUtil.d(TAG, "软键盘显示");
            vBottom.setVisibility(VISIBLE);//设置了一个view代替了设置layout_marginBottom
            scrollView.fullScroll(ScrollView.FOCUS_DOWN);
            etOpinion.setFocusable(true);//获取焦点不能少
            etOpinion.setFocusableInTouchMode(true);
            etOpinion.requestFocus();
        }else {
            LogUtil.d(TAG, "软键盘隐藏");
            vBottom.setVisibility(GONE);
        }

    }





/**
 * The observer that will be notified when the height of 
 * the keyboard has changed
 */
public interface KeyboardHeightObserver {

    /** 
     * Called when the keyboard height has changed, 0 means keyboard is closed,
     * >= 1 means keyboard is opened.
     * 
     * @param height        The height of the keyboard in pixels
     * @param orientation   The orientation either: Configuration.ORIENTATION_PORTRAIT or 
     *                      Configuration.ORIENTATION_LANDSCAPE
     */
    void onKeyboardHeightChanged(int height, int orientation);
}

/**
 * The keyboard height provider, this class uses a PopupWindow
 * to calculate the window height when the floating keyboard is opened and closed.
 */
public class KeyboardHeightProvider extends PopupWindow {

    /**
     * The tag for logging purposes
     */
    private final static String TAG = "sample_KeyboardHeightProvider";

    /**
     * The keyboard height observer
     */
    private KeyboardHeightObserver observer;

    /**
     * The cached landscape height of the keyboard
     */
    private int keyboardLandscapeHeight;

    /**
     * The cached portrait height of the keyboard
     */
    private int keyboardPortraitHeight;

    /**
     * The view that is used to calculate the keyboard height
     */
    private View popupView;

    /**
     * The parent view
     */
    private View parentView;

    /**
     * The root activity that uses this KeyboardHeightProvider
     */
    private Activity activity;

    /**
     * Construct a new KeyboardHeightProvider
     *
     * @param activity The parent activity
     */
    public KeyboardHeightProvider(Activity activity) {
        super(activity);
        this.activity = activity;

        LayoutInflater inflator = (LayoutInflater) activity.getSystemService(Activity.LAYOUT_INFLATER_SERVICE);
        this.popupView = inflator.inflate(R.layout.dcm_soft_inputpopupwindow, null, false);
        setContentView(popupView);

        setSoftInputMode(LayoutParams.SOFT_INPUT_ADJUST_RESIZE | LayoutParams.SOFT_INPUT_STATE_ALWAYS_VISIBLE);
        setInputMethodMode(PopupWindow.INPUT_METHOD_NEEDED);

        parentView = activity.findViewById(android.R.id.content);

        setWidth(0);//  这样既能测量高度，又不会导致界面不能点击
//        setWidth(LayoutParams.MATCH_PARENT);
        setHeight(LayoutParams.MATCH_PARENT);

        popupView.getViewTreeObserver().addOnGlobalLayoutListener(new OnGlobalLayoutListener() {

            @Override
            public void onGlobalLayout() {
                if (popupView != null) {
                    handleOnGlobalLayout();
                }
            }
        });
    }

    /**
     * Start the KeyboardHeightProvider, this must be called after the onResume of the Activity.
     * PopupWindows are not allowed to be registered before the onResume has finished
     * of the Activity.
     */
    public void start() {

        if (!isShowing() && parentView.getWindowToken() != null) {
            setBackgroundDrawable(new ColorDrawable(0));
            showAtLocation(parentView, Gravity.NO_GRAVITY, 0, 0);
        }
    }

    /**
     * Close the keyboard height provider,
     * this provider will not be used anymore.
     */
    public void close() {
        this.observer = null;
        this.activity = null;
        dismiss();
    }

    /**
     * Set the keyboard height observer to this provider. The
     * observer will be notified when the keyboard height has changed.
     * For example when the keyboard is opened or closed.
     *
     * @param observer The observer to be added to this provider.
     */
    public void setKeyboardHeightObserver(KeyboardHeightObserver observer) {
        this.observer = observer;
    }

    /**
     * Get the screen orientation
     *
     * @return the screen orientation
     */
    private int getScreenOrientation() {
        return activity.getResources().getConfiguration().orientation;
    }

    /**
     * Popup window itself is as big as the window of the Activity.
     * The keyboard can then be calculated by extracting the popup view bottom
     * from the activity window height.
     */
    private void handleOnGlobalLayout() {

        Point screenSize = new Point();
        activity.getWindowManager().getDefaultDisplay().getSize(screenSize);

        Rect rect = new Rect();
        popupView.getWindowVisibleDisplayFrame(rect);

        // REMIND, you may like to change this using the fullscreen size of the phone
        // and also using the status bar and navigation bar heights of the phone to calculate
        // the keyboard height. But this worked fine on a Nexus.
        int orientation = getScreenOrientation();
        int keyboardHeight = screenSize.y - rect.bottom;

        if (keyboardHeight == 0) {
            notifyKeyboardHeightChanged(0, orientation);
        } else if (orientation == Configuration.ORIENTATION_PORTRAIT) {
            this.keyboardPortraitHeight = keyboardHeight;
            notifyKeyboardHeightChanged(keyboardPortraitHeight, orientation);
        } else {
            this.keyboardLandscapeHeight = keyboardHeight;
            notifyKeyboardHeightChanged(keyboardLandscapeHeight, orientation);
        }
    }

    /**
     *
     */
    private void notifyKeyboardHeightChanged(int height, int orientation) {
        if (observer != null) {
            observer.onKeyboardHeightChanged(height, orientation);
        }
    }
}
```