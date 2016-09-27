## android中的消息机制
**Handler、Looper和MessageQueue**
*   Message被包装在Looper中，且Looper是线程相关的，每个线程只有一个Looper
*   UI主线程的启动参见framework中的ActivityThread.java，在main(String[] args)函数中启动了Looper.loop消息循环
*   新建一个Handler对象后，在Handler类的构造函数中会将Handler对象与所在线程关联，因此Handler对应了所在线程的Looper
*   Handler的sendMessage向所关联的消息队列发送message，message内则包含了对应的消息处理的target:Handler
*   Looper.loop循环中，message执行了回调message.target.dispatchMessage(message),进而调用了使用者所Overrdie的Handler.handleMessage()方法

---

---

## **AsyncTask异步框架**

**Usage**

1.  **``` new AsyncTask().execute(Params...params)```** :UI线程中调用，触发异步任务AsyncTask
2.  **``` @Override onPreExecute() ```**:在UI线程中执行，异步任务启动前的准备工作
3.  **``` @Override doInBackground(Params...)```**:在任务子线程中执行，通常用于理耗时任务。在此方法中用户可以自行调用``` publishProgress(Progress...)```，发送``` MESSAGE_POST_PROGRESS```消息至主线程，进而触发主线程中的方法``` onProgressUpdate(Progress...)```
4.  **```@Override onProgressUpdate(Progress...)```**:后台任务调用了``` publishProgress()```后，主线程对消息的回应处理，用于更新UI
5.  **```@Override onPostExecute(Result) ```**:任务子线程执行完毕后触发结束消息的处理方法，收尾工作。

---

**看看源码(API23)**

AsyncTask将Handler、Looper和MessageQueuq进行了包装，用线程池执行后台任务。因此主要来看后台任务**FutureTask**、消息处理**Handler**以及**ThreadPoolExecutor**是如何设计的：

*   **Handler：**

     **InternalHandler**, 其构造函数获取MainLoop，即Handler是和主线程消息循环绑定的。消息处理方法```handleMessage```则对```MESSAGE_POST_RESULT```和```MESSAGE_POST_PROGRESS```消息做出了响应。
     
*   **FutureTask**

    AsyncTask构造函数创建了Callable类型的对象**mWorker**,call()方法则包装了```doInBackGround()```方法，```postResult()```方法得到result并发送MESSAGE_POST_RESULT消息。当然，在doInBackground中则可以调用``` publishProgress(Progress...)```。mWorker用于构造FutureTask类型**mFuture**。mFuture就是任务执行体了。
    
    比较了几个版本的源码之后，我发现API22之后，Handler的创建是在mWorker 的call()方法中的```postResult()```方法中完成的, 采用了单例模式的```getHandler()```创建/获取InternalHandler。早期的版本(Android5.0)则在类的静态成员中就已经创建此Handler对象。尽管这里Handler的创建在子线程，但是由于其构造函数与主线程Looper绑定，因此消息处理仍然在主线程完成。

*   **executors相关**
    
    用户在主线程中调用execute(Params...parms)后，实际上执行了```executeOnExecutor(sDefaultExecutor, params)```，此方法内首先执行```onPreExecute()```，随后则是**执行任务过**程：**```sDefaultExecutor.execute(mFuture)```**。

    再来看看```sDefaultExecutors```, 这是一个```SerialExecutor```类的实例，其```execute()```方法用于维持一个任务队列，并将将Runnable放入mTasks队尾，随后Poll一个task送入```THREAD_POOL_EXECUTOR```执行。也就是说```sDefaultExecutors```只起到维护任务队列的过程，**真正的执行体还是```THREAD_POOL_EXECUTOR```**，这是一个线程池```new ThreadPoolExecutor(...)```
    
    值得注意的是，尽管将任务送入了线程池```THREAD_POOL_EXECUTOR```，但是```SerialExecutor.execute()```方法使用了同步锁，实质上是一个线性入队执行的单线程过程，从类名上也是可以看出来的：```public synchronized void execute(final Runnable r)```，即使多个线程提交任务，也得一个一个来。这也是android3.0之后的变化，旧版本的AsyncTask是支持在线程池中多任务并发的。
    
    线程池的执行过程不再赘述，submit()和execute()最终都是调用了callable的call()方法。call()过程中发送的两类消息则可以被InternalHandler获取并得到相应处理。