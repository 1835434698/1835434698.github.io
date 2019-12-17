**一、IPC的说明**

IPC是Inter-Process Communication的缩写，含义为进程间通信或跨进程通信，是指两个进程之间进行数据交换的过程。

IPC不是Android独有的，任何一个操作系统都需要有相应的IPC机制，比如Windows上可以通过剪贴板，管道和邮槽来进行进程间通信；Linux上可以通过命名管道、共享内容、信号量等进行进程间通信。

对于Android来说，它是一种基于Linux内核的移动操作系统，但它的进程间通信方式并不能完全继承自Linux；相反，它有自己的进程间通信方式。

在Android中可以通过Binder轻松的实现进程间通信。Android还支持Socket，通过Socket可以实现任意两个终端之间的通信，同一设备的两个进程通过Socket通信自然也是可以的。

 

**二、Android中IPC的使用**

首先，只有面对多进程的情况才需要考虑进程间通信。多进程的情况分为两种：

第一种是一个应用因为某些原因，自身需要采用多进程模式来实现，比如有些模块需要运行在单独的进程中、或者为了加大一个应用的可使用内存，需要通过多进程来获取多分内存空间。

另一种是当前应用需要向其他的应用获取数据，所以必须采用跨进程的方式来获取数据，比如使用系统的ContentProvider去查询数据。

通过给四大组件在AndroidMenifest中指定android:process属性，可以轻易地开启多进程模式。

 

**三、Android中IPC带来的问题**

两个应用共享数据：Android系统会为每个应用分配一个唯一的UID，具有相同UID的应用才能共享数据。两个应用通过ShareUID跑在同一个进程是有要求的，需要这两个应用有相同的ShareUID并且签名相同才可以。在这种情况下，他们可以相互访问对方的私有数据，比如data目录，组件信息等，不管他们是否跑在同一个进程。

Android系统为每个应用分配了一个独立的虚拟机，或者说为每一个进程都分配一个独立的虚拟机，不同的虚拟机在内存分配上有不同的地址空间，这就导致在不同的虚拟机中访问同一个对象会产生多分副本。所有运行在不同进程中的四大组件，只要它们之间需要通过内存来共享数据，都会共享失败，这也是多进程带来的主要影响。

一般来说，使用多进程会造成如下的问题：

（1）静态成员和单例模式完全失效（不同的虚拟机中访问同一个对象会产生多分副本）

（2）线程同步机制完全失效（不在同一块内存，不管是所对象还是锁全局类都无法保证线程同步）

（3）SharePreferences的可靠性下降（不支持两个进程同时写操作）

（4）Application会多次创建（因为创建新进程会分配独立虚拟机，相当于启动一个新的应用）

虽说不能直接的共享内存，但是通过跨进程通信还是可以实现数据交互。

 

**四、Android中各种IPC方式**

1、使用Bundle

四大组件中三大组件Activity、Service、Receiver都支持在Intent中传递Bundle数据。

由于Bundle实现了Parcelable接口，所以它可以很方便的在不同的进程间传输数据。当然我们传输的数据必须能够被序列化，比如基本类型、实现了Parcelable接口的对象、实现了Serializable接口的对象以及一些Android支持的特殊对象。

2、使用文件共享

两个进程通过读写同一个文件来交换数据，比如A进程把数据写入文件，B进程通过读取这个文件来获取数据。

Android系统基于Linux，使得并发读写文件可以没有限制的进行，甚至两个线程同时对文件读写操作都是允许的，尽管可能出问题，因此文件共享方式适合在对数据同步要求不高的进程间进行通信。

SharedPreferences也属于文件的一种，但是由于系统对它的读写有一定的缓存策略，即在内存中会有一份SharedPreferences文件的缓存；因此在多进程模式下，系统对它的读写就变得不可靠，会有很大几率丢失数据，不建议在进程间通信中使用SharedPreferences。

3、使用Messenger

Messenger可以理解为信使，通过它可以再不同进程中传递Message对象，在Message中放入我们需要传递的数据，就可以实现数据的进程间传递了。

Messenger是一种轻量级的IPC方案，它的底层实现是AIDL。由于它一次处理一个请求，因此在服务端不需要考虑线程同步的问题，因为服务端不存在并发执行的情形。

4、使用AIDL

AIDL是 Android Interface definition language的缩写，它是一种android内部进程通信接口的描述语言。AIDL可以处理发送到服务器端大量的并发请求（不同与Messenger的串行处理方式），也可以实现跨进程的方法调用。

在Android中使用方法：创建一个Service和一个AIDL接口，接着创建一个类继承自AIDL接口中的Stub类并实现Stub中的抽象方法，在Service的onBind方法中返回这个类的对象，然后客户端绑定服务端Service，建立连接后就可以访问远程服务器了。

