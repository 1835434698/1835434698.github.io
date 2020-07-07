## 一、traceview ：

图形的形式展示执行时间、调用栈等。

信息全面，包含所有线程。

1、使用方式

Debug.startMethodTracing("");

Debug.stopMethodTracing("");

生成文件在SD卡：Android/data/packagename/files



## 二、systrace:

结合Android内核的数据，生成Html报告。

API 18以上使用，推荐TraceCompat。

1、使用方式

​    TraceCompat.beginSection("hahahahha");
​    TraceCompat.endSection();



```bash
$ cd android-sdk/platform-tools/systrace
$ python systrace.py --time=10 -o mynewtrace.html sched gfx view wm app
或者 
$ python systrace.py --time=10 -a com.allin.social -o mynewtrace.html sched gfx view wm app
生成报告

```

