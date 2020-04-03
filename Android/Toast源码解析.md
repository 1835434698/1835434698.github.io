Toast源码解析

```java
Toast.makeText(mApp, "", Toast.LENGTH_SHORT).show();
//最终由NotificationManagerService负责show。

1、首先创建一个Toast
    需要传递一个上下文，但是这个上下文只是用来获取包名、SystemService等。
    public static Toast makeText(Context context, CharSequence text, @Duration int duration) {
        return makeText(context, null, text, duration);
    }

 public static Toast makeText(@NonNull Context context, @Nullable Looper looper,
            @NonNull CharSequence text, @Duration int duration) {
        Toast result = new Toast(context, looper);//实例一个toast  调用001
//得到布局文件。设置时间与展示消息内容。
        LayoutInflater inflate = (LayoutInflater)
                context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        View v = inflate.inflate(com.android.internal.R.layout.transient_notification, null);
        TextView tv = (TextView)v.findViewById(com.android.internal.R.id.message);
        tv.setText(text);

        result.mNextView = v;
        result.mDuration = duration;

        return result;
    }


    /**001
     * Constructs an empty Toast object.  If looper is null, Looper.myLooper() is used.
     * @hide
     */
    public Toast(@NonNull Context context, @Nullable Looper looper) {
        mContext = context;//传递给toast上下文
        mTN = new TN(context.getPackageName(), looper);//创建一个TN 001-1
        //设置位置，对齐方式
        mTN.mY = context.getResources().getDimensionPixelSize(
                com.android.internal.R.dimen.toast_y_offset);
        mTN.mGravity = context.getResources().getInteger(
                com.android.internal.R.integer.config_toastDefaultGravity);
    }

//001-1
 TN(String packageName, @Nullable Looper looper) {
            // XXX This should be changed to use a Dialog, with a Theme.Toast
            // defined that sets up the layout params appropriately.
            final WindowManager.LayoutParams params = mParams;//设置参数
            params.height = WindowManager.LayoutParams.WRAP_CONTENT;
            params.width = WindowManager.LayoutParams.WRAP_CONTENT;
            params.format = PixelFormat.TRANSLUCENT;
            params.windowAnimations = com.android.internal.R.style.Animation_Toast;
            params.type = WindowManager.LayoutParams.TYPE_TOAST;
            params.setTitle("Toast");
            params.flags = WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON
                    | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                    | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;

            mPackageName = packageName;

            if (looper == null) {//得到Looper
                // Use Looper.myLooper() if looper is not specified.
                looper = Looper.myLooper();
                if (looper == null) {
                    throw new RuntimeException(
                            "Can't toast on a thread that has not called Looper.prepare()");
                }
            }
            mHandler = new Handler(looper, null) {//创建Handler处理消息
                @Override
                public void handleMessage(Message msg) {
                    switch (msg.what) {
                        case SHOW: {//展示toast
                            IBinder token = (IBinder) msg.obj;
                            handleShow(token);
                            break;
                        }
                        case HIDE: {//隐藏toast
                            handleHide();
                            // Don't do this in handleHide() because it is also invoked by
                            // handleShow()
                            mNextView = null;
                            break;
                        }
                        case CANCEL: {//返回toast
                            handleHide();
                            // Don't do this in handleHide() because it is also invoked by
                            // handleShow()
                            mNextView = null;
                            try {
                                getService().cancelToast(mPackageName, TN.this);
                            } catch (RemoteException e) {
                            }
                            break;
                        }
                    }
                }
            };
        }

2、show
    /**
     * Show the view for the specified duration.
     */
    public void show() {
        if (mNextView == null) {
            throw new RuntimeException("setView must have been called");
        }

        INotificationManager service = getService();//创建NotificationManager
        String pkg = mContext.getOpPackageName();
        TN tn = mTN;
        tn.mNextView = mNextView;
        final int displayId = mContext.getDisplayId();

        try {
            service.enqueueToast(pkg, tn, mDuration, displayId);//丢给NotificationManager去处理。内部维护了一个handle的发送机制，然后在001-1的handle中的SHOW判断的时候收到处理
        } catch (RemoteException e) {
            // Empty
        }
    }

```