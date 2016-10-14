## 1 Binder机制：驱动与数据传输

Android中的Binder机制涉及Java应用层、Native应用层（libbinder、C++）和Linux内核层（binder driver，C），其中Java层是对C++ Native层的包装，而应用层进程之间的通信则最终通过内核中的驱动Binder dirver完成。这种跨进程调用，进程之间的数据必须放在内核空间然后等待。

![image](http://img.my.csdn.net/uploads/201610/14/1476404610_9170.png)

Binder机制作为Android的进程间通信方式(IPC)，采用mmap共享内存方式，应用层直接从与kernel共享的缓冲区读取，只产生一次读写提高了数据传输效率。其实质是在Client-Server之间利用虚拟的字符设备/dev/binder作为CS两端的数据传输中介。应用层（Client、Server）的通信是**通过与driver的IO**实现的：Client与Driver设备IO，Server则**轮询**Driver中的数据，并将结果反馈至Driver，Client再从Diver中读取结果。当然这里的Server通信依赖Binder机制，是一个Binder Service。

应用层与内核中的driver之间的IO操作使用统一的接口函数```ioctl(fd,CMD,&bwr)```，通过switch(CMD)进行相应的IO操作，所传输的数据则存放在binder_read_write结构体中。


```
struct binder_write_read {
 binder_size_t write_size;
 binder_size_t write_consumed;
 binder_uintptr_t write_buffer;
 binder_size_t read_size;
 binder_size_t read_consumed;
 binder_uintptr_t read_buffer;
};
```
存放在bwr的buffer中的数据也有格式，即消息ID+binder_transaction_data
```
struct binder_transaction_data {
 union {
 __u32 handle;
 binder_uintptr_t ptr;
 } target;
 binder_uintptr_t cookie;
 __u32 code;
 __u32 flags;
 pid_t sender_pid;
 uid_t sender_euid;
 binder_size_t data_size;
 binder_size_t offsets_size;
 union {
 struct {
 binder_uintptr_t buffer;
 binder_uintptr_t offsets;
 } ptr;
 __u8 buf[8];
 } data;
};
```
上面的binder_transact_data中包括了从应用层传输过来的Parcel类的数据data，以及Binder对象数据。驱动层只负责操作转换Binder相关的数据，对应用层Parcel中的普通数据并不关心。当然如果Parcel中打包了Binder对象，驱动也会进行处理。Binder对象在跨进程传输时，驱动会对其进行转换。

---


## 2 Binder/Service：libbinder中的接口与对象

![image](http://gityuan.com/images/binder/addService/add_media_player_service.png)
*来源：Gityuan* [Binder系列5—注册服务(addService)](http://gityuan.com/2015/11/14/binder-add-service/)

IBinder与Service业务接口INTERFACE之间的关系如上图所示，图中没有指出的是BpMediaPlayerService类对BpBinder的聚合关系
1.  Binder实体对象：BBinder，Binder服务的提供者，Binder机制中提供服务的Service必须继承自BBinder类；
2.  Binder引用对象：BpBinder(handle)，Server的实体对象在Client进程的代表。客户进程拿到handle后，可以
3.  Binder代理对象：BpINTERFACE(BpBinder(handle)):实现了Service服务中的业务接口INTERFACE的接口对象，通过代理对象，Client能够像使用本地对象一样使用远端的实体对象提供的服务。
4.  IBinder对象：BBinder+BpBinder的统称。

图中以MediaPlayerService为例，其业务代码接口IMediaPlayerService提供给实体对象和代理对象相同的业务函数接口。BBinder实体对象MediaPlayerService继承了BnINTERFACE，并重写了onTransact()成员函数作为实际的业务处理过程。在CS模式中，Client获取了Service/Binder引用对象后，构造了BpMPS(BpBinder(handle))，在使用业务函数时，其参数经由驱动传递至Service进程，服务进程轮询driver中的数据、执行业务逻辑并reply。

Client当然可以跳过代理对象，直接通过Binder引用对象BpBinder(handle)使用服务，但是代理对象具备了处理业务的接口，依靠native层的**libbinder框架带来业务逻辑接口的统一**，Client就像使用本地接口一样使用远程的服务接口。

下面可以简单探讨一下Binder机制下服务类的写法。以ExmapleService为例，其业务逻辑接口由IExampleService定义，业务接口为```virtual int getData();```。建立的ExampleService类则实现了业务接口
```
class ExampleService : public BnExampleService
{
public:
    public int getData(){return mData;}
private:
    mData=0x123456;
}

```
相应的，BnExmapleService中的onTransact()：
```
BnExampleService::onTransact(int code, Parcel &data, Parcel* reply)
{
    switch(code)
    {
        case GET_DATA:
            reply->writeInt(this.getData());
    }
    ......
}

```
代理类BpExmaple应当这样实现
```
class BpExampleService:public BpInterface<IExampleService>
{
public:
    virtual int getData()
    {
        remote()->transact(GET_DATA, data, &reply);
        return reply.readInt();
    }
}

```
Client获得代理类BpExample，其引用对象remote=BpBinder(handle)。引用对象的transact()实际上交由IPCThreadState执行，其请求码（函数号）为GET_DATA。Service端收到请求码后，调用```BBinder::transact()```，继而调用```ExampleServic::onTransact()```返回。

---

## 3 Binder对象在驱动中的转换

Binder对象在传输中是跨进程的，其生命周期的管理是一个重点。服务的实体BBinder死亡后，Client进程中的引用对象也应当删除。Client进程中的ProcessState管理本进程中的所有Binder引用类的创建和释放，而**引用对象和实体对象之间的关联**则由驱动负责管理。Driver驱动维护了一颗红黑树，每个进程中的binder_proc结构体都插入树中，每个进程中的binder_proc都保存了Binder对象的node节点表和node_ref引用表。Binder对象的插入和查询就是在这棵树中执行的，

在传输过程中，Binder实体对象和引用对象均使用**flat_binder_object结构体来表示Binder对象**。
```
struct flat_binder_object {
 __u32 type;
 __u32 flags;
 union {
 binder_uintptr_t binder;
 __u32 handle;
 };
 binder_uintptr_t cookie;
};
```
以上type常用BINDER_TYPE_BINDER/BINDER_TYPE_HANDLE标识，flag表示传输方式，union则存储了binder的本地指针**或者**远程引用对象的句柄handle，cookie只在打包BBinder中使用，存放了BBinder的指针。

Driver中的一个重要工作就是**处理传递中的Binder对象**，driver会将flat_binder_object结构体拆开并做相应操作。

```
switch (type) {
    case BINDER_TYPE_BINDER:
    case BINDER_TYPE_WEAK_BINDER:
    ......
    struct binder_node *node = binder_get_node(proc, fp->binder);
    ......
    binder_get_ref_for_node(target_proc, node);
    ...
    if (fp->type == BINDER_TYPE_BINDER)
        fp->type = BINDER_TYPE_HANDLE;
    ......
    fp->handle = ref->desc;
```
当类型为BINDER_TYPE_BINDER时，这是一个Binder实体对象，此时会使用```binder_get_node```在发送进程的Binder对象节点nodes中查找节点，使用```binder_get_ref_for_node```在目标进程中查找引用节点，若无则创建新节点。修改type为handle类型，将节点引用表中的序号赋给handle。

```
case BINDER_TYPE_HANDLE:
case BINDER_TYPE_WEAK_HANDLE: {
    struct binder_ref *ref = binder_get_ref(proc, fp->handle);
    if (ref->node->proc == target_proc) {
        fp->type = BINDER_TYPE_BINDER;
        fp->binder = ref->node->ptr;
        fp->cookie = ref->node->cookie;
    else
        struct binder_ref *new_ref=binder_get_ref_for_node(target_proc, ref->node);
```

当类型为BINDER_TYPE_HANDLE时，这是一个Binder引用对象，首先```binder_get_ref(proc, fp->handle)```根据handle在发送进程的节点引用表中查找引用节点。当目标进程就是Binder实体对象所在进程时，修改type并设置字段，其中cookie存放了BBinder指针。如果不是Binder对象所在进程，则在目标进程中新建一个节点对象的引binder_ref，type不会改动。这个binder_ref数据结构如下：
```
struct binder_ref {
    struct rb_node rb_node_desc;
    struct rb_node rb_node_node;
    struct hlist_node node_entry;
    struct binder_proc *proc;
    struct binder_node *node;
    uint32_t desc;
    int strong;
    int weak;
    struct binder_ref_death *death;
};
```
其中，数值**uint32_desc**就是引用项在红黑树中的序号，即BpBinder中的mHandle。

根据上面的过程，可以简要分析常用场景中Binder对象在整个传输过程中做了怎样的转换：

![image](http://img.my.csdn.net/uploads/201610/14/1476408704_3527.png)
Binder机制中CS两端通信的数据单元是Parcel，Binder对象写入Parcel中时需要使用```writeStrongBinder```函数，读取Binder对象则使用```readStrongBinder```,当然这个过程中Binder对象仍然要用flat_binder_object结构体表示。

当某个Server/Service要向ServiceManager注册时，发送本地的Binder实体对象，写入Parcel中的Binder如下：
```
obj.type = BINDER_TYPE_BINDER;
obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());
obj.cookie = reinterpret_cast<uintptr_t>(local);
```
此Binder对象数据经驱动转换后type=BINDER_TYPE_HANDLE，可以注册进ServiceManager了。而当某个第三方Client想要获取这个Service时，ServiceManager发送给驱动的Binder对象如下
```
obj->type = BINDER_TYPE_HANDLE;
obj->handle = handle;
```
第三方Client进程中并没有这个Binder实体，因此Client进程中新建了一个节点引用这个Binder，拿到了mHandle，后续利用```readStrongBinder```构造了代理对象。

而当这个第三方Client手握handle句柄值，想调用Service所在进程的业务逻辑时，向驱动发送的Binder对象其type=BINDER_TYPE_HANDLE。驱动根据根据handle找到了目标进程，发现Client所引用的Binder对象就在目标进程里，于是改动这个结构体内的type,target和cookie。文章后面会提到，**正是由于这种转换，Service拿到转换后的Binder对象，会执行相应的业务逻辑并返回结果**。

总而言之，CS模型中的Binder对象经由驱动往来穿梭，驱动通过对flat_binder_object结构的操作,实现了Binder对象的不同状态的转变。

---


## 4 ServiceManager做了什么
Client和Service之间的通信，依赖Driver驱动，同时也要依赖ServiceManager在这个通信过程中起到名字查询作用。ServiceManager是一个守护进程，为各类Binder Service提供**名字查询功能，以及返回Binder Service的引用**。ServiceManager同样依赖Binder机制提供服务，其引用句柄handle=0。SM提供给外部的服务主要是注册服务addService(sp<IBinder>)，查询和返回服务getService(String* name)等，其接口为IServiceManager.h。

ServiceManager的主函数没有依赖libbinder框架，而是自建了一个简单的类似原理的bind.c和主函数:
>frameworks\native\cmds\servicemanager\service_manager.c)*,

```
int main(int argc, char **argv)
{
    bs = binder_open(128*1024);
    
    if (binder_become_context_manager(bs)) {
        ALOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }
    ... ...
    selinux_enabled = is_selinux_enabled();
    sehandle = selinux_android_service_context_handle();
    svcmgr_handle = BINDER_SERVICE_MANAGER;
    binder_loop(bs, svcmgr_handler);

    return 0;
}
```

主要步骤是，对/dev/binder设备进行初始化后，将本进程**设置为Binder管理进程，检查权限，最后进入消息循环**```binder_loop(bs, svcmgr_handler)```中轮询Driver中的数据，使用消息处理函数svcmgr_handler处理请求。

ServiceManager维护了一个服务列表svc_list，列表内是svcinfo结构体。注册服务时则将新的Service加入列表，查询/获取服务则将搜索列表返回相应服务的handle：
```
obj->flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
obj->type = BINDER_TYPE_HANDLE;
obj->handle = handle;
obj->cookie = 0;
```
上面的Binder对象obj，是一个flat_binder_object结构体，ServiceManager将obj传输至驱动。


需要指出的是，Binder服务的句柄**handle是一个u_int32数值，只对发起请求的Client端进程和驱动有意义，不同进程中指代相同Binder实体的句柄在数值上可能是不同的**，ServiceManager记录了所有系统service对应的handle，当用户进程需要获取某个系统service的代理时，SMS就会在内部按service名查找到合适的句柄值handle，并“逻辑上”传递给用户进程，于是用户进程会得到一个新的合法句柄值，这个新句柄值可能在数值上和SMS所记录的句柄值不同，然而，它们指代的却是同一个Service实体，句柄的合法性是由Binder驱动保证的。

---

## 5 服务的注册过程
这一段各种资料讲的比较多，常常以MediaPlayerService的注册过程入手分析，在注册过程中，MPS是一个Client，向ServiceManager这个Server注册自己。当注册完毕后，MPS充当一个服务的角色，轮询驱动中的消息。
> frameworks\av\media\mediaserver\main_mediaserver.cpp

```
int main(int argc __unused, char** argv)
{
    sp<ProcessState> proc(ProcessState::self());    
    sp<IServiceManager> sm = defaultServiceManager();      
    MediaPlayerService::instantiate();     
    ProcessState::self()->startThreadPool();    
    IPCThreadState::self()->joinThreadPool();
 }
```

    proc(ProcessState::self())

每个Client进程都要对本进程中Binder引用对象进行生命周期管理，这里打开了/dev/binder驱动,mmap了1MB空间用于binder事务。

    defaultServiceManager()

获取ServiceManager，准确的说是获取serviceManager的代理。通过IInterface的宏展开，转化得到了最终的ServiceManager代理对象：new BpServiceManager(),其成员变量mRemote=BpBinder(0);0这个handle就指向了ServiceManagerService这个服务。这个过程中IInterface的**asInterface()**比较重要。

    MediaPlayerService::instantiate()    

BpServiceManager(BpBinder(0)).addService(Str name,new MPS)，接着进入IServiceManager的业务逻辑，利用了ServiceManager的远程对象(引用对象)BpHandle(0)。接着进入BpBinder.transact过程，实质上还是将任务交给了IPCThreadState:```IPC->transact(mHandle, code, data, reply,flags)```，对于Client来说，所需数据就通过**reply**获取。

每个线程都有一个IPCThreadState，成员变量mProcess保存了ProcessState变量(每个进程只有一个)，mIn用来接收来自Binder设备的数据，mOut用来存储发往Binder设备的数据，默认大小均为256字节。PCThreadState进行transact事务处理分3部分：**errorCheck()数据错误检查，writeTransactionData()写入传输数据，waitForResponse()等待驱动数据响应**。

其中，waitForResponse()是一个死循环，利用```IPC.talkWithDriver()```与驱动之间通信,```IPC.talkWithDriver()```中又调用了```ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)```进入与内核，开始于驱动的通信,ioctl调用了内核中的binder.c。

> kernal\drivers\staging\android\binder.c

进入ioctl后，经由```binder_thread_write()```中的判断分支进入```binder_transaction()```过程。binder_transaction()内容比较多，主要步骤是：

1.  找到代表目标进程的节点，handle=0则指向ServiceManager
2.  搜寻目标线程，使用目标线程中的todo队列
3.  对本次Binder调用事务过程创建binder_transaction结构体，设置相应的数据
4.  在目标进程的缓冲区分配空间，并复制用户进程的数据到内核
5.  **处理传输的Binder对象**，前面已经分析了这个过程
6.  将本次调用的binder_transaction结构体链接到线程的transaction_stack列表中.

binder_thread_write()调用结束后，将继续binder_thread_read()处理当前线程和进程中的todo队列。


下面离开内核，回到应用层。在waitForResponse()这个循环中，除了talkWithDriver()，还调用了IPC.executeCommand(cmd)过程，用于响应驱动传来的指令，处理来自Driver的数据了。**对于Service来说，业务逻辑就在这里执行。**

```
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    switch (cmd) {
    case BR_TRANSACTION:
    {
        if (tr.target.ptr) {
            sp<BBinder> b((BBinder*)tr.cookie);
            error = b->transact(tr.code, buffer, &reply, tr.flags); 
        } 
    ......    
    return result;
}
```
在驱动对Binder对象的转换过程中提到，Client发送的Binder对象进入Service所在的目标进程时，其target、cookier都做了转换。对于经驱动处理后的Binder对象，由于Service进程自身就是Binder实体对象，因此执行了以上的```BBinder->transact()```函数。```BBinder::transact()```函数又调用了Binder实体类的```onTransact()```函数。前面分析过libbinder中的类和接口，Service类继承了BnExampleService并重写了onTransact()，这个onTransact中使用了ServiceManager中的方法体，**业务逻辑就在这里执行**。

既然使用transaction描述过程，表明Binder调用过程是一个事务，需要符合ACID原则。下图中BC_TRANSACTION和BR_TRANSACTION是一个完整的事务过程，BC_REPLY和BR_REPLY是一个完整的事务过程。Client向Server请求的过程是一个同步过程，Client需要等待driver中的reply，在此期间线程阻塞。

![image](http://img.my.csdn.net/uploads/201610/14/1476404609_2669.png)

---

## 6 Binder机制中的多线程
还有两行代码没有分析，下面来看看。
```
ProcessState::self()->startThreadPool();
IPCThreadState::self()->joinThreadPool();
```
第一行看起来是启动了线程池，看看startThreadPool函数怎么实现的：
```
void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);
    }
}
```
对mThreadPoolStarted的判断，表明这个startThreadPool只能启动一次，线程池启动后，这个值为true。接下来看看spawnPooledThread()。
```
void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();
        ALOGV("Spawning new pooled thread, name=%s\n", name.string());
        sp<Thread> t = new PoolThread(isMain);
        t->run(name.string());
    }
}
```
函数中创建了一个PoolThread类，PoolThread.run()则创建了新线程。isMain变量表示这个线程是否是线程池中的第一个。上面提到在Binder事务中还要执行```IPCThreadState::executeCommand(cmd)``响应驱动传来的消息，当驱动传来BR_SPAWN_LOOPER时，就要执行spawnPooledThread(false),这就是驱动传过来的消息，要创建新线程了。

```
case BR_SPAWN_LOOPER:
        mProcess->spawnPooledThread(false);
        break;
```
PoolThread类的执行体是threadLoop()函数，函数里是```IPCThreadState::self()->joinThreadPool(mIsMain)```，也就是说**线程池里的线程都是在执行```joinThreadPool(bool)```**。对于第一个线程，这是由应用层创建的，isMain=true；**其他线程均是驱动告知目标进程创建的，isMain=false。**

该看看线程执行体了：

```
void IPCThreadState::joinThreadPool(bool isMain)
{
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);
    set_sched_policy(mMyThreadId, SP_FOREGROUND);
    status_t result;
    do {
        processPendingDerefs();
        result = getAndExecuteCommand();
        if(result == TIMED_OUT && !isMain) {
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);
    mOut.writeInt32(BC_EXIT_LOOPER);
    talkWithDriver(false);
}

```
这个执行体是一个循环体，使用了两个函数```processPendingDerefs()```和```getAndExecuteCommand()```,前者用来处理IPCThreadState对象中Binder对象的引用计数。后者顾名思义就是读取驱动的指令并执行，内部就是调用了```talkWithDriver()和executeCommand()```。isMain变量写入mOut，告诉驱动本线程已经建立，同时便于驱动区分**应用层主动创建的“主”线程**和**受驱动通知创建的线程**，非主线程在空闲时会被驱动要求退出。

应用层进程创建了多个线程与驱动通信，即有多个线程在调用```ioctl(mProcess->driverFD,cmd,&data)```。而同一个进程中的多个线程共用一个驱动文件描述符driverFD，因此有多个线程在等待同一个驱动缓冲区域返回的数据。在这里，**驱动记录了每次Binder调用时的线程ID，唤醒对应的线程读取缓冲区数据**。在Binder机制中，**由内核驱动来处理应用层线程的创建和唤醒**，这和一般的IO模型不同。

---
## 7 总结
以上内容主要是便于**简单、粗暴、直观地**理解进程间通信的基本原理和基本流程，一般资料中翔实的过程代码细节就不涉及了。

对于进程间通信，最想了解的不外乎最直观的两点：**数据在进程间是怎样流动传输的，客户端是怎样拿到服务端的接口的**，这两点背后的机制在于Binder驱动层的**mmap共享内存、红黑树节点和引用的维护**。理解了以上两点后，剩下的不过是梳理应用层libbinder构建的与底层驱动通信的模型，这其中涉及类与接口的设计、Binder调用事务过程以及多线程。

---
END:
这是一篇笔记，参考了：

[刘超 深入解析Android 5.0系统](https://book.douban.com/subject/26377840/)

[Weishu's Notes :Binder学习指南](http://weishu.me/2016/01/12/binder-index-for-newer/)

[Gityuan #binder](http://gityuan.com/tags/#binder)

[红茶一杯话Binder（ServiceManager篇）](https://my.oschina.net/youranhongcha/blog/149578)

[Glacier的专栏:Android Binder机制](http://blog.csdn.net/coding_glacier/article/details/7520199)