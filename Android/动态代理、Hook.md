# 动态代理

## 1、代理的对象是一个接口并且是某个类的成员变量。

这里举例：PopupWindow

1、首选获取成员变量的真名。

​	其中mWindowManager是别名即让程序员来叫的名字。其真名则是PopupWindow.class.getDeclaredField("mWindowManager")获取到的即对程序来叫的名字。

```java
Field windowManagerField = PopupWindow.class.getDeclaredField("mWindowManager");
```

2、设置setAccessible(true);

```java
windowManagerField.setAccessible(true);
```

3、获取到真正的接口对象mWindowManager

其中**popupWindow**为传递过来的真实对象，不是自己构建的，除非其是静态的。

```java
mWindowManager = windowManagerField.get(popupWindow);
```

4、创建WindowManager的动态代理对象proxy

```java
//创建WindowManager的动态代理对象proxy
Object proxy = Proxy.newProxyInstance(Handler.class.getClassLoader(), new Class[]{WindowManager.class}, this);
```

5、实现InvocationHandler的invoke。

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Logger.d(TAG, "invoke");;
    try {
        //拦截方法mWindowManager.addView(View view, ViewGroup.LayoutParams params);
        if (method != null && method.getName() != null && method.getName().equals("addView")
                && args != null && args.length == 2) {
            Logger.d(TAG, "invoke - addView");;
            //获取WindowManager.LayoutParams，即：ViewGroup.LayoutParams
            WindowManager.LayoutParams params = (WindowManager.LayoutParams) args[1];
            //禁止录屏
            setNoScreenRecord(params);
        }
    } catch (Exception ex) {
        ex.printStackTrace();
    }
    return method.invoke(mWindowManager, args);//mWindowManager是3中获取的
}
```

6、注入动态代理对象proxy

```java
//注入动态代理对象proxy（即：mWindowManager对象由proxy对象来代理）
windowManagerField.set(popupWindow, proxy);
```



## 2、代理对象是一个接口，

举例NotificationManager

1、首选获取成员函数的真名。

```java
Method getService = NotificationManager.class.getDeclaredMethod("getService");
```

2、设置setAccessible(true);

```java
getService.setAccessible(true);
```

3、首选获取NotificationManager对象，因为是唯一，所以可以自己获取。

```java
NotificationManager notificationManager = (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);
```

4、得到被代理对象

```java
sOriginService = getService.invoke(notificationManager);
```

5、得到被代理的类

```java
Class iNotiMngClz = Class.forName("android.app.INotificationManager");
```

6、创建代理类

```java
Object proxyNotiMng = Proxy.newProxyInstance(context.getClass().getClassLoader(), new
        Class[]{iNotiMngClz}, this);
```

7、实现InvocationHandler的invoke。

```java
@Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            Log.d(TAG, "invoke(). method:" + method);
            String name = method.getName();
            Log.d(TAG, "invoke: name=" + name);
            if (args != null && args.length > 0) {
                for (Object arg : args) {
                    Log.d(TAG, "invoke: arg=" + arg);
                }
            }
            // 操作交由 sOriginService 处理，不拦截通知
            return method.invoke(sOriginService, args);
//            return null;
        }
```

8、得到1中获取的方法赋值给的成员变量sService，并且设置setAccessible(true);

```java
Field sServiceField = NotificationManager.class.getDeclaredField("sService");
sServiceField.setAccessible(true);
```

9、设置代理

```javaq
sServiceField.set(notificationManager, proxyNotiMng);
```





```java
        Class<?> WindowManagerImpl = Class.forName("android.view.WindowManagerImpl");
//        WindowManagerImpl.getMethod("getService");//这是获取方法的
        Field field = WindowManagerImpl.getDeclaredField("mGlobal");//这是获取成员变量的
```