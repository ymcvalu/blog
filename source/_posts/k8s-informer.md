---
title: k8s informer源码分析
date: 2019-12-28 22:50:56
tags:
    - k8s - informer
---

# kubernetes之Informer

 ![](/img/kubernetes-high-level-component-archtecture.jpg)

我们与`k8s`集群的交互，主要是通过向`api-server`提交资源对象。

举个例子，我们要部署一个应用，则只需要提交一个`deployment`类型的资源对象到`api-server`。

`api-server`在验证用户提交的资源对象通过之后，会将该资源对象保存到`etcd`集群中。

同时，`api-server`还需要通知`deployment-manager`有一个新的`deployment`资源对象被新增了，从而能够对它进行处理，比如创建`replicaSet`。

也就是，在`k8s`中，需要有一个机制，能够让资源对象的`controller`能够感知到资源对象。

`Informer`接口就是用于实现该功能。

`Informer`提供了：

- `ListAndWatch`：用于跟`api-server`同步资源对象列表和更新
- 带索引的缓存功能：将资源对象缓存到本地，可以直接通过缓存查询资源对象内容，减轻`api-server`压力
- 事件回调功能：提供回调机制，当有资源对象新增、更新或者删除时可以回调自定义接口

`Informer`是`client-go`中一个非常核心的功能，除了在`controller-manager`、`scheduler`等组件中被广泛使用，在各种自定义资源对象的控制器中也离不开它的身影。

![](/img/informer.png)

上图描述了一个资源对象的控制器的大体工作流程，来自于极客时间的《深入剖析Kubernetes》课程：

- 控制器通过`informer`与`api-server`同步资源对象的列表和变更
- 在事件回调中，将事件加入到`workQueue`中，这里`workQueue`可以协调生产者与消费者之间的速率，而且消费失败的事件可以重新加入队列，一般使用限速队列，当消费失败重新加入队列到下次重新消费之前，限速器会根据重试次数产生一定的延时，因为一般消费失败，马上进行重试很大概率还是会失败。
- 在控制循环中，不断从`workQueue`取出事件进行处理



接下来看一下`informer`的实现，相关的代码都在`k8s.io/client-go/tools/cache`包下面

### Informer相关的接口

```go
type ResourceEventHandler interface {
	OnAdd(obj interface{})
	OnUpdate(oldObj, newObj interface{})
	OnDelete(obj interface{})
}

type SharedInformer interface {
	// 添加一个事件handler
	AddEventHandler(handler ResourceEventHandler)
	// 添加一个事件handler，同时指定resync周期，上一个方法没有指定则使用默认的
    // 如果resync周期大于0，则会定期将本地缓存中的资源对象加入到DeltaFifo中，重新触发事件
	AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration)
	// 获取本地缓存
	GetStore() Store
	
	GetController() Controller
	// 开始从apiServer同步资源对象
    // 该方法会阻塞直到stop
	Run(stopCh <-chan struct{})
	// 是否List已经执行，并且List返回的资源对象已经从DeltaFifo中被pop完
	HasSynced() bool
	// 获取最新的资源版本号;k8s依赖于etcd，每个资源对象都有自己的全局唯一的资源版本号，并且是全局递增的
	LastSyncResourceVersion() string
}

type SharedIndexInformer interface {
	SharedInformer
	// 给本地缓存添加索引
	AddIndexers(indexers Indexers) error
    // 获取带索引的本地缓存
	GetIndexer() Indexer
}
```

### 探索informer实现

接下来，我们看一下`SharedIndexInformer`接口的实现

##### 创建

