---
layout: post
title: The mechanism of client-go Informer
description: describe the mechanism of Informer including the implementation of method NewSharedIndexInformer, NewInformer and so on
---

## 1. Common Components

### 1.1 threadSafeMap
threadSafeMap stores each object and builds different type of index for it in order to access it flexibly.

#### Class Diagram
![]({{ site.url }}/images/2017-08-29-Client-go-tools-cache-Informer/client-go.tools.cache.ThreadSafeStore.svg)

#### Data Structrue Of theadSafeMap
![]({{ site.url }}/images/2017-08-29-Client-go-tools-cache-Informer/theadSafeMap_data_structure.png)
1.  'Items' stores each object which has a unique key.
1.  The keys of 'Indexers' are equal to the keys of 'Indices'
1.  The value of 'Indexers[type]' is an index function which will compute a list of indexes for an object which is stored in 'Items[key]' and the indexes compose the keys of 'Indices[type]'
1.  The value of 'Indices[type][index]' is a set of keys which is a subset of keys of 'Items'

### 1.2 cache
cache adds one more member of 'KeyFunc' which can compute the key of an object and stores the pair of key and object in threadSafeMap.Items. cache builds on top of threadSafeMap and leverages up its functions to implement interfaces.

#### Class Diagram
![]({{ site.url }}/images/2017-08-29-Client-go-tools-cache-Informer/client-go.tools.cache.cache.svg)

### 1.3 ListerWatcher
ListerWatcher define the methods which can list and watch a resource.

#### Class Diagram
![]({{ site.url }}/images/2017-08-29-Client-go-tools-cache-Informer/client-go.tools.cache.ListerWatcher.svg)

## 2. Controller

### Class Diagram
![]({{ site.url }}/images/2017-08-29-Client-go-tools-cache-Informer/client-go.tools.cache.Controller_simple.svg)

### Create an instance of Conroller
code path: staging\src\k8s.io\client-go\tools\cache\controller.go

method: New

codes:
```
func New(c *Config) Controller {
    ctlr := &controller{
        config: *c,
        clock:  &clock.RealClock{},
    }
    return ctlr
}
```

### Components of Controller

#### 2.1 Config 
Config is used as the parameter for creating the instance of Controller.

Config.Queue: it stores the events of resource and behaves as the queue.

Config.ListerWatcher: it is the instance of ListWatch. see 1.3 above.

Config.ProccessFunc: it is a function which processes the event of resource.

Config.ObjectType: it it the resource type to be listed and watched

#### 2.2 Reflector
Reflector works as the producer which watches the resource and stores the events in the Reflector.Store.

Reflector watches the resource as the following flow.
1.  invokes Reflector.ListerWatcher.List to list all the objectis and get the current resource version
1.  invokes Reflector.ListerWatcher.Watch to watch the resource circularly.

#### 2.3 DeltaFIFO
DeltaFIFO has two roles that one is queue and another one is store. First, it stores and fetches the
object by the way of queue. Second, it implements the interface of 'Store' that can add/update/delelete/get object.
```
The method of Update/Delete of DeltaFIFO does not update or delete the object stored in it,
but it add an event of update/delete in DeltaFIFO. So DeltaFIFO should work with Reflector only.
```

##### Class Diagram
![]({{ site.url }}/images/2017-08-29-Client-go-tools-cache-Informer/client-go.tools.cache.DeltaFIFO.svg)

## 3. SharedIndexInformer

### Class Diagram
![]({{ site.url }}/images/2017-08-29-Client-go-tools-cache-Informer/client-go.tools.cache.SharedInformer_simple.svg)

### Create an instance of SharedIndexInformer
code path: staging\src\k8s.io\client-go\tools\cache\shared_informer.go

method   : NewSharedIndexInformer

slice of codes :
```
sharedIndexInformer := &sharedIndexInformer{
    processor:                       &sharedProcessor{clock: realClock},
    indexer:                         NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, indexers),
    listerWatcher:                   lw,
    objectType:                      objType,
    resyncCheckPeriod:               defaultEventHandlerResyncPeriod,
    defaultEventHandlerResyncPeriod: defaultEventHandlerResyncPeriod,
    cacheMutationDetector:           NewCacheMutationDetector(fmt.Sprintf("%T", objType)),
    clock: realClock,
}
```

### Components Of SharedIndexInformer

#### 3.1 processorListener

##### Class Diagram
![]({{ site.url }}/images/2017-08-29-Client-go-tools-cache-Informer/client-go.tools.cache.processorListener.svg)

##### Principle
![]({{ site.url }}/images/2017-08-29-Client-go-tools-cache-Informer/processorListener_principle.png)
1.  The method of 'Add' adds an event into the chanel of 'addCh'
1.  The method of 'Pop' pops the event from 'addCh' and sends it to the chanel of 'nextCh'
1.  The method of 'Run' traverses 'nextCh' and invokes the corresponding interfaces of 'ResourceEventHandler' by passing the event

```
The method of 'Pop' defines a buffer of event named 'pendingNotifications' which
will store the event when the chanel of 'nextCh' is full.
```

#### 3.2 sharedProcessor

##### Class Diagram
![]({{ site.url }}/images/2017-08-29-Client-go-tools-cache-Informer/client-go.tools.cache.sharedProcessor.svg)

##### Principle
sharedProcessor builds upon processorListener and includes several instances of processorListener. sharedProcessor,distibute invokes processorListener.Add to add an event, and sharedProcessor.run
invokes processorListener.run and processorListener.pop to execute the event.

#### 3.3 CacheMutationDetector
As its name said, it is used to detect whether the cache has been mutated.

##### Class Diagram
![]({{ site.url }}/images/2017-08-29-Client-go-tools-cache-Informer/client-go.tools.cache.CacheMutationDetector.svg)

##### Principle
The method of 'AddObject' makes a copy of object and stores the pair of object and its copy in 'cachedObjs'. The method of 'Run' traverse 'cachedObjs' and checks whether any object has been mutated.