5、使用ContentProvider

ContentProvider是Android中提供的专门用于不同应用间进行数据共享的方式，天生适合进程间通信。

ContentProvider的底层实现也是Binder，但是它的使用过程比AIDL简单的多，因为系统做了封装，使得无需关心细节即可轻松实现IPC。ContentProvider主要以表格的形式组织数据，和数据库很类似，但ContentProvider对底层的数据存储方式没有任何要求，既可以使用Sqlite数据库，也可以使用文件方式，甚至可以使用内存中的一个对象来存储。

6、使用Socket

Socket套接字，是网络通信中的概念，分为流式套接字和用户数据奥套接字两种，对应于网络的传输控制层中的TCP和UDP协议。

两个进程可以通过Socket来实现信息的传输，Socket本身可以支持传输任意字节流。

===选择合适的IPC方式===，如下表

| 名称            | 优点                                                         | 缺点                                                         | 适用场景                                         |
| --------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------ |
| Bundle          | 简单易用                                                     | 只能传输Bundle支持的数据类型                                 | 四大组件的进程间通信                             |
| 文件共享        | 简单易用                                                     | 不适合高并发场景，并且无法做到进程间即时通信                 | 无并发访问清醒，交换简单的数据，实时性不搞的场景 |
| AIDL            | 功能强大，支持一对多并发通信，支持实时通信                   | 使用稍复杂，需要处理好线程同步                               | 一对多通信且有RPC需求                            |
| Messenger       | 功能一般，支持一对多串行通信，支持实时通信                   | 不能很好的处理高并发情形，不支持RPC，数据通过Message进行传输，因此只能传输Bundle支持的数据类型 | 低并发的一对多即时通信，无RPC需求                |
| ContentProvider | 在数据源访问方面功能强大，支持一对多并发数据共享，可通过Call方法扩展其他操作 | 可以理解为受约束的AICL，主要提供数据的CRUD数据               | 一对多的进程间数据共享                           |
| Socket          | 功能强大，可以通过网络传输字节流，支持一对多并发实时通信     | 实现细节稍微繁琐，不支持直接的RPC                            | 网络数据交换                                     |

RPC(Remote Procedure Call，远程过程调用)是种C/S的编程模式，出于一种类比的愿望，在一台机器上运行的主程序，可以调用另一台机器上准备好的子程序。

通过RPC可以充分利用非共享内存的多处理器环境，可以简便地将你的应用分布在多台工作站上，应用程序就像运行在一个多处理器的计算机上一样。

 

 **五、Binder介绍**

Binder是Android系统进程间通信方式之一。Linux已经拥有的进程间通信IPC手段包括： 管道（Pipe）、信号（Signal）、跟踪（Trace）、插口（Socket）、报文队列（Message）、共享内存（Share Memory）和信号量（Semaphore）。

Binder框架定义了四个角色：Server，Client，ServiceManager以及Binder驱动。

其中Server，Client，ServiceManager运行于用户空间，驱动运行于内核空间。Binder就是一种把这四个组件粘合在一起的粘结剂了，其中，核心组件便是Binder驱动程序了，Service Manager提供了辅助管理的功能，Client和Server正是在Binder驱动和Service Manager提供的基础设施上，进行Client-Server之间的通信。这四个角色的关系和互联网类似：Server是服务器，Client是客户终端，ServiceManager是域名服务器（DNS），驱动是路由器。

Binder的理解：

1、从IPC角度来说，Binder是Android中的一种跨进程通信方式，该方式在Linux中没有；

2、从Android Framework角度来说，Binder是ServiceManager连接各种Manager和相应ManagerService的桥梁；

3、从Android应用层来说，Binder是客户端和服务端进行通许in的媒介，当BindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个Bind而对象，客户端就可以获取服务端提供的服务或者数据，这里的服务包括普通服务和基于AIDL的服务。

 

在Android开发中，Binder主要用在Service中，包括AIDL和Messenger，普通服务中的Binder不涉及进程间通信。

直观来讲，Binder是Android中的一个类，实现了IBinder接口。

Binder的工作机制：

![img](C:\Users\Administrator\AppData\Local\YNote\data\qq432337FEDCB6EB5443353EB50AE37219\dd8a7faf64e44c8ca2d498e167fc3bcd\4-1621501459.png)

当客户端发起远程请求时，由于当前线程会被挂起直至服务端进程返回数据，所以一个远程方法是很耗时的，那么不能在UI线程中发起此远程请求；

由于服务端的Binder方法运行在Binder的线程池中，所以Binder方法不管是否耗时都应该采用同步的方式去实现，因为它已经运行在一个线程中了。