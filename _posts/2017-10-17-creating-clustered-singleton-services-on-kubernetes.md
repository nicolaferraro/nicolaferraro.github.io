---
title:  "Creating Clustered Singleton Services on Kubernetes"
modified: 2017-10-17 08:50:00 +0200
last_modified_at: 2017-10-17 08:50:00 +0200
tags: [Kubernetes, Openshift, Apache Camel, JBoss Fuse, Spring-Boot, Java]
categories: [Dev]
header:
    image: post-logo-kubernetes.png
    teaser: post-logo-kubernetes.png
---
Creating a cluster of related containers is really easy with Openshift and Kubernetes.
Resources such as *Deployment* support scaling to multiple instances natively. Services and load balancers are provided for free.
But any time you create a cluster of containers, as soon as your application becomes more complex than a "Hello World!", 
there's a question that often arises: "How can I run a single instance of a component in the whole cluster?"

## Why do you need a singleton service? 

Think to a common scenario where you have an application running on Openshift with `n` replicas. It's not difficult to imagine cases where you need a singleton service.

### Scenario 1

Suppose that you want to do **scheduled operations on a database**. For sure, you don't want all the pods of the same application do the same batch operation together.

For this task there are Kubernetes `CronJob` resources, but if you don't want to *start a different pod for every task you create*, they are not the perfect solution.
`CronJob` resources require also some additional effort from a management point of view.

You may want to schedule tasks in your application. E.g. in spring-boot you can use the `@Scheduled` annotation:

```java
@Component
public class ScheduledTasks {

    @Scheduled(fixedRate = 5000)
    public void doTask() {
        // do something
    }
}
```

But if you do this, each running pod will call it's `doTask` method. And you don't want this.

### Scenario 2
 
