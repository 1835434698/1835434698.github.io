## WorkManager源码解析

1、首先引入implementation "androidx.work:work-runtime:2.3.1"

注意版本号要与'androidx.appcompat:appcompat:1.1.0'对应，要不然会有版本冲突。这个google做的不太好，官方文档也没有说明。

2、实现Worker类。

复写doWork方法。在其内部可以写自己要运行的代码，代码会运行子线程中。

```java
@NonNull
@Override
public Result doWork() {
    Logger.d(TAG, "doWork name = "+Thread.currentThread().getName());

    return Result.success();//有成功、失败、重试三个方法。并且成功、失败可以携带Data返回。
}


        /**
         * Returns an instance of {@link Result} that can be used to indicate that the work
         * completed successfully. Any work that depends on this can be executed as long as all of
         * its other dependencies and constraints are met.
         *
         * @return An instance of {@link Result} indicating successful execution of work
         */
        @NonNull
        public static Result success() {
            return new Success();
        }

        /**
         * Returns an instance of {@link Result} that can be used to indicate that the work
         * completed successfully. Any work that depends on this can be executed as long as all of
         * its other dependencies and constraints are met.
         *
         * @param outputData A {@link Data} object that will be merged into the input Data of any
         *                   OneTimeWorkRequest that is dependent on this work
         * @return An instance of {@link Result} indicating successful execution of work
         */
        @NonNull
        public static Result success(@NonNull Data outputData) {
            return new Success(outputData);
        }

        /**
         * Returns an instance of {@link Result} that can be used to indicate that the work
         * encountered a transient failure and should be retried with backoff specified in
         * {@link WorkRequest.Builder#setBackoffCriteria(BackoffPolicy, long, TimeUnit)}.
         *
         * @return An instance of {@link Result} indicating that the work needs to be retried
         */
        @NonNull
        public static Result retry() {
            return new Retry();
        }

        /**
         * Returns an instance of {@link Result} that can be used to indicate that the work
         * completed with a permanent failure. Any work that depends on this will also be marked as
         * failed and will not be run. <b>If you need child workers to run, you need to use
         * {@link #success()} or {@link #success(Data)}</b>; failure indicates a permanent stoppage
         * of the chain of work.
         *
         * @return An instance of {@link Result} indicating failure when executing work
         */
        @NonNull
        public static Result failure() {
            return new Failure();
        }

        /**
         * Returns an instance of {@link Result} that can be used to indicate that the work
         * completed with a permanent failure. Any work that depends on this will also be marked as
         * failed and will not be run. <b>If you need child workers to run, you need to use
         * {@link #success()} or {@link #success(Data)}</b>; failure indicates a permanent stoppage
         * of the chain of work.
         *
         * @param outputData A {@link Data} object that can be used to keep track of why the work
         *                   failed
         * @return An instance of {@link Result} indicating failure when executing work
         */
        @NonNull
        public static Result failure(@NonNull Data outputData) {
            return new Failure(outputData);
        }

```

