---
title: Talk about Kubernetes cronJob controller
date: 2019-12-14 13:57:32
weight: 5
draft: false
categories: ["Coding"]
tags: ["Golang", "Kubernetes", "Linux"]
lightgallery: true
comments: true
thumbnail: https://blog.johnwu.cc/images/featured/kubernetes.png
---

### 背景

之前一段时间正好接触到 kubernetes cronjob, 在接入时遇上了在一定量级下 cronjob schedule delay 的问题, 故开始读了下代码, 发现了一些问题并试着调优了下

### 存在的问题

按生产环境实际测试来看约 250-375  个 `*/1 * * * *`  每分钟 interval 的 cronjob 就会产生 delay, cronjob 和 controller manager 没有异常 event  但新产生的 job 出现了延迟, 由于我们设置了 `startingDeadlineSeconds` 故累加起来的 delay 最终导致了 cron 任务严重滞后

### 代码解读

出于分析上述问题的目的, 读了下 cronjob controller 的代码, 代码量不多, 可能由于[没上 GA](https://github.com/kubernetes/enhancements/pull/978) 的原因, 整个 controllor 代码的设计也比较过程式, 不会像其他组件用到一些比如 Informer, refractor之类的组件读起来相对晦涩

下面开始解读下 [release1.17](https://github.com/kubernetes/kubernetes/blob/release-1.17/pkg/controller/cronjob/cronjob_controller.go) 分支的 k8s cronjob controller 代码

- Controller struct

```go
type Controller struct {
	kubeClient clientset.Interface
	jobControl jobControlInterface
	sjControl  sjControlInterface
	podControl podControlInterface
	recorder   record.EventRecorder
}
```

cronjob controller 结构体, 即下文中常见的 jm(jobManager) , 主要包了 k8s internal api clinet `kubeclinet`, `jobControl` 和 `sjControl` k8s job 控制块，cronjob controller 会直接操作 job, 由 job 再去创建 pod, 并不会直接接触到 pod 对象(包括读)

- 入口函数 Run:

```go
// Run starts the main goroutine responsible for watching and syncing jobs.
func (jm *Controller) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	klog.Infof("Starting CronJob Manager")
	// Check things every 10 second.
	go wait.Until(jm.syncAll, 10*time.Second, stopCh)
	<-stopCh
	klog.Infof("Shutting down CronJob Manager")
}
```

cronjob controller 是个单线程单执行流的调度器, 由固定每 10s 的 interval 的 goroutine 做一次 syncAll 调用

- 主 loop 函数 syncAll

```go
// syncAll lists all the CronJobs and Jobs and reconciles them.
func (jm *Controller) syncAll() {
	// List children (Jobs) before parents (CronJob).
	// This guarantees that if we see any Job that got orphaned by the GC orphan finalizer,
	// we must also see that the parent CronJob has non-nil DeletionTimestamp (see #42639).
	// Note that this only works because we are NOT using any caches here.
	jobListFunc := func(opts metav1.ListOptions) (runtime.Object, error) {
		return jm.kubeClient.BatchV1().Jobs(metav1.NamespaceAll).List(opts)
	}

	js := make([]batchv1.Job, 0)
	err := pager.New(pager.SimplePageFunc(jobListFunc)).EachListItem(context.Background(), metav1.ListOptions{}, func(object runtime.Object) error {
		jobTmp, ok := object.(*batchv1.Job)
		if !ok {
			return fmt.Errorf("expected type *batchv1.Job, got type %T", jobTmp)
		}
		js = append(js, *jobTmp)
		return nil
	})

	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Failed to extract job list: %v", err))
		return
	}

	klog.V(4).Infof("Found %d jobs", len(js))
	cronJobListFunc := func(opts metav1.ListOptions) (runtime.Object, error) {
		return jm.kubeClient.BatchV1beta1().CronJobs(metav1.NamespaceAll).List(opts)
	}

	jobsBySj := groupJobsByParent(js)
	klog.V(4).Infof("Found %d groups", len(jobsBySj))
	err = pager.New(pager.SimplePageFunc(cronJobListFunc)).EachListItem(context.Background(), metav1.ListOptions{}, func(object runtime.Object) error {
		sj, ok := object.(*batchv1beta1.CronJob)
		if !ok {
			return fmt.Errorf("expected type *batchv1beta1.CronJob, got type %T", sj)
		}
		syncOne(sj, jobsBySj[sj.UID], time.Now(), jm.jobControl, jm.sjControl, jm.recorder)
		cleanupFinishedJobs(sj, jobsBySj[sj.UID], jm.jobControl, jm.sjControl, jm.recorder)
		return nil
	})

	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Failed to extract cronJobs list: %v", err))
		return
	}
}
```

首先 `pager.New(pager.SimplePageFunc(jobListFunc))`通过 Pager 调用了 `jobListFunc` 回调函数, 用于 list 出所有 namespace 下的 k8s job 对象, 并将这些 jobs 加入 slice 中, 这个 slices `js := make([]batchv1.Job, 0)` 用于在之后对 sync 单个 cronJob 时作为是否已经 trigger job 的判断

同理 ` pager.New(pager.SimplePageFunc(cronJobListFunc)).EachListItem` list 所有 cronjob 对象并对每个对象调用 `syncOne` 做实际 cronjob 调度, 在调度完后调用 `cleanupFinishedJobs` 完成清理工作

​		- 对于成功执行的 job 根据 `HistoryLimit` 进行 apiserver 中的资源清理

​		- 对于执行失败的 job 按 limitBackoff 的限制进行重试

 	   - 若处于非上述两种状态的 job 则忽略

 - 主调度函数 syncOne

   ```go
   func syncOne(sj *batchv1beta1.CronJob, js []batchv1.Job, now time.Time, jc jobControlInterface, sjc sjControlInterface, recorder record.EventRecorder) {
   	nameForLog := fmt.Sprintf("%s/%s", sj.Namespace, sj.Name)

     // 首先在之前的 batchv1.Job slice 中顺序查找是否有当前 cronJob 的子 job 并且看看是否有不在 jobActive 列表中的孤儿，以及已经执行完成但是还在 Active 列表中的 job，根据 job 状态记录 event (UnexpectedJob, SawCompletedJob)，删掉不对应的状态
   	childrenJobs := make(map[types.UID]bool)
   	for _, j := range js {
   		childrenJobs[j.ObjectMeta.UID] = true
   		found := inActiveList(*sj, j.ObjectMeta.UID)
   		if !found && !IsJobFinished(&j) {
   			recorder.Eventf(sj, v1.EventTypeWarning, "UnexpectedJob", "Saw a job that the controller did not create or forgot: %s", j.Name)
   		} else if found && IsJobFinished(&j) {
   			_, status := getFinishedStatus(&j)
   			deleteFromActiveList(sj, j.ObjectMeta.UID)
   			recorder.Eventf(sj, v1.EventTypeNormal, "SawCompletedJob", "Saw completed job: %s, status: %v", j.Name, status)
   		}
   	}

   	// 接着判断 Active 列表中是否有不在当前 cronjob 子 job 里的 job, 如果有则记录 MissingJob event 并从 Active 列表中移除
   	for _, j := range sj.Status.Active {
   		if found := childrenJobs[j.UID]; !found {
   			recorder.Eventf(sj, v1.EventTypeNormal, "MissingJob", "Active job went missing: %v", j.Name)
   			deleteFromActiveList(sj, j.UID)
   		}
   	}

     // 更新 cronjob 的状态
   	updatedSJ, err := sjc.UpdateStatus(sj)
   	if err != nil {
   		klog.Errorf("Unable to update status for %s (rv = %s): %v", nameForLog, sj.ResourceVersion, err)
   		return
   	}
   	*sj = *updatedSJ

     // 判断 cronjob 是否被删除, 若被删除则停止调度该 cronjob
   	if sj.DeletionTimestamp != nil {
   		// The CronJob is being deleted.
   		// Don't do anything other than updating status.
   		return
   	}

     // 判断 cronjob 是否被暂停, 若被暂停则停止调度该 cronjob
   	if sj.Spec.Suspend != nil && *sj.Spec.Suspend {
   		klog.V(4).Infof("Not starting job for %s because it is suspended", nameForLog)
   		return
   	}

     // getRecentUnmetScheduleTimes 是按 crontab 计算出的 job 执行时间表
     // 主要是根据配置的 unix cron table 计算出下一次 schedule 时间并做一些有效性判断
   	times, err := getRecentUnmetScheduleTimes(*sj, now)
   	if err != nil {
   		recorder.Eventf(sj, v1.EventTypeWarning, "FailedNeedsStart", "Cannot determine if job needs to be started: %v", err)
   		klog.Errorf("Cannot determine if %s needs to be started: %v", nameForLog, err)
   		return
   	}
   	// 若没有取到该 cronjob 的 table 时间表, 则认为该 job nextSchedule time 可能不合理, 停止调度该 cronjob
   	if len(times) == 0 {
   		klog.V(4).Infof("No unmet start times for %s", nameForLog)
   		return
   	}
     // 若取到时间表, 则计算出最近一次需要执行的时间, 若最近一次执行时间超过了 currentTime + StartingDeadlineSeconds 则标记为 tooLate 停止调度并记录 event
   	if len(times) > 1 {
   		klog.V(4).Infof("Multiple unmet start times for %s so only starting last one", nameForLog)
   	}

   	scheduledTime := times[len(times)-1]
   	tooLate := false
   	if sj.Spec.StartingDeadlineSeconds != nil {
   		tooLate = scheduledTime.Add(time.Second * time.Duration(*sj.Spec.StartingDeadlineSeconds)).Before(now)
   	}
   	if tooLate {
   		klog.V(4).Infof("Missed starting window for %s", nameForLog)
   		recorder.Eventf(sj, v1.EventTypeWarning, "MissSchedule", "Missed scheduled time to start a job: %s", scheduledTime.Format(time.RFC1123Z))
   		return
   	}
     // 若当前 cronjob 设置了并发策略则按照对应的并发策略进行并行 job 调度
   	if sj.Spec.ConcurrencyPolicy == batchv1beta1.ForbidConcurrent && len(sj.Status.Active) > 0 {
   		klog.V(4).Infof("Not starting job for %s because of prior execution still running and concurrency policy is Forbid", nameForLog)
   		return
   	}
   	if sj.Spec.ConcurrencyPolicy == batchv1beta1.ReplaceConcurrent {
   		for _, j := range sj.Status.Active {
   			klog.V(4).Infof("Deleting job %s of %s that was still running at next scheduled start time", j.Name, nameForLog)

   			job, err := jc.GetJob(j.Namespace, j.Name)
   			if err != nil {
   				recorder.Eventf(sj, v1.EventTypeWarning, "FailedGet", "Get job: %v", err)
   				return
   			}
   			if !deleteJob(sj, job, jc, recorder) {
   				return
   			}
   		}
   	}

     // 根据CronJob Spec中JobTemplate的配置获取Job对象，其中Job对象的名字会加上scheduledTime计算出的Hash，目前是unix timestamp
   	jobReq, err := getJobFromTemplate(sj, scheduledTime)
   	if err != nil {
   		klog.Errorf("Unable to make Job from template in %s: %v", nameForLog, err)
   		return
   	}
     // 调用 createJob 接口根据上面拿到的 jobTemplate 创建一个新 job
   	jobResp, err := jc.CreateJob(sj.Namespace, jobReq)
   	if err != nil {
   		// If the namespace is being torn down, we can safely ignore
   		// this error since all subsequent creations will fail.
   		if !errors.HasStatusCause(err, v1.NamespaceTerminatingCause) {
   			recorder.Eventf(sj, v1.EventTypeWarning, "FailedCreate", "Error creating job: %v", err)
   		}
   		return
   	}
   	klog.V(4).Infof("Created Job %s for %s", jobResp.Name, nameForLog)
   	recorder.Eventf(sj, v1.EventTypeNormal, "SuccessfulCreate", "Created job %v", jobResp.Name)

   	// 将刚创建的Job加到CronJob的Active列表中，设置LastScheduleTime，更新CronJob
   	ref, err := getRef(jobResp)
   	if err != nil {
   		klog.V(2).Infof("Unable to make object reference for job for %s", nameForLog)
   	} else {
   		sj.Status.Active = append(sj.Status.Active, *ref)
   	}
     // lastSchedulerTime 常用于监控判断 job schedule 是否符合预期
   	sj.Status.LastScheduleTime = &metav1.Time{Time: scheduledTime}
   	if _, err := sjc.UpdateStatus(sj); err != nil {
   		klog.Infof("Unable to update status for %s (rv = %s): %v", nameForLog, sj.ResourceVersion, err)
   	}

   	return
   }
   ```

- 获取 cron table 的 `getRecentUnmetScheduleTimes` 函数

```go
// getRecentUnmetScheduleTimes 会根据传递进来的 jobSpec 中的 time series 中来判断这些时间的有效性, 比如是否过期, 如果过期数量过多(>100)则认为该 job schedule 上有问题, 直接停止对该 cronjob 的调度
func getRecentUnmetScheduleTimes(sj batchv1beta1.CronJob, now time.Time) ([]time.Time, error) {
	starts := []time.Time{}
  // 使用 robfig/cron 对 cron schedule 进行解析
	sched, err := cron.ParseStandard(sj.Spec.Schedule)
	if err != nil {
		return starts, fmt.Errorf("unparseable schedule: %s : %s", sj.Spec.Schedule, err)
	}

	var earliestTime time.Time
  // 判断初始时间，如果CronJob之前被执行过，则以上次执行实现为准，如果没有执行过，则以CronJob创建时间为准
	if sj.Status.LastScheduleTime != nil {
		earliestTime = sj.Status.LastScheduleTime.Time
	} else {
		// If none found, then this is either a recently created scheduledJob,
		// or the active/completed info was somehow lost (contract for status
		// in kubernetes says it may need to be recreated), or that we have
		// started a job, but have not noticed it yet (distributed systems can
		// have arbitrary delays).  In any case, use the creation time of the
		// CronJob as last known start time.
		earliestTime = sj.ObjectMeta.CreationTimestamp.Time
	}
  // 如果设置了StartingDeadlineSeconds, 并且当前时间减去该值比初始时间还晚, 那就以新的时间为准, 通常会极为 lateTime 从而放弃对旧 job 的调度
	if sj.Spec.StartingDeadlineSeconds != nil {
		schedulingDeadline := now.Add(-time.Second * time.Duration(*sj.Spec.StartingDeadlineSeconds))

		if schedulingDeadline.After(earliestTime) {
			earliestTime = schedulingDeadline
		}
	}
  // 如果预期调度时间比现在还晚则停止调度
	if earliestTime.After(now) {
		return []time.Time{}, nil
	}

  // 计算从初始时间到现在所有需要执行的任务的时间, 并判断是否有过期状态的 job
	// 如果过期数量过多(>100)则认为该 job schedule 上有问题, 直接停止对该 cronjob 的调度
	for t := sched.Next(earliestTime); !t.After(now); t = sched.Next(t) {
		starts = append(starts, t)
		if len(starts) > 100 {
			// We can't get the most recent times so just return an empty slice
			return []time.Time{}, fmt.Errorf("too many missed start time (> 100). Set or decrease .spec.startingDeadlineSeconds or check clock skew")
		}
	}
	return starts, nil
}

```

基本上主要的业务逻辑都在这里了，整体上看还是十分简单粗暴的，是个单执行流 one by one 地不停轮询、计算需要执行的任务、任务执行时间表、同步任务状态的过程。



其中从上述代码也观察到3点可能的问题:

- 在 syncAll 函数中用到 pager.List 这是个非常冗余的操作, 每次调度时都在遍历一些可能未产生变更的 cronjob, 应该改为 watch 的机制
- 对于 cronjob 状态的变更可以不通过在每次 loop 时主动查询并更新, 而是通过 informer 注册 event 回调的方式, 在有变更时同步状态
- 对于过期数量超过 100 的 cronjob 会直接停止调度, 并且只记录的 event, 没有一个 drop old jobs 自愈的过程



### 调优

首先对于 pager.List 尝试替换成 informer watch 的机制, 思路也比较简单, 原先是通过 pager.List 传入的回调来获取 namespace 下所有的 job/cronjob, 现在改为在新建 controller 时注册进 watch event, 监听到变更事件时通过 k8s 封装好的 internal.api.sharedInformer 取回并构造成相同 struct 的对象即可

先看一段原本 pager.List 的代码:

```go
	cronJobListFunc := func(opts metav1.ListOptions) (runtime.Object, error) {
		return jm.kubeClient.BatchV1beta1().CronJobs(metav1.NamespaceAll).List(opts)
	}
	err = pager.New(pager.SimplePageFunc(cronJobListFunc)).EachListItem(context.Background(), metav1.ListOptions{}, func(object runtime.Object) error {
    ...
  }
```

接着按上述方式在 controller 中先注册 event:

```go
type Controller struct {
	kubeClient clientset.Interface
	jobControl jobControlInterface
	sjControl  sjControlInterface
	podControl podControlInterface
	recorder   record.EventRecorder

  // Codes after refractor
	queue workqueue.RateLimitingInterface
	cronjobSynced cache.InformerSynced
	syncHandler func(key string) error
	cronjobLister batchv1beta1Lister.CronJobLister
}

// 接着注册 Informer 并声明 CronJobListener 对应的 callback(add/update/delete)
func NewCronJobController(kubeClient clientset.Interface) (*CronJobController, error) {
	jm, err := NewController(kubeClient)
	if err != nil {
		return nil, err
	}
		queue:      workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "cronjob"),
	}
	cronjobInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
	})

	return jm, nil
}

```

k8s informer 的机制需要有 event trigger 来判断在什么事件下触发, 故我们可以简单先加上增删改时的 trigger

故上述代码修改为

```go
func (jm *CronJobController) addCronjob(obj interface{}) {
	d := obj.(*batchv1beta1.CronJob)
	glog.V(4).Infof("Adding CronJob %s", d.Name)
	jm.enqueue(d)
}

func (jm *CronJobController) updateCronjob(old, cur interface{}) {
	oldC := old.(*batchv1beta1.CronJob)
	curC := cur.(*batchv1beta1.CronJob)
	glog.V(4).Infof("Updating CronJob %s", oldC.Name)
	jm.enqueue(curC)
}

func (jm *CronJobController) deleteCronjob(obj interface{}) {
	d, ok := obj.(*batchv1beta1.CronJob)
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("couldn't get object from tombstone %#v", obj))
			return
		}
		d, ok = tombstone.Obj.(*batchv1beta1.CronJob)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("tombstone contained object that is not a CronJob %#v", obj))
			return
		}
	}
	glog.V(4).Infof("Deleting CronJob %s", d.Name)
	jm.enqueue(d)
}

// 接着注册 Informer 并声明 CronJobListener 对应的 callback(add/update/delete)
func NewCronJobController(kubeClient clientset.Interface) (*CronJobController, error) {
	jm, err := NewController(kubeClient)
	if err != nil {
		return nil, err
	}
		queue:      workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "cronjob"),
	}
	// event trigger
	cronjobInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    jm.addCronjob,
		UpdateFunc: jm.updateCronjob,
		DeleteFunc: jm.deleteCronjob,
	})

	// 有了 event trigger 后需要处理当事件触发获取到 cronjob 时该做什么操作
  // 按之前的逻辑来看应该对每个 cronjob 调用 syncOne
	jm.cronjobLister = cronjobInformer.Lister()
  jm.cronjobSynced = cronjobInformer.Informer().HasSynced
  jm.syncHandler = jm.syncOne
	return jm, nil
}
```

接着上述代码只是声明了一堆需要注册的 event trigger(增删改 cronjob 时将cronjob对象放入 workQueue) 和 event  handler(syncOne)

但 syncAll 主函数仍然做着原先的工作, 所以我期望的是在主函数中阻塞的获取 workQueue 中的 job, 并按 FIFO 的方式 process 每个 job (暂时没必要对于每个 job 起 goroutine syncOne, 会增加复杂性, 本身的处理能力猜测是足够的)

故将 syncAll 主函数代码改为

```go
// 提供 enqueue 函数用于将 batchv1beta1.CronJob 入队
func (jm *CronJobController) enqueue(cronjob *batchv1beta1.CronJob) {
	key, _ := controller.KeyFunc(cronjob)
	sjs := sjl.Items
	glog.V(4).Infof("Found %d cronjobs", len(sjs))

	jobsBySj := groupJobsByParent(js)
	jm.queue.Add(key)
}

// 提供一个从 informer 中获取对象的 key 并 enqueue 到 worker queue 的函数
func (jm *CronJobController) EnqueueCronjob() {
		cjobs, err := jm.cronjobLister.List(labels.Everything())
		if err != nil {
			fmt.Errorf("Could not list all cronjobs %v", err)
			return
		}
		for _, job := range cjobs {
			jm.enqueue(job)
		}
}

// 在 syncAll 主函数中起 goroutine 开始 watch cronjob, 一旦拿到 job 则调用注册的 syncOne 函数
func (jm *CronJobController) syncAll() {
    ...
    go wait.Until(func() {
      EnqueueCronjob()
	  }, time.Second*10, stopCh)

  	key, quit := jm.queue.Get()
    // get 是 block 的操作, 若收到 sigterm/sigkill 信号则会产生中断
    if quit {
      return false
    }
    defer jm.queue.Done(key)

  	// 调用注册的回调
    err := jm.syncHandler(key.(string))
    ...
}
```

这样整体逻辑大概清晰了, 需要做些检查性的耕作，比如由于无法确认是否为 deleted cronjob 故需要在 syncOne 函数中再加一次有效性判断等等

```go
func syncOne(...) {
  ...
  cronjob, err := jm.cronjobLister.CronJobs(namespace).Get(name)
	if errors.IsNotFound(err) {
		glog.V(2).Infof("CronJob %v has been deleted", key)
		return nil
  }
  if err != nil {
      return err
  }
  ...
}
```

所以综上, 我们就把 pager.List 的方式改造成了 informer watch 的方式，整体改造的代码如下

```go
type Controller struct {
	kubeClient clientset.Interface
	jobControl jobControlInterface
	sjControl  sjControlInterface
	podControl podControlInterface
	recorder   record.EventRecorder

  // Codes after refractor
	queue workqueue.RateLimitingInterface
	cronjobSynced cache.InformerSynced
	syncHandler func(key string) error
	cronjobLister batchv1beta1Lister.CronJobLister
}

func (jm *CronJobController) addCronjob(obj interface{}) {
	d := obj.(*batchv1beta1.CronJob)
	glog.V(4).Infof("Adding CronJob %s", d.Name)
	jm.enqueue(d)
}

func (jm *CronJobController) updateCronjob(old, cur interface{}) {
	oldC := old.(*batchv1beta1.CronJob)
	curC := cur.(*batchv1beta1.CronJob)
	glog.V(4).Infof("Updating CronJob %s", oldC.Name)
	jm.enqueue(curC)
}

func (jm *CronJobController) deleteCronjob(obj interface{}) {
	d, ok := obj.(*batchv1beta1.CronJob)
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("couldn't get object from tombstone %#v", obj))
			return
		}
		d, ok = tombstone.Obj.(*batchv1beta1.CronJob)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("tombstone contained object that is not a CronJob %#v", obj))
			return
		}
	}
	glog.V(4).Infof("Deleting CronJob %s", d.Name)
	jm.enqueue(d)
}

// 接着注册 Informer 并声明 CronJobListener 对应的 callback(add/update/delete)
func NewCronJobController(kubeClient clientset.Interface) (*CronJobController, error) {
	jm, err := NewController(kubeClient)
	if err != nil {
		return nil, err
	}
		queue:      workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "cronjob"),
	}
	// event trigger
	cronjobInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    jm.addCronjob,
		UpdateFunc: jm.updateCronjob,
		DeleteFunc: jm.deleteCronjob,
	})

	// 有了 event trigger 后需要处理当事件触发获取到 cronjob 时该做什么操作
  // 按之前的逻辑来看应该对每个 cronjob 调用 syncOne
	jm.cronjobLister = cronjobInformer.Lister()
  jm.cronjobSynced = cronjobInformer.Informer().HasSynced
  jm.syncHandler = jm.syncOne
	return jm, nil
}

// 提供 enqueue 函数用于将 batchv1beta1.CronJob 入队
func (jm *CronJobController) enqueue(cronjob *batchv1beta1.CronJob) {
	key, _ := controller.KeyFunc(cronjob)
	sjs := sjl.Items
	glog.V(4).Infof("Found %d cronjobs", len(sjs))

	jobsBySj := groupJobsByParent(js)
	jm.queue.Add(key)
}

// 提供一个从 informer 中获取对象并 enqueue 到 worker queue 的函数
func (jm *CronJobController) EnqueueCronjob() {
		cjobs, err := jm.cronjobLister.List(labels.Everything())
		if err != nil {
			fmt.Errorf("Could not list all cronjobs %v", err)
			return
		}
		for _, job := range cjobs {
			jm.enqueue(job)
		}
}

// 在 syncAll 主函数中起 goroutine 开始 watch cronjob, 一旦拿到 job 则调用注册的 syncOne 函数
func (jm *CronJobController) syncAll() {
    ...
    go wait.Until(func() {
      EnqueueCronjob()
	  }, time.Second*10, stopCh)

  	key, quit := jm.queue.Get()
    // get 是 block 的操作, 若收到 sigterm/sigkill 信号则会产生中断
    if quit {
      return false
    }
    defer jm.queue.Done(key)

  	// 调用注册的回调
    err := jm.syncHandler(key.(string))
    ...
}

func syncOne(...) {
  ...
  cronjob, err := jm.cronjobLister.CronJobs(namespace).Get(name)
	if errors.IsNotFound(err) {
		glog.V(2).Infof("CronJob %v has been deleted", key)
		return nil
  }
  if err != nil {
      return err
  }
  ...
}
```

