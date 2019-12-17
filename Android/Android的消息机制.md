**Android的消息机制**

主线程Looper从消息队列读取消息，当读完所有消息时，主线程阻塞。子线程往消息队列发送消息，并且往管道文件写数据，主线程即被唤醒，从管道文件读取数据，主线程被唤醒只是为了读取消息，当消息读取完毕，再次睡眠。因此loop的循环并不会对CPU性能有过多的消耗。



思考题： 

1.ViewRootImpl如何验证UI操作是否来自于UI线程？

2.Android系统为什么限制子线程进行UI操作？

3.Handler创建时会采用当前线程的Looper来构建内部的消息循环系统，如果当前线程没有Looper会报什么样的错误？如何解决这个问题？

4.ThreadLoacl的工作原理和使用场景。

**Handler简介**

Handler是Android消息机制的上层接口；通过Handler可以轻松的将一个任务切换到Handler所在的线程中去执行。具体的使用场景如下：

有时候需要在工作线程执行耗时的I/O操作，操作完成后需要在UI上做出一些响应，由于安卓开发			规范的限制工作线程是不能直接访问UI控件的，否则就会触发系统异常。这个时候可以通过在主线程中创建的Handler将更新UI的操作切换到主线程中执行。

本质上来说，Handler并不是专门用于更新UI的，他只是常常被开发者用于更新UI。

**Handler的运行机制**

Android的消息机制主要是指Handler的运行机制。

Handler的运行需要底层MessageQueue和Looper的支撑。

**Message**：消息对象；具体结构，稍后介绍<todo>

**MessageQueue**：消息队列。内部存储一组消息,以队列的形式对外提供插入（*enqueueMessage*）和移除（*next*）的接口；实质上是采用单链表(插入和删除比较有优势)的结构存储消息列表。

**Looper**:消息循环对象。Looper会以无限循环的形式去查找是否有新的消息，有则处理，否则就一直处于wait状态。（Looper怎么保证每个线程只存在一个 ThreadLoacl的相关概念）

![img](C:\Users\Administrator\AppData\Local\YNote\data\qq432337FEDCB6EB5443353EB50AE37219\2b7e907d1a3349a898fa808f00b3c18f\clipboard.png)

Hanlder核心类在UML中的体现

以上的主要的组成单元了解完毕之后，剩下的部分就是详细的了解。Handler是如何有机的将这些整合在一起的。（这里着重源码层面的分析）

**Handler核心类的源码分析**

**构造器部分**

从源码来看Handler具有7个构造器，看起来Handler构造器很复杂，其实不然。实际上Hanlder只有两个真正的构造器，其他的5个均为缺省构造器，即通过this方法来调用两个真实的构造器。

![Hanlder源码中的7个构造器](C:\Users\Administrator\AppData\Local\YNote\data\qq432337FEDCB6EB5443353EB50AE37219\cb1733a231214b07ab926828f66caa12\clipboard.png)

![五个缺省的构造器](C:\Users\Administrator\AppData\Local\YNote\data\qq432337FEDCB6EB5443353EB50AE37219\0ccc9a72d4a74348be26f8be83d94153\clipboard.png)

如上图所示，五个缺省的构造器，旨在为在不同的应用场景下创建Handler提供快捷构造的方案。根据使用场景可以大致分为两种场景：

1，不需要传入Looper对象的主线程；

2，需要传入Looper对象的工作线程（工作线程中，创建Handler进行线程间通信）；

**主线程创建时用到的构造器分析**

![非主线程调用会报异常](C:\Users\Administrator\AppData\Local\YNote\data\qq432337FEDCB6EB5443353EB50AE37219\922c55266be14b81bb90b674b05d582c\clipboard.png)

这里需要注意的是**Looper**类中的静态方法*myLooper*()的实现细节，因为涉及到了面试中常问的**ThreadLocal**的工作原理和使用场景，面试中一定要注意。

![ThreadLocal的使用](C:\Users\Administrator\AppData\Local\YNote\data\qq432337FEDCB6EB5443353EB50AE37219\08dceba08e854b87aefca7ac6ef0f955\clipboard.png)

在这个主线程使用的Handler构造器中不需要传入Looper的原因是因为主线程在应用启动的时候已经为我们创建了Looper对象了。这样的一个话的结论未免有些空洞，下面的代码可以说明这一点。