```go
type sharedIndexInformer struct {
    // 带索引的本地缓存
	indexer    Indexer
    // 控制器，上面图中的Reflector实际上是controller的
	controller Controller
	// 保存事件handler列表，并且分发事件
	processor             *sharedProcessor
    // 用于检测本地缓存中的资源对象是否被更改了
    // 如果更新了本地缓存，则会导致缓存与apiServer的数据不一致
    // 如果我们确实要更新资源对象，则应该先使用DeepCopy获取一个副本，然后在副本上进行更新
	cacheMutationDetector MutationDetector
	// 提供与apiServer交互的ListAndWatch接口
	listerWatcher ListerWatcher
    // 该informer所关注的资源对象的类型
	objectType    runtime.Object
    // 定时检查是否需要resync的周期
	resyncCheckPeriod time.Duration
    // 通过AddEventHandler方法添加事件handler的默认resync周期
	defaultEventHandlerResyncPeriod time.Duration
    
    clock clock.Clock
    // 是否已经运行、停止
	started, stopped bool
	startedLock      sync.Mutex
    // 保护事件handler列表的锁
	blockDeltas sync.Mutex
}
```

接着看一下对应的创建：

```go
// NewSharedInformer creates a new instance for the listwatcher.
func NewSharedInformer(lw ListerWatcher, objType runtime.Object, resyncPeriod time.Duration) SharedInformer {
	return NewSharedIndexInformer(lw, objType, resyncPeriod, Indexers{})
}

// NewSharedIndexInformer creates a new instance for the listwatcher.
// 参数lw提供listAndWatch的接口
// 参数indexers则是本地缓存的索引函数
func NewSharedIndexInformer(lw ListerWatcher, objType runtime.Object, defaultEventHandlerResyncPeriod time.Duration, indexers Indexers) SharedIndexInformer {
	realClock := &clock.RealClock{}
	sharedIndexInformer := &sharedIndexInformer{
        // 初始化processor
		processor:                       &sharedProcessor{clock: realClock},
        // 创建一个带索引的本地缓存
		indexer:                         NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, indexers),
        // listAndWatch接口
		listerWatcher:                   lw,
		objectType:                      objType,
		resyncCheckPeriod:               defaultEventHandlerResyncPeriod,
		defaultEventHandlerResyncPeriod: defaultEventHandlerResyncPeriod,
        // 默认是一个空实现
		cacheMutationDetector:           NewCacheMutationDetector(fmt.Sprintf("%T", objType)),
		clock:                           realClock,
	}
	return sharedIndexInformer
}

```

上面的`NewIndex`方法，实际上返回的是一个并发安全的`map`，同时可以根据`indexers`中的索引函数来对资源对象进行索引。

##### 启动

```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()

	fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, s.indexer) // indexer会作为fifo的knownObjects

	cfg := &Config{
		Queue:            fifo, // 先进先出队列
		ListerWatcher:    s.listerWatcher,
		ObjectType:       s.objectType,
		FullResyncPeriod: s.resyncCheckPeriod,
		RetryOnError:     false,
        // 该方法会定时调用，判断是否有事件handler需要resync
		ShouldResync:     s.processor.shouldResync, // 是否同步
		// watch到的事件，最终会回调s.HandleDeltas
        // 该方法中会更新本地缓存，然后再通过s.processor将事件分发给注册的事件handler
		Process: s.HandleDeltas, 
	}

	func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()
		// 创建controller
		s.controller = New(cfg) 
		s.controller.(*controller).clock = s.clock
		s.started = true
	}()

	// Separate stop channel because Processor should be stopped strictly after controller
	processorStopCh := make(chan struct{})
	var wg wait.Group
	defer wg.Wait()              // Wait for Processor to stop
	defer close(processorStopCh) // Tell Processor to stop
    // 后台运行定时脏缓存检查
	wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
    // 我们注册的事件handler会被包装成processorListener
    // 后台运行processorListener的pop和run方法，每个processorListener会开两个子协程
	wg.StartWithChannel(processorStopCh, s.processor.run)

	defer func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()
		s.stopped = true // Don't want any new listeners
	}()
    // run controller
	s.controller.Run(stopCh)
}
```

可以看到，在`Run`方法中：

