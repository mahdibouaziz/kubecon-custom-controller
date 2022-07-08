# Writing Kube Controller for Everyone

[Vid refernece](https://www.youtube.com/watch?v=AUNPLQVxvmw&t=36s&ab_channel=CNCF%5BCloudNativeComputingFoundation%5D)

The `Control loop` is the core piece of a controller.

After we created the controller structure, initiated with some stuff, we just:

1. process every single item from the queue
2. do whatever you want to do with your controller
3. then push it back to the queue or ignore it
4. and then update the status and continue

This is the entire logic of every signle controller

# How to start

We have the [sample controller](https://github.com/kubernetes/sample-controller), it is ver simple and supposed to help you get started.

Writing controllers these days still requires writing some amount of boilerplate.

There are some projects that sets up the boilerplate for you like [kubebuilder](https://github.com/kubernetes-sigs/kubebuilder).

`kubebuilder` is very helpful if you're starting with building our own `CRDs`, it will generate the types that go for you, the framework for your controller, and you can focus only on **filling in the logic**. But it is nice to know the base details for what fits with which block and how the blocks should interact with each other.

# Control loop

```go
func (c * Ctrl) worker(){
    for c.processNextItem(){
    }
}

func (c *Ctrl) processNextItem(){
    item := c.queue.Get()
    err := c.syncHandler(item)
    c.hanfleErr(err, item)
}
```

Within that loop there are 3 elements that we will be focusing on:

- `queue`
- `Shared Informers`
- `syncHandler`

# Queue

The Queue is a very simple topic (FIFO).

You put stuff into the queue and you take it out in the sync loop and the work keeps going on over and over.

example of queue

```go
    queue=workqueue.NewNamedRateLimitingQueue(
        workqueue.DefaultControllerRateLimiter(),
        "foos"
    )
```

# Shared Informers

The most important piece of every single controller.

Shared informer is the **shared data cache** and it is **distributing the data** to all the **listeners** that are interested in knowing about the changes that are happening to those data.

Example of a podInformer

```go
    podInformer = InformerFactory.Core().V1().Pods()
```

The most importatnt part of the Shared Informers is the **event handlers**, this is how you register your interest in specific object upadates whether that will be acreation, update, delete of an object.

![alt text](./images/sharedinformerlistener.png?raw=true)

The other important element of the shared informer is the **listers**

```go
    podStore = podInformer.Lister()
```

The lister interface is different from the `client go`, but why would you use a `lister` an not a regular `client go`?

The reasons for using the `listers` is they are designed specifically to use used within controllers, they have the **access to the cache**.

Using `client go` you always hit the `API server`, and is you have a lot of deployments, you need to care about using the cache as much as possible rather than going back to the API server

# syncHandler

This is where you actually implement the **logic of your controller** (the main loop)

![alt text](./images/synchandler.png?raw=true)

The `queue` is not storing the actual objects, it is storing a `MetaNameSpaceKey` which is basically a **namespace** and the **name of the resource** bundled together.

There are specific helper methods that are designed to encode and decode those keys

1. the first invocation of every sync handler is to get the `namespace` of the resouce that you cant to work with and the specific `name` of the resource.
2. then you will get the object from the cache. (depending on your controller you might be getting more than just one object.)
3. here goes the most important thing of all.<br/>
   because you're using a **shared cache** all the resources you're getting are a `pointer` to the cache object, which means:
   - if you're just **reading** the object you're fine and don't have to do anything.
   - if you cant to **modify** the object, you need to invoke `DeepCopy` on the object. <br/>
     Be careful, `DeepCopy` is a very expensive operation ==> only do it if you will be modifiying the object.
4. now you should be focusing on your logic of the controller

# How the Shared informers are interacting with the controllers

![alt text](./images/explain-shared-informer.png?raw=true)

# Controller ground rules

- Always operate on a **signle item** at a time (single means one replication-controller, or one deployment-controller, ....)
- You need to ensure that you are n**ot hard-coding any ordering between the items that you are processing in the queue** (it should be entirely random - it should not be any interconnection between the objects that you are working on)
- Use **shared informers**
- At the begining of your controler (when you set up everything) you need to wait for the caches to be ready. There is a nicely funciton that you just need to **wait for to notify your controller that the caches are ready**.
- There are other actors in the system so you're not the only controller, be mindful when you're writing stuff to the API, your controller has to be prepared for the **conflicts** (for example when you're updating you resource there might be a conflict because somebody else in the meantime updated your resource) in this case you need to resync your objects again or repeat your update process.
- Ensure that you are providing to users a reasonable error messages for your controllers.
