---
# Common-Defined params
title: "K8s Client Informer mechanism"
date: "2023-03-16"
description: "K8s Client Informer mechansim"
categories:
  - "Cloud Native"
tags:
  - "kunbernetes"
menu: main # Optional, add page to a menu. Options: main, side, footer
---
# K8s Client Informer机制

## 背景

在开发云原生数据库时，底层采用开源的[kubedb](https://github.com/kubedb)作为技术实现方案，查看了kubedb的源码，发现其最核心的机制是k8s Controller机制。可见使用controller机制是将传统技术变为云原生技术的关键。

## 控制器 (Controller)

>首先，Kubernetes 通过[**控制器 (controller)**](https://link.juejin.cn/?target=https%3A%2F%2Fkubernetes.io%2Fdocs%2Fconcepts%2Farchitecture%2Fcontroller%2F) 来维护集群的状态。比如 [ReplicaSetController](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fkubernetes%2Fkubernetes%2Ftree%2Fmaster%2Fpkg%2Fcontroller%2Freplicaset) 负责维护 ReplicaSet 对应的 Pod 数量，无论是 Pod 崩溃，被删除，还是 ReplicaSet 的 Replicas 配置发生变更，ReplicaSetController 都会采取必要的操作以维持各个 ReplicaSet 的 Selector 对应的 Pod 数量与 Replicas 一致。
>
>除了 ReplicaSetController 外，Kubernetes 还提供了很多[内置的控制器](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fkubernetes%2Fkubernetes%2Ftree%2Fmaster%2Fpkg%2Fcontroller)，包括 [EndpointController](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fkubernetes%2Fkubernetes%2Ftree%2Fmaster%2Fpkg%2Fcontroller%2Fendpoint), [NamespaceController](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fkubernetes%2Fkubernetes%2Ftree%2Fmaster%2Fpkg%2Fcontroller%2Fnamespace), [ServiceAccountController](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fkubernetes%2Fkubernetes%2Ftree%2Fmaster%2Fpkg%2Fcontroller%2Fserviceaccount) 等等。

## 控制器的工作原理：Kubernetes API

>  Kubernetes 控制面 (control plane) 的核心是 **API 服务器 (API server)**。除了CRUD的接口，API server还提供了watch接口，方便用户实时获取集群状态。有了这个接口，控制器才得以无延迟（准确地说是低延迟）地对状态变更作出响应。这里指的 "状态变更"，就是我们常说的**事件 (event)**。

### watch接口演示

![Screen Recording 2023-03-17 at 10.37.24](https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202303171049067.gif)

watch是基于客户的角度进行响应，所以可以看到tmp下的config在没watch之前就存在一个kube-root-ca.crt，一旦watch后kube-root-ca.crt对于client来说是added的，所以会触发新增事件。

如果我想要只监听未来发生的事件，watch提供了resourceVersion参数，我们可以先进行一次get，得到当前的configMapList的resourceVersion，再调用接口，实例如下：

![Screen Recording 2023-03-17 at 10.58.34](https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202303171101510.gif)

## 控制器的实现：Informer

>有了接口，我们就足以对集群状态进行观察和控制，但是，控制回路是一个无限循环的过程，每一次向服务器发送的请求，和每一个与服务器保持的连接都会给客户端和服务器端造成额外的资源开销，必须引入缓存机制，减少不必要的资源开销。Informer 就是 Kubernetes 开发者们设计的一套缓存机制。
>
>Kubernetes Informer 相关的代码包含在 [kubernetes/client-go](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fkubernetes%2Fclient-go) 仓库中，主要的包有 [cache](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fkubernetes%2Fclient-go%2Ftree%2Fmaster%2Ftools%2Fcache), [informers](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fkubernetes%2Fclient-go%2Ftree%2Fmaster%2Finformers) 和 [dynamic](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fkubernetes%2Fclient-go%2Ftree%2Fmaster%2Fdynamic)，其中 informers 和 dynamic 包主要实现 Informer 的使用，Informer 本身逻辑主要聚合在 cache 包中

![](https://github.com/kubernetes/sample-controller/blob/master/docs/images/client-go-controller-interaction.jpeg?raw=true)

### client-go使用

```go
func TestD1(t *testing.T) {
	clientSet, err := prepareClientSet()
	if err != nil {
		t.Error(err)
	}

	list, err := clientSet.CoreV1().Namespaces().List(context.TODO(), metav1.ListOptions{})
	if err != nil {
		t.Error(err)
	}
	for _, ns := range list.Items {
		fmt.Println(ns.Name)
	}
}

func prepareClientSet() (*kubernetes.Clientset, error) {
	kubeconfig := os.Getenv("KUBECONFIG")

	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		return nil, err
	}

	//clientset，支持直接请求 api 中各内置资源对象的 restful group 客户端集合
	clientSet, err := kubernetes.NewForConfig(config)
	if err != nil {
		return nil, err
	}
	return clientSet, nil
}
```

```shell
=== RUN   TestD1
default
kube-node-lease
kube-public
kube-system
tmp
--- PASS: TestD1 (6.93s)
PASS
```

### ListWatcher

根据上面的架构图，ListWatcher是离数据最近的地方

- listWatcher接口

  ```go
  // Lister is any object that knows how to perform an initial list.
  type Lister interface {
  	// List should return a list type object; the Items field will be extracted, and the
  	// ResourceVersion field will be used to start the watch in the right place.
  	List(options metav1.ListOptions) (runtime.Object, error)
  }
  
  // Watcher is any object that knows how to start a watch on a resource.
  type Watcher interface {
  	// Watch should begin a watch at the specified version.
  	Watch(options metav1.ListOptions) (watch.Interface, error)
  }
  
  // ListerWatcher is any object that knows how to perform an initial list and start a watch on a resource.
  type ListerWatcher interface {
  	Lister
  	Watcher
  }
  ```

- listWatcher使用

  ```go
  func TestD2(t *testing.T) {
  	clientSet, err := prepareClientSet()
  	if err != nil {
  		t.Error(err)
  		return
  	}
  	client := clientSet.CoreV1().RESTClient()
  	resource := "configmaps"
  	namespace := "tmp"
  	selector := fields.Everything()
  
  	lw := cache.NewListWatchFromClient(client, resource, namespace, selector)
  	list, err := lw.List(metav1.ListOptions{})
  	if err != nil {
  		t.Error(err)
  		return
  	}
  	extractList, err := meta.ExtractList(list)
  	if err != nil {
  		t.Error(err)
  		return
  	}
  	fmt.Println("Initial list:")
  
  	for _, item := range extractList {
  		configMap, ok := item.(*corev1.ConfigMap)
  		if !ok {
  			return
  		}
  		fmt.Println(configMap.Name)
  	}
  
  	accessor, err := meta.ListAccessor(list)
  	if err != nil {
  		t.Error(err)
  		return
  	}
  	//watch接口使用的resourceVersion参数
  	version := accessor.GetResourceVersion()
  	fmt.Println(version)
  
  	watch, err := lw.Watch(metav1.ListOptions{
  		ResourceVersion: version,
  	})
  	if err != nil {
  		t.Error(err)
  		return
  	}
  
  	stop := make(chan os.Signal)
  
  	signal.Notify(stop, os.Interrupt)
  
  	fmt.Println("Start watching...")
  loop:
  	for {
  		select {
  		case <-stop:
  			fmt.Println("stop watch")
  			break loop
  		case event, ok := <-watch.ResultChan():
  			if !ok {
  				fmt.Println("watch error")
  				break loop
  			}
  			configMap, ok := event.Object.(*corev1.ConfigMap)
  			if !ok {
  				fmt.Println("obj is not configmap")
  				break loop
  			}
  			fmt.Println(configMap.Name)
  		}
  	}
  }
  ```

- 输出如下

  ![Screen Recording 2023-03-17 at 14.30.24](https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202303171458995.gif)

- 这样就简单的利用lw监听k8s的某个资源，不过存在如下问题

  - 如果在监听结果后加上耗时业务逻辑则会阻塞监听协程，因此我们通过缓存进行异步
  - 异常中断的话，没有重新连接的机制

### Reflector

Reflector用来解决上述lw的问题

- 构造方法

  ```go
  // NewReflector creates a new Reflector object which will keep the
  // given store up to date with the server's contents for the given
  // resource. Reflector promises to only put things in the store that
  // have the type of expectedType, unless expectedType is nil. If
  // resyncPeriod is non-zero, then the reflector will periodically
  // consult its ShouldResync function to determine whether to invoke
  // the Store's Resync operation; `ShouldResync==nil` means always
  // "yes".  This enables you to use reflectors to periodically process
  // everything as well as incrementally processing the things that
  // change.
  func NewReflector(lw ListerWatcher, expectedType interface{}, store Store, resyncPeriod time.Duration) *Reflector {
  	return NewNamedReflector(naming.GetNameFromCallsite(internalPackages...), lw, expectedType, store, resyncPeriod)
  }
  ```

  	Reflector的构造方法中除了需要传入lw，还需要传入store和resyncPeriod，这两个参数便是用于解决异步和重试的问题

- 使用

  ```go
  func TestD3(t *testing.T) {
  	lw, err := prepareListWatcher()
  	if err != nil {
  		t.Error(err)
  		return
  	}
  	// newStore 用于创建一个 cache.Store 对象，作为当前资源状态的对象存储
  	store := cache.NewStore(cache.MetaNamespaceKeyFunc)
  
  	queue := cache.NewDeltaFIFOWithOptions(cache.DeltaFIFOOptions{
  		KnownObjects: store,
  	})
  
  	// 第 2 个参数是 expectedType, 用此参数限制进入队列的事件，
  	// 当然在 List 和 Watch 操作时返回的数据就只有一种类型，这个参数只起校验的作用；
  	// 第 4 个参数是 resyncPeriod，
  	// 这里传了 0，表示从不重新同步（除非连接超时或者中断），
  	// 如果传了非 0 值，会定期进行全量同步，避免累积和服务器的不一致，
  	// 重新同步过程中会产生 SYNC 类型的事件。
  	reflector := cache.NewReflector(lw, &v1.ConfigMap{}, queue, 0)
  
  	stopCh := make(chan struct{})
  	defer close(stopCh)
  
  	// reflector 开始运行后，队列中就会推入新收到的事件
  	go reflector.Run(stopCh)
  
  	// 注意处理事件过程中维护好 store 状态，包括 Add, Update, Delete 操作，
  	processObj := func(obj interface{}) error {
  		// 最先收到的事件会被最先处理
  		for _, d := range obj.(cache.Deltas) {
  			switch d.Type {
  			case cache.Sync, cache.Replaced, cache.Added, cache.Updated:
  				if _, exists, err := store.Get(d.Object); err == nil && exists {
  					if err := store.Update(d.Object); err != nil {
  						return err
  					}
  				} else {
  					if err := store.Add(d.Object); err != nil {
  						return err
  					}
  				}
  			case cache.Deleted:
  				if err := store.Delete(d.Object); err != nil {
  					return err
  				}
  			}
  			configMap, ok := d.Object.(*v1.ConfigMap)
  			if !ok {
  				return fmt.Errorf("not config: %T", d.Object)
  			}
  			fmt.Printf("%s: %s\n", d.Type, configMap.Name)
  		}
  		return nil
  	}
  
  	fmt.Println("Start syncing...")
  
  	// 持续运行直到 stopCh 关闭
  	wait.Until(func() {
  		for {
  			_, _ = queue.Pop(processObj)
  			//消费队列
  		}
  	}, time.Second, stopCh)
  
  }
  ```

在以上代码中事件接收后被缓存到队列中，避免了阻塞问题，但对队列的消费仍然是同步的，需要再实现多 worker 才能提高效率。在消费时，我们可以再使用workqueue提高效率。

### Controller

controller将reflector封装在内部，在run的时候，进行了reflector的构建

```go
func TestD4(t *testing.T) {
	lw, err := prepareListWatcher()
	if err != nil {
		return
	}

	store := cache.NewStore(cache.MetaNamespaceKeyFunc)

	queue := cache.NewDeltaFIFOWithOptions(cache.DeltaFIFOOptions{
		KnownObjects: store,
	})
	controller := cache.New(&cache.Config{
		Queue:         queue,
		ListerWatcher: lw,
		Process: func(obj interface{}) error {
			for _, d := range obj.(cache.Deltas) {
				switch d.Type {
				case cache.Sync, cache.Replaced, cache.Added, cache.Updated:
					if _, exists, err := store.Get(d.Object); err == nil && exists {
						if err := store.Update(d.Object); err != nil {
							return err
						}
					} else {
						if err := store.Add(d.Object); err != nil {
							return err
						}
					}
				case cache.Deleted:
					if err := store.Delete(d.Object); err != nil {
						return err
					}
				}
				configMap, ok := d.Object.(*v1.ConfigMap)
				if !ok {
					return fmt.Errorf("not config: %T", d.Object)
				}
				fmt.Printf("%s: %s\n", d.Type, configMap.Name)
			}
			return nil
		},
		ObjectType:       &v1.ConfigMap{},
		FullResyncPeriod: 0,
	})

	stopCh := make(chan struct{})
	defer close(stopCh)

	fmt.Println("Start syncing....")

	go controller.Run(stopCh)

	<-stopCh
}
```

```go
func (c *controller) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	go func() {
		<-stopCh
		c.config.Queue.Close()
	}()
  //create reflector
	r := NewReflector(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		c.config.FullResyncPeriod,
	)
	r.ShouldResync = c.config.ShouldResync
	r.WatchListPageSize = c.config.WatchListPageSize
	r.clock = c.clock
	if c.config.WatchErrorHandler != nil {
		r.watchErrorHandler = c.config.WatchErrorHandler
	}

	c.reflectorMutex.Lock()
	c.reflector = r
	c.reflectorMutex.Unlock()

	var wg wait.Group

	wg.StartWithChannel(stopCh, r.Run)

	wait.Until(c.processLoop, time.Second, stopCh)
	wg.Wait()
}
```

### Informer

informer是controller的进一步升级，帮我们解决了store的维护以及对象的转换

```go
func TestD5(t *testing.T) {
	lw, err := prepareListWatcher()
	if err != nil {
		return
	}
  //返回了store，进行消费
	store, controller := cache.NewInformer(lw, &v1.ConfigMap{}, 0, cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			configMap, ok := obj.(*v1.ConfigMap)
			if !ok {
				return
			}
			fmt.Printf("created: %s\n", configMap.Name)
		},
		UpdateFunc: func(old, new interface{}) {
			configMap, ok := old.(*v1.ConfigMap)
			if !ok {
				return
			}
			fmt.Printf("updated: %s\n", configMap.Name)
		},
		DeleteFunc: func(obj interface{}) {
			configMap, ok := obj.(*v1.ConfigMap)
			if !ok {
				return
			}
			fmt.Printf("deleted: %s\n", configMap.Name)
		},
	})
	stopCh := make(chan struct{})
	defer close(stopCh)

	fmt.Println("Start syncing....")

	go controller.Run(stopCh)

	<-stopCh
}
```

### SharedInformer

队列由于不能同时被多个消费者消费，SharedInformer就是为了解决这个问题

```go
func TestD6(t *testing.T) {
	lw, err := prepareListWatcher()
	if err != nil {
		return
	}
	informer := cache.NewSharedInformer(lw, &v1.ConfigMap{}, 0)

	informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			configMap, ok := obj.(*v1.ConfigMap)
			if !ok {
				return
			}
			fmt.Printf("created, printing namespace: %s\n", configMap.Namespace)
		},
	})

	informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			configMap, ok := obj.(*v1.ConfigMap)
			if !ok {
				return
			}
			fmt.Printf("created, printing name: %s\n", configMap.Name)
		},
	})

	stopCh := make(chan struct{})
	defer close(stopCh)

	fmt.Println("Start syncing....")

	go informer.Run(stopCh)
	<-stopCh

}
```

SharedInformer内部维护了sharedProcessor对象

```go
type sharedProcessor struct {
   listenersStarted bool
   listenersLock    sync.RWMutex
   listeners        []*processorListener
   syncingListeners []*processorListener
   clock            clock.Clock
   wg               wait.Group
}
```

在AddEventHandlerWithResyncPeriod方法中，会生成processorListener，添加到sharedProcessor

```go
func (s *sharedIndexInformer) AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration) (ResourceEventHandlerRegistration, error) {
	s.startedLock.Lock()
	defer s.startedLock.Unlock()

	if s.stopped {
		return nil, fmt.Errorf("handler %v was not added to shared informer because it has stopped already", handler)
	}

	if resyncPeriod > 0 {
		if resyncPeriod < minimumResyncPeriod {
			klog.Warningf("resyncPeriod %v is too small. Changing it to the minimum allowed value of %v", resyncPeriod, minimumResyncPeriod)
			resyncPeriod = minimumResyncPeriod
		}

		if resyncPeriod < s.resyncCheckPeriod {
			if s.started {
				klog.Warningf("resyncPeriod %v is smaller than resyncCheckPeriod %v and the informer has already started. Changing it to %v", resyncPeriod, s.resyncCheckPeriod, s.resyncCheckPeriod)
				resyncPeriod = s.resyncCheckPeriod
			} else {
				// if the event handler's resyncPeriod is smaller than the current resyncCheckPeriod, update
				// resyncCheckPeriod to match resyncPeriod and adjust the resync periods of all the listeners
				// accordingly
				s.resyncCheckPeriod = resyncPeriod
				s.processor.resyncCheckPeriodChanged(resyncPeriod)
			}
		}
	}

	listener := newProcessListener(handler, resyncPeriod, determineResyncPeriod(resyncPeriod, s.resyncCheckPeriod), s.clock.Now(), initialBufferSize, s.HasSynced)

	if !s.started {
		return s.processor.addListener(listener), nil
	}

	// in order to safely join, we have to
	// 1. stop sending add/update/delete notifications
	// 2. do a list against the store
	// 3. send synthetic "Add" events to the new handler
	// 4. unblock
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	handle := s.processor.addListener(listener)
	for _, item := range s.indexer.List() {
		// Note that we enqueue these notifications with the lock held
		// and before returning the handle. That means there is never a
		// chance for anyone to call the handle's HasSynced method in a
		// state when it would falsely return true (i.e., when the
		// shared informer is synced but it has not observed an Add
		// with isInitialList being true, nor when the thread
		// processing notifications somehow goes faster than this
		// thread adding them and the counter is temporarily zero).
		listener.add(addNotification{newObj: item, isInInitialList: true})
	}
	return handle, nil
}

```

SharedInformer自身实现了ResourceEventHandler接口

```go
func (s *sharedIndexInformer) OnAdd(obj interface{}) {
	// Invocation of this function is locked under s.blockDeltas, so it is
	// save to distribute the notification
	s.cacheMutationDetector.AddObject(obj)
	s.processor.distribute(addNotification{newObj: obj}, false)
}
func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()

	for listener, isSyncing := range p.listeners {
		switch {
		case !sync:
			// non-sync messages are delivered to every listener
			listener.add(obj)
		case isSyncing:
			// sync messages are delivered to every syncing listener
			listener.add(obj)
		default:
			// skipping a sync obj for a non-syncing listener
		}
	}
}
```

在run的时候创建controller

```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()

	if s.HasStarted() {
		klog.Warningf("The sharedIndexInformer has started, run more than once is not allowed")
		return
	}
	fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
		KnownObjects:          s.indexer,
		EmitDeltaTypeReplaced: true,
	})

	cfg := &Config{
		Queue:            fifo,
		ListerWatcher:    s.listerWatcher,
		ObjectType:       s.objectType,
		FullResyncPeriod: s.resyncCheckPeriod,
		RetryOnError:     false,
		ShouldResync:     s.processor.shouldResync,

    //deal event
		Process:           s.HandleDeltas,
		WatchErrorHandler: s.watchErrorHandler,
	}
```

```go
func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	if deltas, ok := obj.(Deltas); ok {
    // 将自身传入，调了自身onAdd方法，从而触发processor.distribute，实现多消费者共享
		return processDeltas(s, s.indexer, s.transform, deltas)
	}
	return errors.New("object given as Process argument is not Deltas")
}
func processDeltas(
	// Object which receives event notifications from the given deltas
	handler ResourceEventHandler,
	clientState Store,
	deltas Deltas,
	isInInitialList bool,
) error {
	// from oldest to newest
	for _, d := range deltas {
		obj := d.Object

		switch d.Type {
		case Sync, Replaced, Added, Updated:
			if old, exists, err := clientState.Get(obj); err == nil && exists {
				if err := clientState.Update(obj); err != nil {
					return err
				}
				handler.OnUpdate(old, obj)
			} else {
				if err := clientState.Add(obj); err != nil {
					return err
				}
				handler.OnAdd(obj, isInInitialList)
			}
		case Deleted:
			if err := clientState.Delete(obj); err != nil {
				return err
			}
			handler.OnDelete(obj)
		}
	}
	return nil

```

### Indexer

最一开始的store，在informer的每次触发后，都会保存最新的数据在缓存中。cache提供了indexer来为这份缓存创建索引。默认情况下提供的是MetaNamespaceIndexFunc，即通过namespace创建索引

```go
func TestD7(t *testing.T) {

	lw, err := prepareListWatcher()
	if err != nil {
		return
	}
	indexer, informer := cache.NewIndexerInformer(lw, &v1.ConfigMap{}, 0, cache.ResourceEventHandlerFuncs{},
		cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc})

	stopCh := make(chan struct{})
	defer close(stopCh)

	fmt.Println("Start syncing....")

	go informer.Run(stopCh)
	if !cache.WaitForCacheSync(stopCh, informer.HasSynced) {
		panic("timed out waiting for caches to sync")
	}

	// 获取 cache.NamespaceIndex 索引下，索引值为 "tmp" 中的所有键
	keys, err := indexer.IndexKeys(cache.NamespaceIndex, "tmp")

	obj, exist, err := indexer.GetByKey("tmp/hello4")
	if err == nil && exist {
		configMap, ok := obj.(*v1.ConfigMap)
		if ok {
			fmt.Println(configMap.Name)
		}
	}
	for _, v := range keys {
		fmt.Println(v)
	}

}
```

我们可以自定义索引，例如根据labels的值

```go
//index by label key
func MetaLabelsIndexFunc(obj interface{}) ([]string, error) {
	meta, err := meta.Accessor(obj)
	if err != nil {
		return []string{""}, fmt.Errorf("object has no meta: %v", err)
	}
	labels := meta.GetLabels()
	keys := make([]string, 0, len(labels))
	for k := range labels {
		keys = append(keys, k)
	}
	return keys, nil
}
func TestD7(t *testing.T) {

	lw, err := prepareListWatcher()
	if err != nil {
		return
	}
	indexer, informer := cache.NewIndexerInformer(lw, &v1.ConfigMap{}, 0, cache.ResourceEventHandlerFuncs{},
		cache.Indexers{
			cache.NamespaceIndex: cache.MetaNamespaceIndexFunc,
			"labels":             MetaLabelsIndexFunc,
		})
  ......
}
```

### SharedIndexInformer

SharedIndexInformer 是封装程度最高的一个，也是大部分情况下我们会选择使用的一个，同时也是 informers 包中 SharedInformerFactory 给所有内置资源创建 Informer 时所使用的方法。

### InformerFactory

Informer的工程模式

创建每一个informer，我们都需要准备lw以及indexer，go-client包中为我们准备了所有内置resource的工厂构建方法，放在SharedInformerFactory中。所有informer都会被缓存，重复调用都是同一个实例。

- 示例

  ```go
  configMapsInformer := informerFactory.Core().V1().ConfigMaps().Informer()
  ```

  默认情况下factory模式下的informer是NewSharedIndexInformer，并且会生成namespace的indexer

### Workerqueue

从上面的图可以看到，在informer发出事件后，controller使用了一个workerqueue进行异步消费，解决了之前提到的消费同步效率不高的问题。

client-go提供了workqueue包，用于构建workerqueue

```go
rateLimiter := workqueue.DefaultControllerRateLimiter()
queue := workqueue.NewRateLimitingQueue(rateLimiter)


// 默认名情况下提供了2个限流，令牌桶是为了流量并发控制，以及失败限制
func DefaultControllerRateLimiter() RateLimiter {
	return NewMaxOfRateLimiter(
		NewItemExponentialFailureRateLimiter(5*time.Millisecond, 1000*time.Second),
		// 10 qps, 100 bucket size.  This is only for retry speed and its only the overall factor (not per item)
		&BucketRateLimiter{Limiter: rate.NewLimiter(rate.Limit(10), 100)},
	)
}

// 失败后每次执行时间变成baseDelay*2^n,意味着如果不主动放弃这个key，队列会不断重试
// ItemExponentialFailureRateLimiter does a simple baseDelay*2^<num-failures> limit
// dealing with max failures and expiration are up to the caller
type ItemExponentialFailureRateLimiter struct {
	failuresLock sync.Mutex
	failures     map[interface{}]int

	baseDelay time.Duration
	maxDelay  time.Duration
}
```

- 使用

  ```go
  //在informer接收到事件
  configMapInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
  		AddFunc: func(new interface{}) {
  			key, err := cache.MetaNamespaceKeyFunc(new)
  			if err == nil {
  				// 只简单地把 key 放到工作队列中
  				klog.Infof("[%s] received", key)
  				queue.Add(key)
  			}
  		},
  	})
  ```

  ```go
  	for i := 0; i < workers; i++ {
  		go wait.Until(c.runWorker, time.Second, stopCh)
  	}
  
  	// 等待 stopCh 关闭
  	<-stopCh
  	return nil
  }
  
  func (c *SlothController) runWorker() {
  	for c.processNextItem() {
  	}
  }
  func (c *SlothController) processNextItem() bool {
  	// 阻塞住，等待新任务中...
  	key, shutdown := c.queue.Get()
  	if shutdown {
  		return false // 队列已进入退出状态，不要继续处理
  	}
  
  	defer c.queue.Done(key)
  
  	result := c.processItem(key)
  	c.handleErr(key, result)
  
  	return true
  }
  
  func (c *SlothController) processItem(key interface{}) error {
  	time.Sleep(c.nap)
  	klog.Infof("[%s] processed.", key)
  	return nil
  }
  
  // handleErr 用于检查任务处理结果，在必要的时候重试
  func (c *SlothController) handleErr(key interface{}, result error) {
  	if result == nil {
  		// 每次执行成功后清空重试记录。
  		c.queue.Forget(key)
  		return
  	}
  
  	if c.queue.NumRequeues(key) < c.maxRetries {
  		klog.Warningf("error processing %s: %v , retry %v", key, result, c.queue.NumRequeues(key))
  		// 执行失败，重试
  		c.queue.AddRateLimited(key)
  		return
  	}
  
  	// 重试次数过多，日志记录错误
  	c.queue.Forget(key)
  	// runtime.HandleError 是 Kubernetes 官方提供的错误响应方法，
  	// 提供一个错误响应的统一入口。
  	runtime.HandleError(fmt.Errorf("max retries exceeded, "+
  		"dropping item %s out of the queue: %v", key, result))
  }
  ```

### Crd Informer

kubernetes官方提供了所有内置资源的informer，那crd资源呢？

## 最佳实战 Sample Controller

kubernetes官方提供了[SampleController](https://github.com/kubernetes/sample-controller)的自定义controller例子，通过实现crd的controller来让开发者更熟悉informer。

### Crd

参考官方[Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)

### Code-generator

>[code-generator](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fkubernetes%2Fcode-generator) 是 Kubernetes 的代码生成套件，提供了许多代码生成命令，用于生成自定义资源相关的代码，包括我们关心的 Informer。
>
>实际上所有 Kubernetes 内置资源的 Lister, Informer, SharedInformerFactory，都是使用 code-generator 生成的

code generator包含了client-gen, lister-gen,informer-gen 等生成工具，除此之外还提供了generate-groups.sh脚本，方便我们一次生成所有代码。



SampleController的hack文件夹下提供了几个脚本

```shell
#hack 文件
.
├── boilerplate.go.txt #头部版权文件
├── custom-boilerplate.go.txt #自定义头部版权文件
├── tools.go
├── update-codegen.sh #脚本生成
└── verify-codegen.sh 

```

```shell
set -o errexit
set -o nounset
set -o pipefail

SCRIPT_ROOT=$(dirname "${BASH_SOURCE[0]}")/..
#调用code-generator
CODEGEN_PKG=${CODEGEN_PKG:-$(cd "${SCRIPT_ROOT}"; ls -d -1 ./vendor/k8s.io/code-generator 2>/dev/null || echo ../code-generator)}

# generate the code with:
# --output-base    because this script should also be able to run inside the vendor dir of
#                  k8s.io/kubernetes. The output-base is needed for the generators to output into the vendor dir
#                  instead of the $GOPATH directly. For normal projects this can be dropped.
bash "${CODEGEN_PKG}"/generate-groups.sh "deepcopy,client,informer,lister" \
  sample-controller/pkg/generated sample-controller/pkg/apis \
  samplecontroller:v1alpha1 \
  --output-base "$(dirname "${BASH_SOURCE[0]}")/../.." \
  --go-header-file "${SCRIPT_ROOT}"/hack/boilerplate.go.txt

# To use your own boilerplate text append:
#   --go-header-file "${SCRIPT_ROOT}"/hack/custom-boilerplate.go.txt
```

当资源类型有变更时即时执行update-codegen.sh进行更新，也可以使用verify-codegen.sh

### SampleController执行

- 创建crd

  仓库中为我们提供了artifact，使用crd.yaml

  ```yml
  apiVersion: apiextensions.k8s.io/v1
  kind: CustomResourceDefinition
  metadata:
    name: foos.samplecontroller.k8s.io
    # for more information on the below annotation, please see
    # https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/2337-k8s.io-group-protection/README.md
    annotations:
      "api-approved.kubernetes.io": "unapproved, experimental-only; please get an approval from Kubernetes API reviewers if you're trying to develop a CRD in the *.k8s.io or *.kubernetes.io groups"
  spec:
    group: samplecontroller.k8s.io
    versions:
      - name: v1alpha1
        served: true
        storage: true
        schema:
          # schema used for validation
          openAPIV3Schema:
            type: object
            properties:
              spec:
                type: object
                properties:
                  deploymentName:
                    type: string
                  replicas:
                    type: integer
                    minimum: 1
                    maximum: 10
              status:
                type: object
                properties:
                  availableReplicas:
                    type: integer
    names:
      kind: Foo
      plural: foos
    scope: Namespaced
  
  ```

- 执行

  ![Screen Recording 2023-03-20 at 10.36.26](https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202303201045174.gif)

​		***这里触发很多次的原因是我本地已经执行过，没有下载镜像的环境，触发update事件很快。***

- 核心源码

  ```go
  func main() {
  	klog.InitFlags(nil)
  	flag.Parse()
  
  	// set up signals so we handle the first shutdown signal gracefully
  	stopCh := signals.SetupSignalHandler()
  
  	cfg, err := clientcmd.BuildConfigFromFlags(masterURL, kubeconfig)
  	if err != nil {
  		klog.Fatalf("Error building kubeconfig: %s", err.Error())
  	}
  
  	kubeClient, err := kubernetes.NewForConfig(cfg)
  	if err != nil {
  		klog.Fatalf("Error building kubernetes clientset: %s", err.Error())
  	}
  
  	exampleClient, err := clientset.NewForConfig(cfg)
  	if err != nil {
  		klog.Fatalf("Error building example clientset: %s", err.Error())
  	}
  
    //构建informerFactory
  	kubeInformerFactory := kubeinformers.NewSharedInformerFactory(kubeClient, time.Second*30)
  	exampleInformerFactory := informers.NewSharedInformerFactory(exampleClient, time.Second*30)
  
    //创建controller
  	controller := NewController(kubeClient, exampleClient,
  		kubeInformerFactory.Apps().V1().Deployments(),
  		exampleInformerFactory.Samplecontroller().V1alpha1().Foos())
  
  	// notice that there is no need to run Start methods in a separate goroutine. (i.e. go kubeInformerFactory.Start(stopCh)
  	// Start method is non-blocking and runs all registered informers in a dedicated goroutine.
  	kubeInformerFactory.Start(stopCh)
  	exampleInformerFactory.Start(stopCh)
  
    //运行2个worker
  	if err = controller.Run(2, stopCh); err != nil {
  		klog.Fatalf("Error running controller: %s", err.Error())
  	}
  }
  ```

  ```go
  func NewController(
  	kubeclientset kubernetes.Interface,
  	sampleclientset clientset.Interface,
  	deploymentInformer appsinformers.DeploymentInformer,
  	fooInformer informers.FooInformer) *Controller {
  
  	// Create event broadcaster
  	// Add sample-controller types to the default Kubernetes Scheme so Events can be
  	// logged for sample-controller types.
  	utilruntime.Must(samplescheme.AddToScheme(scheme.Scheme))
  	klog.V(4).Info("Creating event broadcaster")
  	eventBroadcaster := record.NewBroadcaster()
  	eventBroadcaster.StartStructuredLogging(0)
  	eventBroadcaster.StartRecordingToSink(&typedcorev1.EventSinkImpl{Interface: kubeclientset.CoreV1().Events("")})
  	recorder := eventBroadcaster.NewRecorder(scheme.Scheme, corev1.EventSource{Component: controllerAgentName})
  
  	controller := &Controller{
  		kubeclientset:     kubeclientset,
  		sampleclientset:   sampleclientset,
  		deploymentsLister: deploymentInformer.Lister(),
  		deploymentsSynced: deploymentInformer.Informer().HasSynced,
  		foosLister:        fooInformer.Lister(),
  		foosSynced:        fooInformer.Informer().HasSynced,
  		workqueue:         workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "Foos"),
  		recorder:          recorder,
  	}
  
  	klog.Info("Setting up event handlers")
  	// Set up an event handler for when Foo resources change
    // add和update事件都入队
  	fooInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
  		AddFunc: controller.enqueueFoo,
  		UpdateFunc: func(old, new interface{}) {
  			controller.enqueueFoo(new)
  		},
  	})
  	// Set up an event handler for when Deployment resources change. This
  	// handler will lookup the owner of the given Deployment, and if it is
  	// owned by a Foo resource then the handler will enqueue that Foo resource for
  	// processing. This way, we don't need to implement custom logic for
  	// handling Deployment resources. More info on this pattern:
  	// https://github.com/kubernetes/community/blob/8cafef897a22026d42f5e5bb3f104febe7e29830/contributors/devel/controllers.md
    //还监听了deployment的event，用于更新foo的status
  	deploymentInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
  		AddFunc: controller.handleObject,
  		UpdateFunc: func(old, new interface{}) {
  			newDepl := new.(*appsv1.Deployment)
  			oldDepl := old.(*appsv1.Deployment)
  			if newDepl.ResourceVersion == oldDepl.ResourceVersion {
  				// Periodic resync will send update events for all known Deployments.
  				// Two different versions of the same Deployment will always have different RVs.
  				return
  			}
  			controller.handleObject(new)
  		},
  		DeleteFunc: controller.handleObject,
  	})
  
  	return controller
  }
  ```

  ```go
  //最终的逻辑处理,与之前描述的几乎一样
  func (c *Controller) Run(workers int, stopCh <-chan struct{}) error {
  	defer utilruntime.HandleCrash()
  	defer c.workqueue.ShutDown()
  
  	// Start the informer factories to begin populating the informer caches
  	klog.Info("Starting Foo controller")
  
  	// Wait for the caches to be synced before starting workers
  	klog.Info("Waiting for informer caches to sync")
  	if ok := cache.WaitForCacheSync(stopCh, c.deploymentsSynced, c.foosSynced); !ok {
  		return fmt.Errorf("failed to wait for caches to sync")
  	}
  
  	klog.Info("Starting workers")
  	// Launch two workers to process Foo resources
  	for i := 0; i < workers; i++ {
  		go wait.Until(c.runWorker, time.Second, stopCh)
  	}
  
  	klog.Info("Started workers")
  	<-stopCh
  	klog.Info("Shutting down workers")
  
  	return nil
  }
  func (c *Controller) runWorker() {
  	for c.processNextWorkItem() {
  	}
  }
  
  // processNextWorkItem will read a single work item off the workqueue and
  // attempt to process it, by calling the syncHandler.
  func (c *Controller) processNextWorkItem() bool {
  	obj, shutdown := c.workqueue.Get()
  
  	if shutdown {
  		return false
  	}
  
  	// We wrap this block in a func so we can defer c.workqueue.Done.
  	err := func(obj interface{}) error {
  		// We call Done here so the workqueue knows we have finished
  		// processing this item. We also must remember to call Forget if we
  		// do not want this work item being re-queued. For example, we do
  		// not call Forget if a transient error occurs, instead the item is
  		// put back on the workqueue and attempted again after a back-off
  		// period.
  		defer c.workqueue.Done(obj)
  		var key string
  		var ok bool
  		// We expect strings to come off the workqueue. These are of the
  		// form namespace/name. We do this as the delayed nature of the
  		// workqueue means the items in the informer cache may actually be
  		// more up to date that when the item was initially put onto the
  		// workqueue.
  		if key, ok = obj.(string); !ok {
  			// As the item in the workqueue is actually invalid, we call
  			// Forget here else we'd go into a loop of attempting to
  			// process a work item that is invalid.
  			c.workqueue.Forget(obj)
  			utilruntime.HandleError(fmt.Errorf("expected string in workqueue but got %#v", obj))
  			return nil
  		}
  		// Run the syncHandler, passing it the namespace/name string of the
  		// Foo resource to be synced.
  		if err := c.syncHandler(key); err != nil {
  			// Put the item back on the workqueue to handle any transient errors.
  			c.workqueue.AddRateLimited(key)
  			return fmt.Errorf("error syncing '%s': %s, requeuing", key, err.Error())
  		}
  		// Finally, if no error occurs we Forget this item so it does not
  		// get queued again until another change happens.
  		c.workqueue.Forget(obj)
  		klog.Infof("Successfully synced '%s'", key)
  		return nil
  	}(obj)
  
  	if err != nil {
  		utilruntime.HandleError(err)
  		return true
  	}
  
  	return true
  }
  
  // syncHandler compares the actual state with the desired, and attempts to
  // converge the two. It then updates the Status block of the Foo resource
  // with the current status of the resource.
  func (c *Controller) syncHandler(key string) error {
  	// Convert the namespace/name string into a distinct namespace and name
  	namespace, name, err := cache.SplitMetaNamespaceKey(key)
  	if err != nil {
  		utilruntime.HandleError(fmt.Errorf("invalid resource key: %s", key))
  		return nil
  	}
  
  	// Get the Foo resource with this namespace/name
  	foo, err := c.foosLister.Foos(namespace).Get(name)
  	if err != nil {
  		// The Foo resource may no longer exist, in which case we stop
  		// processing.
  		if errors.IsNotFound(err) {
  			utilruntime.HandleError(fmt.Errorf("foo '%s' in work queue no longer exists", key))
  			return nil
  		}
  
  		return err
  	}
  
  	deploymentName := foo.Spec.DeploymentName
  	if deploymentName == "" {
  		// We choose to absorb the error here as the worker would requeue the
  		// resource otherwise. Instead, the next time the resource is updated
  		// the resource will be queued again.
  		utilruntime.HandleError(fmt.Errorf("%s: deployment name must be specified", key))
  		return nil
  	}
  
  	// Get the deployment with the name specified in Foo.spec
    // 检查deployment是否创建完成
  	deployment, err := c.deploymentsLister.Deployments(foo.Namespace).Get(deploymentName)
  	// If the resource doesn't exist, we'll create it
  	if errors.IsNotFound(err) {
  		deployment, err = c.kubeclientset.AppsV1().Deployments(foo.Namespace).Create(context.TODO(), newDeployment(foo), metav1.CreateOptions{})
  	}
  
  	// If an error occurs during Get/Create, we'll requeue the item so we can
  	// attempt processing again later. This could have been caused by a
  	// temporary network failure, or any other transient reason.
  	if err != nil {
  		return err
  	}
  
  	// If the Deployment is not controlled by this Foo resource, we should log
  	// a warning to the event recorder and return error msg.
  	if !metav1.IsControlledBy(deployment, foo) {
  		msg := fmt.Sprintf(MessageResourceExists, deployment.Name)
  		c.recorder.Event(foo, corev1.EventTypeWarning, ErrResourceExists, msg)
  		return fmt.Errorf("%s", msg)
  	}
  
  	// If this number of the replicas on the Foo resource is specified, and the
  	// number does not equal the current desired replicas on the Deployment, we
  	// should update the Deployment resource.
  	if foo.Spec.Replicas != nil && *foo.Spec.Replicas != *deployment.Spec.Replicas {
  		klog.V(4).Infof("Foo %s replicas: %d, deployment replicas: %d", name, *foo.Spec.Replicas, *deployment.Spec.Replicas)
  		deployment, err = c.kubeclientset.AppsV1().Deployments(foo.Namespace).Update(context.TODO(), newDeployment(foo), metav1.UpdateOptions{})
  	}
  
  	// If an error occurs during Update, we'll requeue the item so we can
  	// attempt processing again later. This could have been caused by a
  	// temporary network failure, or any other transient reason.
  	if err != nil {
  		return err
  	}
  
  	// Finally, we update the status block of the Foo resource to reflect the
  	// current state of the world
  	err = c.updateFooStatus(foo, deployment)
  	if err != nil {
  		return err
  	}
  
    //事件记录
  	c.recorder.Event(foo, corev1.EventTypeNormal, SuccessSynced, MessageResourceSynced)
  	return nil
  }
  
  ```
  
  最后的事件记录，就是我们平时在describe命令里看到的
  
  ```yml
  Name:         example-foo
  Namespace:    default
  Labels:       <none>
  Annotations:  <none>
  API Version:  samplecontroller.k8s.io/v1alpha1
  Kind:         Foo
  Metadata:
    Creation Timestamp:  2023-03-20T02:36:35Z
    Generation:          1
    Managed Fields:
      API Version:  samplecontroller.k8s.io/v1alpha1
      Fields Type:  FieldsV1
      fieldsV1:
        f:status:
          .:
          f:availableReplicas:
      Manager:      ___go_build_sample_controller
      Operation:    Update
      Subresource:  status
      Time:         2023-03-20T02:36:35Z
      API Version:  samplecontroller.k8s.io/v1alpha1
      Fields Type:  FieldsV1
      fieldsV1:
        f:metadata:
          f:annotations:
            .:
            f:kubectl.kubernetes.io/last-applied-configuration:
        f:spec:
          .:
          f:deploymentName:
          f:replicas:
      Manager:         kubectl-client-side-apply
      Operation:       Update
      Time:            2023-03-20T02:36:35Z
    Resource Version:  43500
    UID:               fa2b93fb-5379-4a83-8676-ed78ea470680
  Spec:
    Deployment Name:  example-foo
    Replicas:         1
  Status:
    Available Replicas:  1
  Events:
    Type    Reason  Age                From               Message
    ----    ------  ----               ----               -------
    Normal  Synced  16m (x7 over 16m)  sample-controller  Foo synced successfully
  ```

### 应用中心查询应用状态

对于ArcherkKE应用中心需要实时查看各种资源以及Crd资源状态，只需要用Informer监听就可以了。

> 动态增加informer，官方问我们准备了dynamic包，使用起来几乎与informer无异
>
> [参考](https://juejin.cn/post/6965476212420378632)

## Kubebuilder

从上述的内容我们了解了一个controller的开发过程，但是好像还是很麻烦，例如crd编写，代码生成，最后controller如何打包发布。[kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)就是进一步帮助k8s developer开发controller(operator)的手脚架工具

### 让CM变成云原生CM

#### 构建CR

我们来创建一个 HUAYUN CloudManager应用，假设内部的CM应用只有frontend和Identity，设计CR如下

>cm 配置文件太多，会影响到deployment的ready副本数，这里front 就用nginx，identity 用busybox代替

```yml
apiVersion: cloudmanagercontroller.huayun.com/v1
kind: CloudManager
metadata:
  name: cloudmanager-sample
spec:
	front:
		deployment:
			#image: harbor.archeros.cn/dev/application-front:archerke3.0_aarch64_0306
			image: nginx:1.20
			name: ake-front
	identity:
		deployment:
			#image: harbor.archeros.cn/dev/identity:archerke3.0_aarch64_0306
			image: busybox:1.32.1
			name: cm-identity
	db:
		statefulSet:
			image: harbor.archeros.cn/library/mariadb:v10.3.35_aarch64_health
			name: cm-db
			storage:
    		class: local-path
    		accessModes: ReadWriteOnce
    		size: 20Gi
    rootPass: '123456'
```

#### 创建operator project

```shell
(base) zero@ZerodeMacBook-Pro cm-operator % go mod init huayun.com/cm-operator
go: creating new go.mod: module huayun.com/cm-operator
## 使用kubebuilder init命令，指令domain为 huayun.com
(base) zero@ZerodeMacBook-Pro cm-operator % kubebuilder init --domain huayun.com
WARN[0000] the platform of this environment (darwin/arm64) is not suppported by kustomize v3 (v3.8.7) which is used in this scaffold. You will be unable to download a binary for the kustomize version supported and used by this plugin. The currently supported platforms are: ["linux/amd64" "linux/arm64" "darwin/amd64"]
Writing kustomize manifests for you to edit...
Writing scaffold for you to edit...
Get controller runtime:
$ go get sigs.k8s.io/controller-runtime@v0.14.1
go: downloading sigs.k8s.io/controller-runtime v0.14.1
go: downloading k8s.io/apimachinery v0.26.0
go: downloading github.com/prometheus/client_golang v1.14.0
go: downloading k8s.io/client-go v0.26.0
go: downloading k8s.io/utils v0.0.0-20221128185143-99ec85e7a448
go: downloading k8s.io/api v0.26.0
go: downloading k8s.io/component-base v0.26.0
go: downloading golang.org/x/time v0.3.0
go: downloading k8s.io/apiextensions-apiserver v0.26.0
go: downloading golang.org/x/sys v0.3.0
go: downloading github.com/matttproud/golang_protobuf_extensions v1.0.2
go: downloading golang.org/x/net v0.3.1-0.20221206200815-1e63c2f08a10
go: downloading golang.org/x/term v0.3.0
go: downloading github.com/fsnotify/fsnotify v1.6.0
go: downloading golang.org/x/text v0.5.0
Update dependencies:
$ go mod tidy
go: downloading go.uber.org/zap v1.24.0
go: downloading github.com/onsi/ginkgo/v2 v2.6.0
go: downloading github.com/onsi/gomega v1.24.1
go: downloading go.uber.org/goleak v1.2.0
Next: define a resource with:
$ kubebuilder create api
(base) zero@ZerodeMacBook-Pro cm-operator %
```

生成目录如下

```shell
├── Dockerfile
├── Makefile
├── PROJECT
├── README.md
├── config
│   ├── crd
│   ├── default
│   ├── manager
│   ├── prometheus
│   └── rbac
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
└── main.go ##程序入口
```

#### 创建api 和 controller

```go
(base) zero@ZerodeMacBook-Pro cm-operator % kubebuilder create api --group cloudmanagercontroller --version v1 --kind CloudManager
Create Resource [y/n]
y
Create Controller [y/n]
y
Writing kustomize manifests for you to edit...
Writing scaffold for you to edit...
api/v1/cloudmanager_types.go
controllers/cloudmanager_controller.go
Update dependencies:
$ go mod tidy
Running make:
$ make generate
mkdir -p /Users/zero/Projects/Personal/cm-operator/bin
test -s /Users/zero/Projects/Personal/cm-operator/bin/controller-gen && /Users/zero/Projects/Personal/cm-operator/bin/controller-gen --version | grep -q v0.11.1 || \
	GOBIN=/Users/zero/Projects/Personal/cm-operator/bin go install sigs.k8s.io/controller-tools/cmd/controller-gen@v0.11.1
go: downloading sigs.k8s.io/controller-tools v0.11.1
go: downloading golang.org/x/tools v0.4.0
go: downloading github.com/gobuffalo/flect v0.3.0
go: downloading k8s.io/utils v0.0.0-20221107191617-1a15be271d1d
go: downloading golang.org/x/net v0.4.0
/Users/zero/Projects/Personal/cm-operator/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
Next: implement your new API and generate the manifests (e.g. CRDs,CRs) with:
$ make manifests
```

命令执行了以下动作

- 在PROJECT文件中增加API资源声明。

  ```yml
  # Code generated by tool. DO NOT EDIT.
  # This file is used to track the info used to scaffold your project
  # and allow the plugins properly work.
  # More info: https://book.kubebuilder.io/reference/project-config.html
  domain: huayun.com
  layout:
  - go.kubebuilder.io/v3
  projectName: cm-operator
  repo: huayun.com/cm-operator
  resources:
  - api:
      crdVersion: v1
      namespaced: true
    controller: true
    domain: huayun.com
    group: cloudmanagercontroller
    kind: CloudManager
    path: huayun.com/cm-operator/api/v1
    version: v1
  version: "3"
  ```

- 新增CRD和CR描述文件

  ```shell
  config
  ├── crd
  │   ├── kustomization.yaml
  │   ├── kustomizeconfig.yaml
  │   └── patches
  │       ├── cainjection_in_cloudmanagers.yaml
  │       └── webhook_in_cloudmanagers.yaml
  ├── manager
  │   ├── kustomization.yaml
  │   └── manager.yaml
  ├── rbac
  │   ├── auth_proxy_client_clusterrole.yaml
  │   ├── auth_proxy_role.yaml
  │   ├── auth_proxy_role_binding.yaml
  │   ├── auth_proxy_service.yaml
  │   ├── cloudmanager_editor_role.yaml
  │   ├── cloudmanager_viewer_role.yaml
  │   ├── kustomization.yaml
  │   ├── leader_election_role.yaml
  │   ├── leader_election_role_binding.yaml
  │   ├── role_binding.yaml
  │   └── service_account.yaml
  └── samples
      └── cloudmanagercontroller_v1_cloudmanager.yaml
  
  ```

- 新增api的类型目录

  ```shell
  api
  └── v1
      ├── cloudmanager_types.go
      ├── groupversion_info.go
      └── zz_generated.deepcopy.go
  
  ```

我们可以修改cloudmanager_types.go文件，增加custom spec

根据我们设计的Crd，修改如下

```go
// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// CloudManagerSpec defines the desired state of CloudManager
type CloudManagerSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	Front    *FrontSpec    `json:"front"`
	Identity *IdentitySpec `json:"identity"`
	Db       *DbSpec       `json:"db"`
}

type FrontSpec struct {
	Deployment *DeploymentSpec `json:"deployment"`
}
type IdentitySpec struct {
	Deployment *DeploymentSpec `json:"deployment"`
}
type DbSpec struct {
	StatefulSet *StatefulSetSpec `json:"statefulSet"`
}

type StorageSpec struct {
	Class       string                            `json:"class"`
	AccessModes corev1.PersistentVolumeAccessMode `json:"accessModes"`
	Size        resource.Quantity                 `json:"size"`
}

type DeploymentSpec struct {
	Image string `json:"image"`
	Name  string `json:"name"`
}
type StatefulSetSpec struct {
	Image   string       `json:"image"`
	Name    string       `json:"name"`
	Storage *StorageSpec `json:"storage"`
}

// CloudManagerStatus defines the observed state of CloudManager
type CloudManagerStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file
	Phase string `json:"phase,omitempty"`
}

//+kubebuilder:object:root=true
//+kubebuilder:subresource:status

// CloudManager is the Schema for the cloudmanagers API
// 增加打印列
// +kubebuilder:printcolumn:name="Status",type=string,JSONPath=`.status.phase`
type CloudManager struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   CloudManagerSpec   `json:"spec,omitempty"`
	Status CloudManagerStatus `json:"status,omitempty"`
}

//+kubebuilder:object:root=true

// CloudManagerList contains a list of CloudManager
type CloudManagerList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []CloudManager `json:"items"`
}

func init() {
	SchemeBuilder.Register(&CloudManager{}, &CloudManagerList{})
}
```

- main.go启动时创建controller

  ```go
  
  	if err = (&controllers.CloudManagerReconciler{
  		Client: mgr.GetClient(),
  		Scheme: mgr.GetScheme(),
  	}).SetupWithManager(mgr); err != nil {
  		setupLog.Error(err, "unable to create controller", "controller", "CloudManager")
  		os.Exit(1)
  	}
  ```

- 每次修改API定义后，需要执行命令自动重新生成代码和CRD：

  ```
  $ make generate
  $ make manifests
  ```

#### 实现Controller

假设一个cm实例创建时，要完成如下动作

- 2个Deployment，包括frontend和identity
- 1个Statefulset，包括mariadb

```go
/*
Copyright 2023.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package controllers

import (
	"context"
	"github.com/go-logr/logr"
	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/tools/record"

	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"

	cloudmanagercontrollerv1 "huayun.com/cm-operator/api/v1"
)

// CloudManagerReconciler reconciles a CloudManager object
type CloudManagerReconciler struct {
	client.Client
	Scheme   *runtime.Scheme
	Recorder record.EventRecorder
}

//+kubebuilder:rbac:groups=cloudmanagercontroller.huayun.com,resources=cloudmanagers,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=cloudmanagercontroller.huayun.com,resources=cloudmanagers/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=cloudmanagercontroller.huayun.com,resources=cloudmanagers/finalizers,verbs=update

// Reconcile is part of the main kubernetes reconciliation loop which aims to
// move the current state of the cluster closer to the desired state.
// TODO(user): Modify the Reconcile function to compare the state specified by
// the CloudManager object against the actual cluster state, and then
// perform operations to make the cluster state reflect the state specified by
// the user.
//
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.14.1/pkg/reconcile
func (r *CloudManagerReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	logger := log.FromContext(ctx)

	logger.WithValues("cloudManager", req.Name)

	/**
	phase1  fetch cloudManager by name
	*/
	var cm cloudmanagercontrollerv1.CloudManager
	logger.Info("fetching cloudManager Resource")
	if err := r.Get(ctx, req.NamespacedName, &cm); err != nil {
		logger.Error(err, "unable fetch cloudManager")
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	/**
	phase2 clean up old resource which had owned by Cm Resource
	*/
	if err := r.cleanupOwnedResources(ctx, logger, &cm); err != nil {
		logger.Error(err, "failed to clean up old resources for this cloudManager")
		return ctrl.Result{}, err
	}

	/*
		phase3  create or update front identity db
	*/
	frontDeploymentName := cm.Spec.Front.Deployment.Name
	frontDeploy := &appsv1.Deployment{
		ObjectMeta: metav1.ObjectMeta{
			Name:      frontDeploymentName,
			Namespace: req.Namespace,
		},
	}

	//create or update front deployment
	if _, err := ctrl.CreateOrUpdate(ctx, r.Client, frontDeploy, func() error {
		labels := map[string]string{
			"controller": req.Name,
			"app":        "front",
		}
		if frontDeploy.Spec.Selector == nil {
			frontDeploy.Spec.Selector = &metav1.LabelSelector{MatchLabels: labels}
		}
		// set labels to template.objectMeta for our deployment
		if frontDeploy.Spec.Template.ObjectMeta.Labels == nil {
			frontDeploy.Spec.Template.ObjectMeta.Labels = labels
		}

		containers := []corev1.Container{
			{
				Name:  "front-container",
				Image: cm.Spec.Front.Deployment.Image,
			},
		}
		if frontDeploy.Spec.Template.Spec.Containers == nil {
			frontDeploy.Spec.Template.Spec.Containers = containers
		}

		if err := ctrl.SetControllerReference(&cm, frontDeploy, r.Scheme); err != nil {
			logger.Error(err, "unable to set ownerReference from CloudManager to FrontDeployment")
			return err
		}

		return nil
	}); err != nil {
		logger.Error(err, "unable to ensure frontDeployment is correct")
		return ctrl.Result{}, err
	}

	//check front status
	if err := r.Get(ctx, types.NamespacedName{Name: frontDeploymentName, Namespace: req.Namespace}, frontDeploy); err != nil {
		logger.Error(err, "failed to get resource", "component", frontDeploymentName)
		return ctrl.Result{}, err
	}
	if frontDeploy.Status.ReadyReplicas < frontDeploy.Status.Replicas/2+1 {
		return ctrl.Result{Requeue: true}, nil
	}

	//create or update  identity deployment
	identityDeploymentName := cm.Spec.Identity.Deployment.Name
	identityDeploy := &appsv1.Deployment{
		ObjectMeta: metav1.ObjectMeta{
			Name:      identityDeploymentName,
			Namespace: req.Namespace,
		},
	}

	if _, err := ctrl.CreateOrUpdate(ctx, r.Client, identityDeploy, func() error {
		labels := map[string]string{
			"controller": req.Name,
			"app":        "identity",
		}
		if identityDeploy.Spec.Selector == nil {
			identityDeploy.Spec.Selector = &metav1.LabelSelector{MatchLabels: labels}
		}
		// set labels to template.objectMeta for our deployment
		if identityDeploy.Spec.Template.ObjectMeta.Labels == nil {
			identityDeploy.Spec.Template.ObjectMeta.Labels = labels
		}

		containers := []corev1.Container{
			{
				Name:    "identity-container",
				Image:   cm.Spec.Identity.Deployment.Image,
				Command: []string{"/bin/sh", "-c", "sleep 60m"},
			},
		}
		if identityDeploy.Spec.Template.Spec.Containers == nil {
			identityDeploy.Spec.Template.Spec.Containers = containers
		}

		if err := ctrl.SetControllerReference(&cm, identityDeploy, r.Scheme); err != nil {
			logger.Error(err, "unable to set ownerReference from CloudManager to IdentityDeployment")
			return err
		}

		return nil
	}); err != nil {
		logger.Error(err, "unable to ensure identityDeployment is correct")
		return ctrl.Result{}, err
	}
	//check  identity status
	if err := r.Get(ctx, types.NamespacedName{Name: identityDeploymentName, Namespace: req.Namespace}, identityDeploy); err != nil {
		logger.Error(err, "failed to get resource", "component", identityDeploymentName)
		return ctrl.Result{}, err
	}
	if identityDeploy.Status.ReadyReplicas < identityDeploy.Status.Replicas/2+1 {
		return ctrl.Result{Requeue: true}, nil
	}

	//create or update db auth secret
	dbAuthSecretName := cm.Spec.Db.StatefulSet.Name + "auth"
	dbAuthSecret := &corev1.Secret{
		ObjectMeta: metav1.ObjectMeta{
			Name:      dbAuthSecretName,
			Namespace: req.Namespace,
		},
	}
	if _, err := ctrl.CreateOrUpdate(ctx, r.Client, dbAuthSecret, func() error {
		dbAuthSecret.Data = map[string][]byte{
			"username": []byte("root"),
			"password": []byte(cm.Spec.Db.RootPass),
		}
		if err := ctrl.SetControllerReference(&cm, dbAuthSecret, r.Scheme); err != nil {
			logger.Error(err, "unable to set ownerReference from CloudManager to DbStatefulSet")
			return err
		}

		return nil
	}); err != nil {
		logger.Error(err, "unable to ensure DbStatefulSet is correct")
		return ctrl.Result{}, err
	}

	//create or update db  statefulSet
	dbStatefulSetName := cm.Spec.Db.StatefulSet.Name
	dbStatefulSet := &appsv1.StatefulSet{
		ObjectMeta: metav1.ObjectMeta{
			Name:      dbStatefulSetName,
			Namespace: req.Namespace,
		},
	}
	if _, err := ctrl.CreateOrUpdate(ctx, r.Client, dbStatefulSet, func() error {
		labels := map[string]string{
			"controller": req.Name,
			"app":        "db",
		}
		if dbStatefulSet.Spec.Selector == nil {
			dbStatefulSet.Spec.Selector = &metav1.LabelSelector{MatchLabels: labels}
		}
		// set labels to template.objectMeta for our statefulSet
		if dbStatefulSet.Spec.Template.ObjectMeta.Labels == nil {
			dbStatefulSet.Spec.Template.ObjectMeta.Labels = labels
		}
		containers := []corev1.Container{
			{
				Name:  "db-container",
				Image: cm.Spec.Db.StatefulSet.Image,
				Env: []corev1.EnvVar{
					{
						Name: "MARIADB_ROOT_PASSWORD",
						ValueFrom: &corev1.EnvVarSource{
							SecretKeyRef: &corev1.SecretKeySelector{
								Key: "password",
								LocalObjectReference: corev1.LocalObjectReference{
									Name: dbAuthSecretName,
								},
							},
						},
					},
				},
			},
		}
		if dbStatefulSet.Spec.Template.Spec.Containers == nil {
			dbStatefulSet.Spec.Template.Spec.Containers = containers
		}

		dbStatefulSet.Spec.VolumeClaimTemplates = []corev1.PersistentVolumeClaim{
			volumeClaimTemplates(&cm),
		}

		if err := ctrl.SetControllerReference(&cm, dbStatefulSet, r.Scheme); err != nil {
			logger.Error(err, "unable to set ownerReference from CloudManager to DbStatefulSet")
			return err
		}

		return nil
	}); err != nil {
		logger.Error(err, "unable to ensure DbStatefulSet is correct")
		return ctrl.Result{}, err
	}

	if err := r.Get(ctx, types.NamespacedName{Name: dbStatefulSetName, Namespace: req.Namespace}, dbStatefulSet); err != nil {
		logger.Error(err, "failed to get resource", "component", dbStatefulSetName)
		return ctrl.Result{}, err
	}
	if dbStatefulSet.Status.ReadyReplicas < dbStatefulSet.Status.Replicas/2+1 {
		return ctrl.Result{Requeue: true}, nil
	}

	/**
	phase3 update cm status
	*/
	cm.Status.Phase = "ready"
	err := r.Status().Update(ctx, &cm)
	if err != nil {
		logger.Error(err, "unable to update cloudManager status")
		return ctrl.Result{}, err
	}

	//create event
	r.Recorder.Eventf(&cm, corev1.EventTypeNormal, "Updated", "Update cm.status.phase: %s", cm.Status.Phase)

	return ctrl.Result{}, nil
}

// SetupWithManager sets up the controller with the Manager.
func (r *CloudManagerReconciler) SetupWithManager(mgr ctrl.Manager) error {

	r.Recorder = mgr.GetEventRecorderFor("CloudManager")

	// add  resource index to resource object which cloudManager resource owns
	if err := mgr.GetFieldIndexer().IndexField(context.TODO(), &appsv1.Deployment{}, resourceOwnerKey, func(rawObj client.Object) []string {
		// grab the deployment object, extract the owner...
		deployment := rawObj.(*appsv1.Deployment)
		owner := metav1.GetControllerOf(deployment)
		if owner == nil {
			return nil
		}
		// ...make sure it's a Foo...
		if owner.APIVersion != apiGVStr || owner.Kind != "CloudManager" {
			return nil
		}

		// ...and if so, return it
		return []string{owner.Name}
	}); err != nil {
		return err
	}
	if err := mgr.GetFieldIndexer().IndexField(context.TODO(), &appsv1.StatefulSet{}, resourceOwnerKey, func(rawObj client.Object) []string {
		// grab the deployment object, extract the owner...
		statefulSet := rawObj.(*appsv1.StatefulSet)
		owner := metav1.GetControllerOf(statefulSet)
		if owner == nil {
			return nil
		}
		// ...make sure it's a Foo...
		if owner.APIVersion != apiGVStr || owner.Kind != "CloudManager" {
			return nil
		}

		// ...and if so, return it
		return []string{owner.Name}
	}); err != nil {
		return err
	}

  //同时监听Service,StatefulSet,Deployment
	return ctrl.NewControllerManagedBy(mgr).
		For(&cloudmanagercontrollerv1.CloudManager{}).
		Owns(&corev1.Service{}).
		Owns(&appsv1.StatefulSet{}).
		Owns(&appsv1.Deployment{}).
		Complete(r)
}

// cleanupOwnedResources will delete any existing Deployment resources that
// were created for the given cm that no longer match the
// cm.spec.front.name / identity.name  /  db.name field.
func (r *CloudManagerReconciler) cleanupOwnedResources(ctx context.Context, log logr.Logger, cm *cloudmanagercontrollerv1.CloudManager) error {
	log.Info("finding existing Deployments for Foo resource")

	var deployments appsv1.DeploymentList
	if err := r.List(ctx, &deployments, client.InNamespace(cm.Namespace), client.MatchingFields(map[string]string{resourceOwnerKey: cm.Name})); err != nil {
		return err
	}

	// Delete deployment if the deployment name doesn't match cm.spec.front or identity name
	for _, deployment := range deployments.Items {
		if deployment.Name == cm.Spec.Front.Deployment.Name ||
			deployment.Name == cm.Spec.Identity.Deployment.Name {
			// If this deployment's name matches the one on the Cm resource
			// then do not delete it.
			continue
		}

		// Delete old deployment object which doesn't match cm.spec.front or identity name
		if err := r.Delete(ctx, &deployment); err != nil {
			log.Error(err, "failed to delete Deployment resource")
			return err
		}

		log.Info("delete deployment resource: " + deployment.Name)
		r.Recorder.Eventf(cm, corev1.EventTypeNormal, "Deleted", "Deleted deployment %q", deployment.Name)
	}
	var statefulSetList appsv1.StatefulSetList
	if err := r.List(ctx, &statefulSetList, client.InNamespace(cm.Namespace), client.MatchingFields(map[string]string{resourceOwnerKey: cm.Name})); err != nil {
		return err
	}

	// Delete statefulSet if the statefulSet name doesn't match cm.spec.db.name
	for _, statefulSet := range statefulSetList.Items {
		if statefulSet.Name == cm.Spec.Db.StatefulSet.Name {
			// If this statefulSet's name matches the one on the Cm resource
			// then do not delete it.
			continue
		}

		// Delete old statefulSet object which doesn't match cm.spec.deploymentName
		if err := r.Delete(ctx, &statefulSet); err != nil {
			log.Error(err, "failed to delete StatefulSet resource")
			return err
		}

		log.Info("delete deployment resource: " + statefulSet.Name)
		r.Recorder.Eventf(cm, corev1.EventTypeNormal, "Deleted", "Deleted statefulSet %q", statefulSet.Name)
	}

	return nil
}

var (
	resourceOwnerKey = ".metadata.controller"
	DataVolumeName   = "datadir"
	apiGVStr         = cloudmanagercontrollerv1.GroupVersion.String()
)

func volumeClaimTemplates(cr *cloudmanagercontrollerv1.CloudManager) corev1.PersistentVolumeClaim {
	pvc := corev1.PersistentVolumeClaim{
		ObjectMeta: metav1.ObjectMeta{
			Namespace:   cr.Namespace,
			Name:        DataVolumeName,
			Annotations: map[string]string{},
			Labels:      map[string]string{},
			OwnerReferences: []metav1.OwnerReference{
				*metav1.NewControllerRef(cr, schema.GroupVersionKind{
					Group:   cr.GroupVersionKind().Group,
					Version: cr.GroupVersionKind().Version,
					Kind:    cr.Kind,
				}),
			},
		},
		Spec: corev1.PersistentVolumeClaimSpec{
			StorageClassName: &cr.Spec.Db.StatefulSet.Storage.Class,
			AccessModes: []corev1.PersistentVolumeAccessMode{
				cr.Spec.Db.StatefulSet.Storage.AccessModes,
			},
			Resources: corev1.ResourceRequirements{
				Requests: corev1.ResourceList{
					corev1.ResourceStorage: cr.Spec.Db.StatefulSet.Storage.Size,
				},
			},
		},
	}
	return pvc
}

```

·执行

```
// 请求成功，不再排队
return ctrl.Result{}, nil
// 请求失败，重新加入队列
return ctrl.Result{}, err
// 请求因某种原因需要重新加入队列
return ctrl.Result{Requeue: true}, nil
// 请求因某种原因，需要在指定时间后重新加入队列
return ctrl.Result{RequeueAfter: time.Second*5}, nil
```

#### 

![Screen Recording 2023-03-20 at 17.39.45](https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202303201747531.gif)

#### 小结

相较于使用helm部署cm，controller更细化控制，日志，通知等事件都控制在内部代码中，更适合一个独立应用。（沃趣rds也是通过operator，来创建所有的服务）