- 设置状态为开始运行
- 创建一个`deltaFifo`和`controller`
- 后台运行脏缓存检查
- 后台为每个`processorListener`启动`pop`和`run`工作协程，开始监听事件。一旦有新的事件到来，会先调用`s.HandleDeltas`将其更新到本地缓存，然后通过`channel`通知每个`processorListener`，`processorListener`再去回调我们注册的事件handler。
- 调用`controller`的`Run`方法，实际跟`api-server`的`listAndWatch`交互是由`controller`来负责的

### 解密controller

接下来我们看一下`controller`的实现：

```go
// Controller is a generic controller framework.
type Controller interface {
    // 开始运行
	Run(stopCh <-chan struct{})
    // Informer的HasSynced的实现
	HasSynced() bool
    // Informer的LastSyncResourceVersion的实现
	LastSyncResourceVersion() string
}

// Controller is a generic controller framework.
type controller struct {
	config         Config
	reflector      *Reflector
	reflectorMutex sync.RWMutex
	clock          clock.Clock
}

// New makes a new Controller from the given Config.
func New(c *Config) Controller {
	ctlr := &controller{
		config: *c,
		clock:  &clock.RealClock{},
	}
	return ctlr
}
```

接下来看一下主要的`Run`方法的逻辑：

```go
func (c *controller) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	go func() {
		<-stopCh
		c.config.Queue.Close()
	}()
    
    // 创建一个Reflector
	r := NewReflector(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		c.config.FullResyncPeriod,
	)
	r.ShouldResync = c.config.ShouldResync
	r.clock = c.clock

	c.reflectorMutex.Lock()
	c.reflector = r
	c.reflectorMutex.Unlock()

	var wg wait.Group
	defer wg.Wait()
	// 后台开始运行r.Run
    // 该方法会执行ListAndWatch，并且定时触发resync检查
	wg.StartWithChannel(stopCh, r.Run)
	// 运行c.processLoop，直到stop
	wait.Until(c.processLoop, time.Second, stopCh)
}
```

看一下`processLoop`方法：

```go
func (c *controller) processLoop() {
	for {
        // 这里的Queue，就是上面在s.Run里面创建的DeltaFifo
        // Process实际上就是s.HandleDeltas
        // ListAndWatch产生的事件会添加到DeltaFifo中，该方法不断从队列获取事件并回调Process
		obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
		if err != nil {
			if err == ErrFIFOClosed {
				return
			}
            // 如果需要重试，则重新加入队列中
			if c.config.RetryOnError {
				// This is the safe way to re-enqueue.
				c.config.Queue.AddIfNotPresent(obj)
			}
		}
	}
}
```

### Reflector：ListAndWatch

我们接下来看一下`Reflector`的主要逻辑：

```go
func (r *Reflector) Run(stopCh <-chan struct{}) {
	klog.V(3).Infof("Starting reflector %v (%s) from %s", r.expectedTypeName, r.resyncPeriod, r.name)
	wait.Until(func() {
        // 终于看到调用ListAndWatch了
		if err := r.ListAndWatch(stopCh); err != nil {
			utilruntime.HandleError(err)
		}
	}, r.period, stopCh)
}
```

接下来看一下`ListAndWatch`方法，该方法有点长，只保留相关代码：