3、将实现的Worker放入OneTimeWorkRequest.Builder(）中创建出builder

```java
 OneTimeWorkRequest.Builder builder = new OneTimeWorkRequest.Builder(UpLoadWorker.class);


//构造builder
        public Builder(@NonNull Class<? extends ListenableWorker> workerClass) {
            super(workerClass);
            mWorkSpec.inputMergerClassName = OverwritingInputMerger.class.getName();//只是获取一个名字
        }

 Builder(@NonNull Class<? extends ListenableWorker> workerClass) {
            mId = UUID.randomUUID();//生成一个id
            mWorkerClass = workerClass;//创建的worker传递给当前类对象
            mWorkSpec = new WorkSpec(mId.toString(), workerClass.getName());//对每一worker创建一个工作空间 ，工作空间中存放一些词worker的设置参数。（调用 003-1）
            addTag(workerClass.getName());//添加到队列 （调用003-2）
        }

//003-1  
    public WorkSpec(@NonNull String id, @NonNull String workerClassName) {
        this.id = id;
        this.workerClassName = workerClassName;
    }
//003-2
  	public final @NonNull B addTag(@NonNull String tag) {
            mTags.add(tag);//一个HashSet
            return getThis();
        }

```

4、创建OneTimeWorkRequest请求体

```java
OneTimeWorkRequest uploadWorkRequest = builder
                .setInitialDelay(Duration.ofHours(20))//设置延迟时间(调用004-1)
                .setInputData(null)//设置输入参数.（调用004-2）
                .build();//(调用004-3)
//004-1
public @NonNull B setInitialDelay(@NonNull Duration duration) {
            mWorkSpec.initialDelay = duration.toMillis();//得到时间设置给3提到的mWorkSpec
            return getThis();
        }
//004-2
public final @NonNull B setInputData(@NonNull Data inputData) {
            mWorkSpec.input = inputData;//设置给3提到的mWorkSpec
            return getThis();
        }
//004-3
public final @NonNull W build() {
            W returnValue = buildInternal();//（调用004-4）
            // Create a new id and WorkSpec so this WorkRequest.Builder can be used multiple times.
            mId = UUID.randomUUID();//生成id
            mWorkSpec = new WorkSpec(mWorkSpec);//生成自己的mWorkSpec
            mWorkSpec.id = mId.toString();//赋值给自己持有的mWorkSpec
            return returnValue;//返回OneTimeWorkRequest
        }
//004-4 检查版本问题。并且创建并且返回OneTimeWorkRequest
OneTimeWorkRequest buildInternal() {
            if (mBackoffCriteriaSet
                    && Build.VERSION.SDK_INT >= 23
                    && mWorkSpec.constraints.requiresDeviceIdle()) {
                throw new IllegalArgumentException(
                        "Cannot set backoff criteria on an idle mode job");
            }
            if (mWorkSpec.runInForeground
                    && Build.VERSION.SDK_INT >= 23
                    && mWorkSpec.constraints.requiresDeviceIdle()) {
                throw new IllegalArgumentException(
                        "Cannot run in foreground with an idle mode constraint");
            }
            return new OneTimeWorkRequest(this);//（调用004-5）
        }
//004-5 
 OneTimeWorkRequest(Builder builder) {
     //设置id、mWorkSpec、mTags列表
        super(builder.mId, builder.mWorkSpec, builder.mTags);
    }


```

5、获取WorkManager 并且放入需要执行的任务。并且开始运行。

```java
WorkManager.getInstance(this).enqueue(uploadWorkRequest);//直接运行，可以是单一，可以是列表。（调用005-9）
workA---->workB---->workC
WorkManager.getInstance(this)//单例获取 （调用005-1）
        .beginWith(workA)//设置从那个工作任务开始（也可以不设置）（调用005-4）
        .then(workB)  //这三行其实就是创建了一个WorkContinuationImpl，是有序的任务。被WorkManagerImpl互相持有
        .then(workC)//
        .enqueue();//开始执行任务（WorkContinuation的enqueue（））（调用005-7）

//005-1
    public static @NonNull WorkManager getInstance(@NonNull Context context) {
        return WorkManagerImpl.getInstance(context);//单例获取 （调用005-2）
    }
//005-2
    @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    public static @NonNull WorkManagerImpl getInstance(@NonNull Context context) {
        synchronized (sLock) {//同步锁
            WorkManagerImpl instance = getInstance();//单例获取 （调用005-3）
            if (instance == null) {//为空
                Context appContext = context.getApplicationContext();
                if (appContext instanceof Configuration.Provider) {
                    initialize(
                            appContext,
                            ((Configuration.Provider) appContext).getWorkManagerConfiguration());//初始化 （调用 005-3）
                    instance = getInstance(appContext);//再次获取单例
                } else {
                    throw new IllegalStateException("WorkManager is not initialized properly.  You "
                            + "have explicitly disabled WorkManagerInitializer in your manifest, "
                            + "have not manually called WorkManager#initialize at this point, and "
                            + "your Application does not implement Configuration.Provider.");
                }
            }

            return instance;
        }
    }

//005-2  两个WorkManagerImpl 如果sDelegatedInstance==null 就返回未锁定的sDefaultInstance
    @Deprecated
    @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    public static @Nullable WorkManagerImpl getInstance() {
        synchronized (sLock) {//再次同步锁
            if (sDelegatedInstance != null) {
                return sDelegatedInstance;
            }

            return sDefaultInstance;
        }
    }

//05-3
    @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    public static void initialize(@NonNull Context context, @NonNull Configuration configuration) {
        synchronized (sLock) {
            if (sDelegatedInstance != null && sDefaultInstance != null) {
                throw new IllegalStateException("WorkManager is already initialized.  Did you "
                        + "try to initialize it manually without disabling "
                        + "WorkManagerInitializer? See "
                        + "WorkManager#initialize(Context, Configuration) or the class level "
                        + "Javadoc for more information.");
            }

            if (sDelegatedInstance == null) {
                context = context.getApplicationContext();
                if (sDefaultInstance == null) {
                    sDefaultInstance = new WorkManagerImpl(
                            context,
                            configuration,
                            new WorkManagerTaskExecutor(configuration.getTaskExecutor()));//创建一个WorkManagerImpl
                }
                sDelegatedInstance = sDefaultInstance;//赋值给了另外一个WorkManagerImpl（005-2 提到的两个WorkManagerImpl）
            }
        }
    }

//005-4
    public final @NonNull WorkContinuation beginWith(@NonNull OneTimeWorkRequest work) {
        return beginWith(Collections.singletonList(work));//（调用005-5）
    }
//005-5
    @Override
    public @NonNull WorkContinuation beginWith(@NonNull List<OneTimeWorkRequest> work) {
        if (work.isEmpty()) {
            throw new IllegalArgumentException(
                    "beginWith needs at least one OneTimeWorkRequest.");
        }
        return new WorkContinuationImpl(this, work);//（调用005-6）
    }
//005-6
    WorkContinuationImpl(
            @NonNull WorkManagerImpl workManagerImpl,
            @NonNull List<? extends WorkRequest> work) {
        this(
                workManagerImpl,
                null,
                ExistingWorkPolicy.KEEP,
                work,
                null);//构造完WorkContinuationImpl
    }

//005-7
    @Override
    public @NonNull Operation enqueue() {
        // Only enqueue if not already enqueued.
        if (!mEnqueued) {
            // The runnable walks the hierarchy of the continuations
            // and marks them enqueued using the markEnqueued() method, parent first.
            EnqueueRunnable runnable = new EnqueueRunnable(this);//创建运行EnqueueRunnable
            mWorkManagerImpl.getWorkTaskExecutor().executeOnBackgroundThread(runnable);//放入线程池，并且开始运行（盗用WorkManagerTaskExecutor的方法 005-8）（EnqueueRunnable的run方法看005-11）
            mOperation = runnable.getOperation();
        } else {
            Logger.get().warning(TAG,
                    String.format("Already enqueued work ids (%s)", TextUtils.join(", ", mIds)));
        }
        return mOperation;
    }
//005-8
    public void executeOnBackgroundThread(Runnable r) {
        mBackgroundExecutor.execute(r);
    }

//005-9
 public final Operation enqueue(@NonNull WorkRequest workRequest) {
        return enqueue(Collections.singletonList(workRequest));//（调用005-10）
    }
//005-10
    public Operation enqueue(
            @NonNull List<? extends WorkRequest> workRequests) {

        // This error is not being propagated as part of the Operation, as we want the
        // app to crash during development. Having no workRequests is always a developer error.
        if (workRequests.isEmpty()) {
            throw new IllegalArgumentException(
                    "enqueue needs at least one WorkRequest.");
        }
        return new WorkContinuationImpl(this, workRequests).enqueue();//回归到005-7
    }

//005-11
    @Override
    public void run() {
        try {
            if (mWorkContinuation.hasCycles()) {
                throw new IllegalStateException(
                        String.format("WorkContinuation has cycles (%s)", mWorkContinuation));
            }
            boolean needsScheduling = addToDatabase();
            if (needsScheduling) {
                // Enable RescheduleReceiver, only when there are Worker's that need scheduling.
                final Context context =
                        mWorkContinuation.getWorkManagerImpl().getApplicationContext();
                PackageManagerHelper.setComponentEnabled(context, RescheduleReceiver.class, true);
                scheduleWorkInBackground();//调用005-12
            }
            mOperation.setState(Operation.SUCCESS);
        } catch (Throwable exception) {
            mOperation.setState(new Operation.State.FAILURE(exception));
        }
    }
//005-12
public void scheduleWorkInBackground() {
        WorkManagerImpl workManager = mWorkContinuation.getWorkManagerImpl();
        Schedulers.schedule(
                workManager.getConfiguration(),
                workManager.getWorkDatabase(),
                workManager.getSchedulers());//调用005-13
    }
//005-13
    public static void schedule(
            @NonNull Configuration configuration,
            @NonNull WorkDatabase workDatabase,
            List<Scheduler> schedulers) {
        if (schedulers == null || schedulers.size() == 0) {
            return;
        }

        WorkSpecDao workSpecDao = workDatabase.workSpecDao();//WorkSpec的设置参数执行。
        List<WorkSpec> eligibleWorkSpecs;

        workDatabase.beginTransaction();
        try {
            eligibleWorkSpecs = workSpecDao.getEligibleWorkForScheduling(
                    configuration.getMaxSchedulerLimit());
            if (eligibleWorkSpecs != null && eligibleWorkSpecs.size() > 0) {
                long now = System.currentTimeMillis();

                // Mark all the WorkSpecs as scheduled.
                // Calls to Scheduler#schedule() could potentially result in more schedules
                // on a separate thread. Therefore, this needs to be done first.
                for (WorkSpec workSpec : eligibleWorkSpecs) {
                    workSpecDao.markWorkSpecScheduled(workSpec.id, now);
                }
            }
            workDatabase.setTransactionSuccessful();
        } finally {
            workDatabase.endTransaction();
        }

        if (eligibleWorkSpecs != null && eligibleWorkSpecs.size() > 0) {
            WorkSpec[] eligibleWorkSpecsArray = eligibleWorkSpecs.toArray(new WorkSpec[0]);
            // Delegate to the underlying scheduler.
            for (Scheduler scheduler : schedulers) {
                scheduler.schedule(eligibleWorkSpecsArray);
            }
        }
    }

```