![主线程Looper的创建时机](C:\Users\Administrator\AppData\Local\YNote\data\qq432337FEDCB6EB5443353EB50AE37219\714fa6565a2148fda773bde798bd62f2\clipboard.png)

![prepareMainLooper的执行逻辑](C:\Users\Administrator\AppData\Local\YNote\data\qq432337FEDCB6EB5443353EB50AE37219\c2a3f56ce29a4196b2c1ff7aee46c3e4\clipboard.png)

![ThreadLocal管理Looper](C:\Users\Administrator\AppData\Local\YNote\data\qq432337FEDCB6EB5443353EB50AE37219\762ad8c610c84d05ae6972bdeb1f9e91\clipboard.png)

ThreadLocal的基本原理？？？

先上结论，分析下Handler中发送消息的代码发现，几乎所有的发送方法都是直接或者间接的调用**setMessageAtTime**方法实现。这里有一个例外方法sendMessageAtFrontOfQueue，该方法的目的是进入队列的头部，因此将msg.when设置为0(正常情况下when为系统启动时间的毫秒数+delayMillis)，通过该操作实现添加到消息队列的队头，除此之外与**setMessageAtTime**方法并无差别；因此这里着重分析该方法如何将消息存入MessageQueue中。

![无论是send Mesage或者post Runable对象都是通过这四个方法进行](C:\Users\Administrator\AppData\Local\YNote\data\qq432337FEDCB6EB5443353EB50AE37219\d2cdc099c56d4d98aa3969245608386a\clipboard.png)

MessageQueue方法对消息的管理是通过单向链表实现消息队列的管理。进入队列的方法是enqueueMessage(Message msg, long when)。实现细节上，第一种情况为插入队首的情况，如当前队列为null、新插入的Message对象when为0或者小于队首的when属性时。具体代码如下图。

![插入队首的情况](C:\Users\Administrator\AppData\Local\YNote\data\qq432337FEDCB6EB5443353EB50AE37219\8d7939e313994f3697954249084edf3b\clipboard.png)

第二种情况是遍历队列，插入when小于next.when的位置或者队尾。具体代码如下图。

![遍历插入相应位置的情况](C:\Users\Administrator\AppData\Local\YNote\data\qq432337FEDCB6EB5443353EB50AE37219\419fd2bb56894f498f6b421366f73fe9\clipboard.png)

至此插入队列的操作介绍完毕。

**消息的轮询机制**

![img](C:\Users\Administrator\AppData\Local\YNote\data\qq432337FEDCB6EB5443353EB50AE37219\493324a3cb1746a4bbded753a3e62537\clipboard.png)

![Looper对象的loop实现](C:\Users\Administrator\AppData\Local\YNote\data\qq432337FEDCB6EB5443353EB50AE37219\1945b4f60b3f498c96417ccbf8f9118c\clipboard.png)

![img](C:\Users\Administrator\AppData\Local\YNote\data\qq432337FEDCB6EB5443353EB50AE37219\b21df36224f043409f0053d1301e4162\clipboard.png)

根据轮询（loop）的代码可以发现，一旦messageQueue的next方法返回null,轮询就结束了，如果next方法不堵塞的话，Looper就无法持续的工作。怎么解决这个问题呢。这里就涉及到Linux中的进程间通信的机制-管道（pipe）。原理：在内存中有一个特殊的文件，这个文件有两个句柄（引用），一个是读取句柄，一个是写入句柄。重点就在与mPtr 这个指针。因此MessageQueue的next方法进行堵塞。

消息队列中没有消息则轮询结束，当调用Handler的发送消息的方法进行消息发送时会唤醒Looper进行消息的轮询，具体消息的分发（Handler的CallBack负责）和处理由Handler（handleMessage）来负责。

![Handler进行消息分发处理](C:\Users\Administrator\AppData\Local\YNote\data\qq432337FEDCB6EB5443353EB50AE37219\5d9dad4f07d84eafa2124891fbec0bfa\clipboard.png)

大致流程介绍完毕。

Ｈａｎｄｌｅｒ这里其实还是只讲到了源码层面，不算真正的底层实现。真正的底层实现涉及到的Ｌｉｎｕｘ知识我还需要进一步的研读。路漫漫其修远兮，吾将上下而求索。