```go
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
 	var resourceVersion string

	options := metav1.ListOptions{ResourceVersion: r.relistResourceVersion()}
	// 执行List
	if err := func() error {
		var list runtime.Object
		var err error
		listCh := make(chan struct{}, 1)
   
		go func() {
			pager := pager.New(pager.SimplePageFunc(func(opts metav1.ListOptions) (runtime.Object, error) {
				return r.listerWatcher.List(opts)
			}))
			if r.WatchListPageSize != 0 {
				pager.PageSize = r.WatchListPageSize
			}			
            // 获取list列表
			list, err = pager.List(context.Background(), options)
			close(listCh)
		}()
		select {
		case <-stopCh:
			return nil
		case <-listCh:
		}
 
		listMetaInterface, err := meta.ListAccessor(list)
		resourceVersion = listMetaInterface.GetResourceVersion()
 		items, err := meta.ExtractList(list)
        // 会调用DeltaFifo的Replace接口，替换掉DeltaFifo的内容
 		if err := r.syncWith(items, resourceVersion); err != nil {
			return fmt.Errorf("%s: Unable to sync list result: %v", r.name, err)
		}
 		// 更新已经同步的资源版本
		r.setLastSyncResourceVersion(resourceVersion)
 		return nil
	}(); err != nil {
		return err
	}

 
	// 定时触发resync检查
	go func() {
		resyncCh, cleanup := r.resyncChan()
		defer func() {
			cleanup() // Call the last one written into cleanup
		}()
		for {
			select {
			case <-resyncCh:
			case <-stopCh:
				return
			case <-cancelCh:
				return
			}
            // ShouldResync会检查是否需要触发resync
			if r.ShouldResync == nil || r.ShouldResync() {
                // 如果需要触发resync，则调用DeltaFifo的Resync
				if err := r.store.Resync(); err != nil {
					resyncerrc <- err
					return
				}
			}
			cleanup()
			resyncCh, cleanup = r.resyncChan()
		}
	}()

	// 开始Watch
	for {

		timeoutSeconds := int64(minWatchTimeout.Seconds() * (rand.Float64() + 1.0))
		
		options = metav1.ListOptions{
			ResourceVersion: resourceVersion,
			TimeoutSeconds: &timeoutSeconds,
			AllowWatchBookmarks: true,
		}

		w, err := r.listerWatcher.Watch(options)
		if err != nil {
			if utilnet.IsConnectionRefused(err) {
				time.Sleep(time.Second)
				continue
			}
			return nil
		}

		// 处理watch到的事件
		if err := r.watchHandler(w, &resourceVersion, resyncerrc, stopCh); err != nil {
			...
			return nil
		}
	}
}

```

`watch`的主要逻辑都在`watchHandler`中：

```go

func (r *Reflector) watchHandler(w watch.Interface, resourceVersion *string, errc chan error, stopCh <-chan struct{}) error {
	start := r.clock.Now()
	eventCount := 0

	defer w.Stop()

loop:
	for {
		select {
		case <-stopCh:
			return errorStopRequested
		case err := <-errc:
			return err
			// watch事件
		case event, ok := <-w.ResultChan():
			if !ok {
				break loop
			}
		 
			meta, err := meta.Accessor(event.Object)
			if err != nil {
				continue
			}
			// 获取对应的资源版本
			newResourceVersion := meta.GetResourceVersion()
			switch event.Type {
			// 新增事件
			case watch.Added:
				// 调用DeltaFifo的Add接口
				err := r.store.Add(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to add watch event object (%#v) to store: %v", r.name, event.Object, err))
				}
				// 更新事件
			case watch.Modified:
				// 调用DeltaFifo的Update接口
				err := r.store.Update(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to update watch event object (%#v) to store: %v", r.name, event.Object, err))
				}
				// 删除事件
			case watch.Deleted:
				err := r.store.Delete(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to delete watch event object (%#v) from store: %v", r.name, event.Object, err))
				}
			case watch.Bookmark:
				// A `Bookmark` means watch has synced here, just update the resourceVersion
			default:
				utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", r.name, event))
			}
			// 更新最后同步的资源版本
			*resourceVersion = newResourceVersion
			r.setLastSyncResourceVersion(newResourceVersion)
			eventCount++
		}
	}
 
 	return nil
}
```



### InformerFactory

`client-go`还提供了一个`SharedInformerFactory`，方便我们创建（多个）`Informer`。



### 总结

我们可以看到，`Reflector`会通过`ListAndWatch`接口，从`api-server`同步资源对象的变更情况，然后添加到`DeltaFifo`队列中；

`controller`会不断从`DeltaFifo`中取出事件，然后回调`sharedIndexInformer`的`HandleDeltas`方法；

`HandleDeltas`方法中，会更新本地缓存，然后根据事件类型调用注册的事件handler。

还要一些细节，比如`DeltaFifo`，`Indexer`，`processorListener`的实现，或者`Informer`的`HasSynced`接口实现等，比较简单就不在讲了。


