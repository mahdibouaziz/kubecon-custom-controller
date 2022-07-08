# Writing-Custom-Controller

[Vid Reference](https://www.youtube.com/watch?v=_BuqPMlXfpE&ab_channel=CNCF%5BCloudNativeComputingFoundation%5D)

# What is a Controller?

A kuberentes Controller is an active reconciliation process:

1. Watch both the **desired state** and the **actual state**.
2. Change the actual state to be more like the desired state.

In a **pseudocode** this would look like a **loop** that just constantly is getting the desired state, getting the current state and then making the changes to bring those into sync

```go
for {
    desired := getDesiredState()
    current := getCurrentState()
    makeChanges(desired,current)
}
```

Inside of Kuberentes, we have the `kube-controller-manager` which inside of it has a whole bunch of different kinds of controllers:

- Deployment
- Daemon
- Node
- Service
- Endpoint
- ...

and they are all doing the reconciliation loop internally.

Outside of Kuberentes, we have the cluster application like

- Kube-DNS
- ingress
- flannel
- etcd-operator
- Prometheus-operator

# How can we use Controllers?

Without modifiying the core Kubernetes codebase you can:

- Extend funtionality of existing objects
- Add new concepts / functionality to cluster
- Automate administration of cluster applications
- Replace existing cluster components

# Building an example: Node Reboot Operator

We will build a **Node Reboot Operator** that will allows an administrator to:

- Trigger a node reboot via kubectl
- Support **rolling** reboots across the cluster.

The operator is going to be composed of a couple components:

- Reboot Agent (all nodes)
- Reboot Controller (single instance)
- They are going to communicate their actual and their desired state via annotaions:
  - reboot-now
  - reboot-in-progress
  - reboot-needed

## Reboot Agent:

- `Daemonset` with pod running on all nodes
- its job is to watch for an annotaion on its own Node object ==> watch for an annotation that says `reboot-now`
  - add `reboot-in-progress` annotation
  - remove `reboot-now` annotation
  - reboot node.

## Reboot Controller:

- Deployment (replica=1)
- Watches for nodes with `reboot-needed` annotation
- Counts `unavailable` nodes (`not-ready` or `reboot-in-progress`)
- While `unavailable < MaxUnavailable`
  - Remove `Reboot-needed` annotaion
  - Set `reboot-now` annotation

## Writing a custom controller

The core building blocks of writing a controller:

1. creating a K8s **client** to interact with API
2. Using the **Informer** for object cache & event handling
3. Communicating desired/actual state via annotaion

the Functional example is here [https://github.com/aaronlevy/kube-controller-demo]

## Reboot Agent - Determinating `self` node downward API

The first thing that we want to do it we need to determine ourself (our current node).

The way to do this:

- Populate an env variable called `NODE_NAME` and we're getting its `valueFrom` a field reference `fieldRef.fieldPath` the is referencing the `spec.nodeName`
  ![alt text](./images/determining-self.png?raw=true)

at the runtime when the `kubelet` runs that Pod its going to inject that environment variable with the correct information and then we as the authors of this controler can consume it in a couple of different ways:

- in our code: get the env variable `node := os.Getenv("NODE_NAME")`

## Kubernetes client

The next thing we need to do is create a client so that we can speak securely to the API server

![alt text](./images/k8s-client.png?raw=true)

and there is 2 main things in this code:

1. we are creating a `kubecfg` (client config) in 2 different ways, one is the `InClusterConfig` and the other creating a config using a `reference kube config file`
   - The `InClusterConfig` is generally what you're going to be using. <br/>
     when you launch a Pod in the cluster, its going to get a `sertvice account` injected into it automatically and that's going to have all of the inforamtion you need to be able to securely contact the API server (generally that's the right way of allowing your pod if its in the cluster to contact he API server)

At this point we have enough informations:

- we can securely contact the API
- we know who we are
- we could write some logic ....

## The problem when using the client without Informer

For example let's wrtie a controller that **print the label of our Node** and the **total number of Nodes**

![alt text](./images/k8s-client-print.png?raw=true)

But tose kind of operations `client.Get()`, `client.List()` can become expensive (always talk to the API server) ==> we need a **cache**.

we also want to be notified of object changes ==> we need **watches**

The solution is the `Informer`

## Creating an Informer (Note an informer is not a Shared Informer)

A simple overview of an informer

![alt text](./images/Creating-informer.png?raw=true)

calling a `NewInformer` need a `ListWatch` object, the Type of objects you care about operating on (in this case we care about nodes ==> its `v1.Node{}`), `resyncPeriod` and `ResourceEventHandlerFuncs`

the `ListWatch` object

![alt text](./images/listWatch.png?raw=true)

the `ResourceEventHandlerFuncs` function

![alt text](./images/ResourceEventHandlerFuncs.png?raw=true)

the `resyncPeriod`: UpdateFunc triggered for all objects at this interval (re-queues objects from cache)

### The result of the NewInformer

The NewInformer will return a `store` and a `controller`

## Pseudo Reboot Agent

We can actually start building our controller

We're going to tie into the `updateFn` function in the `informer` and we don't care when nodes get deleted (if they are deleted we don't care about deleting them) and when nodes get added (we don't want to try to reboot them immediately)

the job of the `updateFn` function is to look does the node have the annotation that says `reboot-now`, and if it does that means it should reboot, in this case we will:

1. get rid of the annotation `reboot-now`
2. update that in the API server (so that's reflected)
3. reboot

The client logic (pseudocode) is:

1. getSelf() - determine who I am
2. create a client
3. create an informer that will use our `self`, `client` and the `updateFn` function
4. and tell the controller to run

![alt text](./images/psuedo-code-reboot-agent.png?raw=true)

## Pseudo Reboot Controller

in this case we care about create, update, delete

we are going to write a `handler` function that can be used for `add`, `delete`, and for the `update` we only care about the new copy of the object

![alt text](./images/psuedo-code-reboot-controller1.png?raw=true)

in the handler function, we will

1. check if the `reboot-needed` annotaion exists
2. check the `getUnavailable() >= MaxUnavailable` nodes
3. delete the `reboot-needed` annotation
4. add the `reboot-now` annotation
5. update the object in the API and the agent will see this change and will reboot the node

![alt text](./images/psuedo-code-reboot-controller2.png?raw=true)

The `getUnavailable()` will show us how to use the cache

![alt text](./images/psuedo-code-reboot-controller3.png?raw=true)

## A few important Notes (Our reboot-controller has bugs)

- You shouldn't modify cash objects <br/>
  in the `handler` we changed directly the annotation in the cashe and if the update fails then our cache is now incorrect<br/>
  ![alt text](./images/psuedo-code-bug1.png?raw=true)<br/>
  The solution to this is using a `DeepCopy` before changing, then change the needed changes and update the new changed copy <br/>
  ![alt text](./images/psuedo-code-solution-to-bug1.png?raw=true)

- You shouldn't do for the most part any kind of operation until you've determined that your **cache is synced completely** <br/>
  ==> Once you start your controller you must ask if the cache has synced yet! and just don't start our processing until that happened <br/>
  ![alt text](./images/wait-to-sync.png?raw=true)
