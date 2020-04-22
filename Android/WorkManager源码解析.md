## WorkManager源码解析

1、首先引入implementation "androidx.work:work-runtime:2.3.1"

注意版本号要与'androidx.appcompat:appcompat:1.1.0'对应，要不然会有版本冲突。这个google做的不太好，官方文档也没有说明。

workmanager 不会随着app被杀死就不执行。



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

        WorkSpecDao workSpecDao = workDatabase.workSpecDao();//（调用005-14）
        List<WorkSpec> eligibleWorkSpecs;

        workDatabase.beginTransaction();//调用005-15
        try {
            eligibleWorkSpecs = workSpecDao.getEligibleWorkForScheduling(
                    configuration.getMaxSchedulerLimit());//根据参数生成WorkSpec列表（调用005-16）
            if (eligibleWorkSpecs != null && eligibleWorkSpecs.size() > 0) {
                long now = System.currentTimeMillis();

                // Mark all the WorkSpecs as scheduled.
                // Calls to Scheduler#schedule() could potentially result in more schedules
                // on a separate thread. Therefore, this needs to be done first.
                for (WorkSpec workSpec : eligibleWorkSpecs) {
                    workSpecDao.markWorkSpecScheduled(workSpec.id, now);//单个处理，包括先后处理顺序，处理线程等工作（调用005-17）
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
                scheduler.schedule(eligibleWorkSpecsArray);//单独线程运行（调用005-18）
            }
        }
    }

//005-14 此方法在androidx.work.impl.WorkDatabase_Impl
    public WorkSpecDao workSpecDao() {
        if (this._workSpecDao != null) {
            return this._workSpecDao;//不为空直接返回
        } else {
            synchronized(this) {//加锁
                if (this._workSpecDao == null) {
                    this._workSpecDao = new WorkSpecDao_Impl(this);//创建对象（代码比较长就不再贴了）
                }
                return this._workSpecDao;
            }
        }
    }
//005-15
    public void beginTransaction() {
        assertNotMainThread();//读参数，是否允许在主线程运行。
        SupportSQLiteDatabase database = mOpenHelper.getWritableDatabase();//创建数据库
        mInvalidationTracker.syncTriggers(database);//异步执行数据库配置操作
        database.beginTransaction();//可以开始操作数据库了
    }