Suppose that you need to **poll data from a remote REST API** and process them. But you **cannot do it concurrently**, because the API supports a **single consumer**.
For example, [Telegram](https://core.telegram.org/bots/api) (the popular messaging app) has a polling api that does not support multiple consumers on the same chat id.   

You can run another pod that consumes data from Telegram and push messages to a queue/topic. Or, if you had the possibility to run a singleton service, you could do it from the same app.

### Scenario 3

Suppose that you want to expose multiple istances of your REST services, but you want that certain operations are **executed in serial order by a single piece of code**.

You have multiple instances, so you need to push data e.g. to a messaging broker and **another pod (or deployment with replicas=1)** would process them.
But if you had a **clustered singleton** feature, you could do it in the same app.
 
Now use your imagination. There are a lot of other cases...

## How can you create it?

Camel 2.20 has been released few days ago. If you have time, you can take a look at the [huge list of release notes](https://issues.apache.org/jira/secure/ReleaseNote.jspa?version=12337871&projectId=12311211).
Or, for something more human friendly, there's [this post by Claus Ibsen](http://www.davsclaus.com/2017/10/apache-camel-220-released-whats-new.html).

One of the improvement introduced in Camel 2.20 is the new **Camel Cluster Service** (thanks to [Luca Burgazzoli](https://lburgazzoli.github.io/)). We designed some general purpose interfaces that can be 
used to **receive simple notifications when your specific instance of a service becomes the leader/master of the whole cluster**.
The cluster here is an abstract term: you define it. But often in Kubernetes/Openshift, a cluster is made by all the pods that compose the same deployment.

Camel supports multiple cluster services: atomix, consul, file, **kubernetes** and zookeeper. The kubernetes cluster service is just one of them and we'll use it to create a clustered singleton easily.

## I've seen this before... 

In case you were asking: this is the successor of the **Camel Master Route** concept available in JBoss Fuse.
We'll see a *new* Camel master route shortly. But the new service is much more powerful than the classic master component.

Yes, it will be available in next version of Fuse (Fuse 7.0, most probably). 

And yes: the *initial* code for the Kubernetes leader election has been [donated to Apache Camel by the Fabric8 team from the camel-master project](https://github.com/fabric8io/fabric8-ipaas/tree/master/camel-master).

And the original idea [comes from the Kubernetes code](https://github.com/kubernetes/kubernetes/blob/8c120dcf2f8c97da8143d36f26746c4f4685b828/staging/src/k8s.io/client-go/tools/leaderelection/leaderelection.go).  

## Ok, now show me the code!

The complete code is present on my Github repo [nicolaferraro/camel-leader-election](https://github.com/nicolaferraro/camel-leader-election).

You need just 3 dependencies in the pom.xml file:

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.camel</groupId>
            <artifactId>camel-master-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.camel</groupId>
            <artifactId>camel-kubernetes-starter</artifactId>
        </dependency>
    </dependencies>
```
 
We are running Camel on spring-boot and we also need the `camel-master-starter` to run **Camel master routes** and also `camel-kubernetes-starter` to tell Camel that 
we want to use the Kubernetes cluster service (that obviously works also on Openshift).

A bit of configuration is necessary. Here's the `application.properties` file:

```
# ...
camel.component.kubernetes.cluster.service.enabled=true

camel.component.kubernetes.cluster.service.cluster-labels[group]=${project.groupId}
camel.component.kubernetes.cluster.service.cluster-labels[app]=${project.artifactId}
```

**Note:** remember to [enable resource filtering](https://github.com/nicolaferraro/camel-leader-election/blob/35d789b176d1c15942f99112652a93e4f4662850/pom.xml#L57-L62) in the `pom.xml`.

The first line enables the Kubernetes cluster service. The other lines tell the cluster service that the cluster is composed 
of all pods having a Pod Label named `group` equals to the current Maven project `groupId` and a label named `app` equals to the Maven `artifactId`.
This is the **default mapping used by the Fabric8 Maven Plugin when creating Kubernetes/Openshift resources**.

Now you can just write a Camel master route:

```java
from("master:lock1:timer:clock")
  .log("Hello World!");
```

Note that this is a simple Camel route prefixed with `master:lock1`. Here `lock1` is the lock name used for synchronization among all pods: 
only the pod that is able to get the lock on `lock1` will start the route (implementation details later).

Singleton services are not limited to Camel routes: **you can also create a generic singleton service!**

For example, you can declare a service like:

```java
public class CustomService {

    public void start() {
        // called on the leader pod to *start* the service
    }

    public void stop() {
        // called on the leader pod to *stop* the service
    }

    // ...
}

```

You can bind the service to the global `CamelClusterService` instance on your application:

```java
@Configuration
class BeanConfiguration {

    @Bean
    public CustomService customService(CamelClusterService clusterService) throws Exception {
        CustomService service = new CustomService();

        clusterService.getView("lock2").addEventListener((CamelClusterEventListener.Leadership) (view, leader) -> {
            // here we get a notification of a change in the leadership
            boolean weAreLeaders = leader.isPresent() && leader.get().isLocal();
            
            if (weAreLeaders && !service.isStarted()) {
                service.start();
            } else if (!weAreLeaders && service.isStarted()) {
                service.stop();
            }
        });
        return service;
    }

}
```

**And that's all. The service will start only on one pod at time**.  

**Note for Openshift users**: the lock mechanism assumes that you're able to modify Kubernetes resources from whithin your pod. This is normally forbidden, unless you give special permissions to the pod.
The quickstart is already configured with a special `ServiceAccount` and `RoleBinding` (see [the fragments here](https://github.com/nicolaferraro/camel-leader-election/tree/master/src/main/fabric8)),
so you don't need to add them manually.

## Running the quickstart

1. Connect to a Openshift instance. You can use [Minishift](https://github.com/minishift/minishift/releases) for local development.
2. Clone the repo:

```
git clone git@github.com:nicolaferraro/camel-leader-election.git
```

3. Deploy:

```
# Enter the project dir
cd camel-leader-election
mvn fabric8:deploy
```

The deployment is already configured to run 3 istances of the application pod. If you enter the log (run `minishift console` to se the logs in the UI), 
you'll see that "Hello World!" is printed continuously in **only one pod**.
**If you kill that pod, after some seconds, another pod will start to print "Hello World!".**

## Where's the magic (implementation details)?

Leader Election is a special kind of consensus in distributed systems and you know that consensus is hard to achieve.
Fortunately Openshift and Kubernetes run a instance of Etcd and *Etcd is one of the few distributed datastores that implement Raft/Paxos correctly*.
So the idea: **why don't we use Etcd to do leader election?**

**Not so easy**. We cannot use Etcd directly inside Kubernetes, it's not there for application purposes. But we can use Kubernetes API to create/change 
resources such as `ConfigMaps`... and `ConfigMaps` are stored in Etcd...

There's a kind of **optimistic locking** mechanism in Kubernetes resources. Every resource has a `metadata.resourceVersion` field that can be used to ensure that only one pod
changes a `ConfigMap` at time. If two pods try to change the same ConfigMap, only one of them can complete the operation correctly.

### So, how can we leverage this feature?

The `camel-kubernetes` cluster service creates a `ConfigMap` named `leaders` (by default) and stores inside it the **pod name of the current owner of each lock declared in the application** (we used `lock1` for the master route and `lock2` for the custom service).

The protocol is (approximately) this:

- Every pod tries continuously to acquire the lock
- A pod can acquire the lock if the lock has not been created yet, or the current owner is not in the set of pods alive in the cluster (filtered with the labels we've set above)
- When a pod acquires the lock, it must wait for `x` seconds before starting the clustered services
- The pod that owns a specific lock must continuously check if he's still the owner of the lock
- When a pod loses the lock because another pod acquired it, it must shutdown the clustered services immediately
- When a pod cannot check if it's still the owner of a lock (because of a network partition or any other error), it must shutdown the clustered services after a timeout `y < x` from last successful check

The protocol outlined above ensures that **only one master is present in the cluster at time**.
**Even in case of network partition** (e.g. a physical node is completely isolated from the rest of the cluster) of a Kubernetes node that was hosting the leader, **the application will shut down the clustered services before a 
new election** can take effect.

For more details, [look at the code!](https://github.com/apache/camel/tree/master/components/camel-kubernetes/src/main/java/org/apache/camel/component/kubernetes/ha)