//005-16
public List<WorkSpec> getEligibleWorkForScheduling(int schedulerLimit) {
        String _sql = "SELECT `required_network_type`, `requires_charging`, `requires_device_idle`, `requires_battery_not_low`, `requires_storage_not_low`, `trigger_content_update_delay`, `trigger_max_content_delay`, `content_uri_triggers`, `WorkSpec`.`id` AS `id`, `WorkSpec`.`state` AS `state`, `WorkSpec`.`worker_class_name` AS `worker_class_name`, `WorkSpec`.`input_merger_class_name` AS `input_merger_class_name`, `WorkSpec`.`input` AS `input`, `WorkSpec`.`output` AS `output`, `WorkSpec`.`initial_delay` AS `initial_delay`, `WorkSpec`.`interval_duration` AS `interval_duration`, `WorkSpec`.`flex_duration` AS `flex_duration`, `WorkSpec`.`run_attempt_count` AS `run_attempt_count`, `WorkSpec`.`backoff_policy` AS `backoff_policy`, `WorkSpec`.`backoff_delay_duration` AS `backoff_delay_duration`, `WorkSpec`.`period_start_time` AS `period_start_time`, `WorkSpec`.`minimum_retention_duration` AS `minimum_retention_duration`, `WorkSpec`.`schedule_requested_at` AS `schedule_requested_at`, `WorkSpec`.`run_in_foreground` AS `run_in_foreground` FROM workspec WHERE state=0 AND schedule_requested_at=-1 ORDER BY period_start_time LIMIT (SELECT MAX(?-COUNT(*), 0) FROM workspec WHERE schedule_requested_at<>-1 AND state NOT IN (2, 3, 5))";
        RoomSQLiteQuery _statement = RoomSQLiteQuery.acquire("SELECT `required_network_type`, `requires_charging`, `requires_device_idle`, `requires_battery_not_low`, `requires_storage_not_low`, `trigger_content_update_delay`, `trigger_max_content_delay`, `content_uri_triggers`, `WorkSpec`.`id` AS `id`, `WorkSpec`.`state` AS `state`, `WorkSpec`.`worker_class_name` AS `worker_class_name`, `WorkSpec`.`input_merger_class_name` AS `input_merger_class_name`, `WorkSpec`.`input` AS `input`, `WorkSpec`.`output` AS `output`, `WorkSpec`.`initial_delay` AS `initial_delay`, `WorkSpec`.`interval_duration` AS `interval_duration`, `WorkSpec`.`flex_duration` AS `flex_duration`, `WorkSpec`.`run_attempt_count` AS `run_attempt_count`, `WorkSpec`.`backoff_policy` AS `backoff_policy`, `WorkSpec`.`backoff_delay_duration` AS `backoff_delay_duration`, `WorkSpec`.`period_start_time` AS `period_start_time`, `WorkSpec`.`minimum_retention_duration` AS `minimum_retention_duration`, `WorkSpec`.`schedule_requested_at` AS `schedule_requested_at`, `WorkSpec`.`run_in_foreground` AS `run_in_foreground` FROM workspec WHERE state=0 AND schedule_requested_at=-1 ORDER BY period_start_time LIMIT (SELECT MAX(?-COUNT(*), 0) FROM workspec WHERE schedule_requested_at<>-1 AND state NOT IN (2, 3, 5))", 1);
        int _argIndex = 1;
        _statement.bindLong(_argIndex, (long)schedulerLimit);
        this.__db.assertNotSuspendingTransaction();
        Cursor _cursor = DBUtil.query(this.__db, _statement, false, (CancellationSignal)null);

        ArrayList var59;
        try {
            int _cursorIndexOfMRequiredNetworkType = CursorUtil.getColumnIndexOrThrow(_cursor, "required_network_type");
            int _cursorIndexOfMRequiresCharging = CursorUtil.getColumnIndexOrThrow(_cursor, "requires_charging");
            int _cursorIndexOfMRequiresDeviceIdle = CursorUtil.getColumnIndexOrThrow(_cursor, "requires_device_idle");
            int _cursorIndexOfMRequiresBatteryNotLow = CursorUtil.getColumnIndexOrThrow(_cursor, "requires_battery_not_low");
            int _cursorIndexOfMRequiresStorageNotLow = CursorUtil.getColumnIndexOrThrow(_cursor, "requires_storage_not_low");
            int _cursorIndexOfMTriggerContentUpdateDelay = CursorUtil.getColumnIndexOrThrow(_cursor, "trigger_content_update_delay");
            int _cursorIndexOfMTriggerMaxContentDelay = CursorUtil.getColumnIndexOrThrow(_cursor, "trigger_max_content_delay");
            int _cursorIndexOfMContentUriTriggers = CursorUtil.getColumnIndexOrThrow(_cursor, "content_uri_triggers");
            int _cursorIndexOfId = CursorUtil.getColumnIndexOrThrow(_cursor, "id");
            int _cursorIndexOfState = CursorUtil.getColumnIndexOrThrow(_cursor, "state");
            int _cursorIndexOfWorkerClassName = CursorUtil.getColumnIndexOrThrow(_cursor, "worker_class_name");
            int _cursorIndexOfInputMergerClassName = CursorUtil.getColumnIndexOrThrow(_cursor, "input_merger_class_name");
            int _cursorIndexOfInput = CursorUtil.getColumnIndexOrThrow(_cursor, "input");
            int _cursorIndexOfOutput = CursorUtil.getColumnIndexOrThrow(_cursor, "output");
            int _cursorIndexOfInitialDelay = CursorUtil.getColumnIndexOrThrow(_cursor, "initial_delay");
            int _cursorIndexOfIntervalDuration = CursorUtil.getColumnIndexOrThrow(_cursor, "interval_duration");
            int _cursorIndexOfFlexDuration = CursorUtil.getColumnIndexOrThrow(_cursor, "flex_duration");
            int _cursorIndexOfRunAttemptCount = CursorUtil.getColumnIndexOrThrow(_cursor, "run_attempt_count");
            int _cursorIndexOfBackoffPolicy = CursorUtil.getColumnIndexOrThrow(_cursor, "backoff_policy");
            int _cursorIndexOfBackoffDelayDuration = CursorUtil.getColumnIndexOrThrow(_cursor, "backoff_delay_duration");
            int _cursorIndexOfPeriodStartTime = CursorUtil.getColumnIndexOrThrow(_cursor, "period_start_time");
            int _cursorIndexOfMinimumRetentionDuration = CursorUtil.getColumnIndexOrThrow(_cursor, "minimum_retention_duration");
            int _cursorIndexOfScheduleRequestedAt = CursorUtil.getColumnIndexOrThrow(_cursor, "schedule_requested_at");
            int _cursorIndexOfRunInForeground = CursorUtil.getColumnIndexOrThrow(_cursor, "run_in_foreground");
            ArrayList _result = new ArrayList(_cursor.getCount());

            while(_cursor.moveToNext()) {
                String _tmpId = _cursor.getString(_cursorIndexOfId);
                String _tmpWorkerClassName = _cursor.getString(_cursorIndexOfWorkerClassName);
                Constraints _tmpConstraints = new Constraints();
                int _tmp = _cursor.getInt(_cursorIndexOfMRequiredNetworkType);
                NetworkType _tmpMRequiredNetworkType = WorkTypeConverters.intToNetworkType(_tmp);
                _tmpConstraints.setRequiredNetworkType(_tmpMRequiredNetworkType);
                int _tmp_1 = _cursor.getInt(_cursorIndexOfMRequiresCharging);
                boolean _tmpMRequiresCharging = _tmp_1 != 0;
                _tmpConstraints.setRequiresCharging(_tmpMRequiresCharging);
                int _tmp_2 = _cursor.getInt(_cursorIndexOfMRequiresDeviceIdle);
                boolean _tmpMRequiresDeviceIdle = _tmp_2 != 0;
                _tmpConstraints.setRequiresDeviceIdle(_tmpMRequiresDeviceIdle);
                int _tmp_3 = _cursor.getInt(_cursorIndexOfMRequiresBatteryNotLow);
                boolean _tmpMRequiresBatteryNotLow = _tmp_3 != 0;
                _tmpConstraints.setRequiresBatteryNotLow(_tmpMRequiresBatteryNotLow);
                int _tmp_4 = _cursor.getInt(_cursorIndexOfMRequiresStorageNotLow);
                boolean _tmpMRequiresStorageNotLow = _tmp_4 != 0;
                _tmpConstraints.setRequiresStorageNotLow(_tmpMRequiresStorageNotLow);
                long _tmpMTriggerContentUpdateDelay = _cursor.getLong(_cursorIndexOfMTriggerContentUpdateDelay);
                _tmpConstraints.setTriggerContentUpdateDelay(_tmpMTriggerContentUpdateDelay);
                long _tmpMTriggerMaxContentDelay = _cursor.getLong(_cursorIndexOfMTriggerMaxContentDelay);
                _tmpConstraints.setTriggerMaxContentDelay(_tmpMTriggerMaxContentDelay);
                byte[] _tmp_5 = _cursor.getBlob(_cursorIndexOfMContentUriTriggers);
                ContentUriTriggers _tmpMContentUriTriggers = WorkTypeConverters.byteArrayToContentUriTriggers(_tmp_5);
                _tmpConstraints.setContentUriTriggers(_tmpMContentUriTriggers);
                WorkSpec _item = new WorkSpec(_tmpId, _tmpWorkerClassName);
                int _tmp_6 = _cursor.getInt(_cursorIndexOfState);
                _item.state = WorkTypeConverters.intToState(_tmp_6);
                _item.inputMergerClassName = _cursor.getString(_cursorIndexOfInputMergerClassName);
                byte[] _tmp_7 = _cursor.getBlob(_cursorIndexOfInput);
                _item.input = Data.fromByteArray(_tmp_7);
                byte[] _tmp_8 = _cursor.getBlob(_cursorIndexOfOutput);
                _item.output = Data.fromByteArray(_tmp_8);
                _item.initialDelay = _cursor.getLong(_cursorIndexOfInitialDelay);
                _item.intervalDuration = _cursor.getLong(_cursorIndexOfIntervalDuration);
                _item.flexDuration = _cursor.getLong(_cursorIndexOfFlexDuration);
                _item.runAttemptCount = _cursor.getInt(_cursorIndexOfRunAttemptCount);
                int _tmp_9 = _cursor.getInt(_cursorIndexOfBackoffPolicy);
                _item.backoffPolicy = WorkTypeConverters.intToBackoffPolicy(_tmp_9);
                _item.backoffDelayDuration = _cursor.getLong(_cursorIndexOfBackoffDelayDuration);
                _item.periodStartTime = _cursor.getLong(_cursorIndexOfPeriodStartTime);
                _item.minimumRetentionDuration = _cursor.getLong(_cursorIndexOfMinimumRetentionDuration);
                _item.scheduleRequestedAt = _cursor.getLong(_cursorIndexOfScheduleRequestedAt);
                int _tmp_10 = _cursor.getInt(_cursorIndexOfRunInForeground);
                _item.runInForeground = _tmp_10 != 0;
                _item.constraints = _tmpConstraints;
                _result.add(_item);
            }

            var59 = _result;
        } finally {
            _cursor.close();
            _statement.release();
        }

        return var59;
    }

//005-17
public int markWorkSpecScheduled(String id, long startTime) {
        this.__db.assertNotSuspendingTransaction();
        SupportSQLiteStatement _stmt = this.__preparedStmtOfMarkWorkSpecScheduled.acquire();
        int _argIndex = 1;
        _stmt.bindLong(_argIndex, startTime);
        _argIndex = 2;
        if (id == null) {
            _stmt.bindNull(_argIndex);
        } else {
            _stmt.bindString(_argIndex, id);
        }

        this.__db.beginTransaction();

        int var7;
        try {
            int _result = _stmt.executeUpdateDelete();
            this.__db.setTransactionSuccessful();
            var7 = _result;
        } finally {
            this.__db.endTransaction();
            this.__preparedStmtOfMarkWorkSpecScheduled.release(_stmt);
        }

        return var7;
    }

//005-18
    public void schedule(@NonNull WorkSpec... workSpecs) {
        if (mIsMainProcess == null) {
            // The default process name is the package name.
            mIsMainProcess = TextUtils.equals(mContext.getPackageName(), getProcessName());
        }

        if (!mIsMainProcess) {
            Logger.get().info(TAG, "Ignoring schedule request in non-main process");
            return;
        }

        registerExecutionListenerIfNeeded();

        // Keep track of the list of new WorkSpecs whose constraints need to be tracked.
        // Add them to the known list of constrained WorkSpecs and call replace() on
        // WorkConstraintsTracker. That way we only need to synchronize on the part where we
        // are updating mConstrainedWorkSpecs.
        List<WorkSpec> constrainedWorkSpecs = new ArrayList<>();
        List<String> constrainedWorkSpecIds = new ArrayList<>();
        for (WorkSpec workSpec : workSpecs) {
            if (workSpec.state == WorkInfo.State.ENQUEUED
                    && !workSpec.isPeriodic()
                    && workSpec.initialDelay == 0L
                    && !workSpec.isBackedOff()) {
                if (workSpec.hasConstraints()) {
                    if (SDK_INT >= 23 && workSpec.constraints.requiresDeviceIdle()) {
                        // Ignore requests that have an idle mode constraint.
                        Logger.get().debug(TAG,
                                String.format("Ignoring WorkSpec %s, Requires device idle.",
                                        workSpec));
                    } else if (SDK_INT >= 24 && workSpec.constraints.hasContentUriTriggers()) {
                        // Ignore requests that have content uri triggers.
                        Logger.get().debug(TAG,
                                String.format("Ignoring WorkSpec %s, Requires ContentUri triggers.",
                                        workSpec));
                    } else {
                        constrainedWorkSpecs.add(workSpec);
                        constrainedWorkSpecIds.add(workSpec.id);
                    }
                } else {
                    Logger.get().debug(TAG, String.format("Starting work for %s", workSpec.id));
                    mWorkManagerImpl.startWork(workSpec.id);
                }
            }
        }

        // onExecuted() which is called on the main thread also modifies the list of mConstrained
        // WorkSpecs. Therefore we need to lock here.
        synchronized (mLock) {
            if (!constrainedWorkSpecs.isEmpty()) {
                Logger.get().debug(TAG, String.format("Starting tracking for [%s]",
                        TextUtils.join(",", constrainedWorkSpecIds)));
                mConstrainedWorkSpecs.addAll(constrainedWorkSpecs);
                mWorkConstraintsTracker.replace(mConstrainedWorkSpecs);
            }
        }
    }
```

