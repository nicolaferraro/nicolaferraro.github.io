---
title:  "Introducing Camel K"
modified: 2018-10-15 15:00:00 +0200
last_modified_at: 2018-10-15 15:00:00 +0200
tags: [Apache Camel, Camel K, Openshift, Kubernetes, Knative, Serverless, JBoss Fuse]
categories: [Dev]
header:
    image: images/post-logo-apache-camel-d.png
    teaser: images/post-logo-apache-camel-d.png
---
Just few months ago, we were discussing about a new project that we could start as part 
of Apache Camel. A project with the potential to change the way people deal with integration.
That project is now here and it's called **"Apache Camel K"**.

The "K" in the title is an obvious reference to [**Kubernetes**](https://kubernetes.io/), you may think. But there's also a less-obvious reference to [**Knative**](https://cloud.google.com/knative/): a community 
project with the target of creating a common set of building blocks for **serverless** applications.
Yes, going "serverless" is the base idea that inspired many architectural decisions for Camel K.

But, let's take this one step at time...

## What is "Camel K"?

Apache Camel K is a lightweight cloud integration platform based on the Apache Camel framework. It **runs natively on Kubernetes and Openshift** and it's specifically designed for **serverless and microservice architectures**.
When I say "it runs", I mean "it runs, now, you can try it!". Just visit the homepage of the project on Github and follow the 
instructions: [**https://github.com/apache/camel-k**](https://github.com/apache/camel-k).

It's based on the "operator pattern" and leverages the [Operator SDK](https://github.com/operator-framework/operator-sdk) to perform
operations on Kubernetes resources (we define some [custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) beside the standard ones).
The operator is written in [Go](https://golang.org/) while the runtime is JVM based and leverages **all the 200+ components** already available 
in Apache Camel.

Like Kubernetes and OpenShift, also **Knative** will be a target platform in the near future.
Specifically, we're following the development of [Knative Eventing](https://github.com/knative/eventing) and [Knative Serving](https://github.com/knative/serving)
building blocks to provide a full support for them, once they reach an adequate level of maturity.

Camel K brings integration to the next level, but at the same time is a return back to the roots for the Camel project: the [**Enterprise Integration Patterns (EIP)**](https://www.enterpriseintegrationpatterns.com/patterns/messaging/).
Camel has been shaped around enterprise integration patterns since its inception and developers have created a DSL that often maps patterns in a 1:1 relationship.

I'm not exaggerating if I state that now: **the Camel DSL is the language of EIP**. It's (at least in my opinion)
the language that expresses better most of the patterns that were present in the original ["book of integration"](https://www.enterpriseintegrationpatterns.com/index.html), 
but also other patterns that have been added by the community during all these years.
And the community keeps adding patterns and new components in every release.

The idea of Camel K is simply stated: **let people use those Enterprise Integration Patterns natively on Kubernetes**, expressing them using the **poweful Camel DSL**.

If I should provide a **architectural overview** on Camel K, I'd draw the following diagrams:

<p style="text-align: center">
    <img src="/images/post-camel-k-architecture.png" alt="Deployment Models for Camel K"/>
</p>


Camel K is what you get if you take the **integration DSL distilled** from the rest of the framework
and offer a way to write integration code that is executed directly on a cloud platform:
it may be a **"modern" cloud** platform like Kubernetes or Openshift, or a **"futuristic" 
cloud** platform like Knative for serverless workloads (Knative can run on both OpenShift and Kubernetes and it's powered by [Istio](https://istio.io/)).

## How does it work?

Speaking technically, the starting point is writing the integration code that we want to run. For example: 

File: *integrate.groovy*
```groovy
// expose a rest endpoint that routes messages to a Kafka topic
rest().post("/resources")
  .route()
    .to("kafka:messages")
    
// transform all messages and publish them on a HTTP endpoint
from("kafka:messages")
  .transform()... // any kind of transformation
  .to("http://myendpoint/messages")
```

Integrations can range from simple [timer-to-log](https://github.com/apache/camel-k/blob/35aa1b3d39508ea901be7a7a1c5f4d256ce0eabb/runtime/examples/routes.js#L29) dummy examples
to complex processing workflows connecting several external systems, but you write them using the same Camel DSL.

Actually, I talked about "a" Camel DSL, but many of you already know that the Camel DSL is not a proper standalone language, but a set of primitives that can be used in **multiple programming languages**.
So far we support the following languages:
- **Groovy**: it is probably the best language for scripting and it is currently the preferred language for writing Camel K integration code
- **Kotlin**: yes, we support also Kotlin that offers a similar experience to Groovy
- **Java**: it's the classic Camel DSL that many of you already know
- **XML**: it's also a classic DSL adaptation and give its best when you plan to use one of the visual editing tools that already exist
- **JavaScript**: yes, we support it as well!

Let's not complicate things too much and consider one of the simplest integration for the moment.
The classic **"Camel Hello World"**:

File: *hello.groovy*
```groovy
from("timer:tick?period=3s")
  .setBody().constant("Hello World from Camel K!!!")
  .to("log:message")
```

A user that wants to run this integration on a cloud platform needs to download a small binary file that is available in the 
[release page on the Camel K Github repository](https://github.com/apache/camel-k/releases). It's called **kamel**.

The **kamel** binary contains also a command to prepare your Kubernetes cluster for running integrations.
Just run:

```
kamel install
```

Will take care of installing the Camel K CRDs, setting up privileges and create the operator (see next) in the current namespace.

**Important:** in some cluster configurations, you need to be a cluster admin to install a CRD (it's a operation that should be done only once for the entire cluster). The `kamel` binary will help you troubleshoot.
If you want to work on a development cluster like *Minishift* or *Minikube*, you can easily follow the [dev cluster setup guide](https://github.com/apache/camel-k/blob/master/docs/cluster-setup.adoc).

Once the cluster is prepared and the operator installed in the current namespace:

```
kamel run hello.groovy
```

And your done! 

This is what happens under the hood:

<p style="text-align: center">
    <img src="/images/post-camel-k-architecture-detail.png" alt="Camel K Architecture Details"/>
</p>

The **kamel** tool will sync your code with a Kubernetes custom resource of Kind **Integration** named **hello** (after the file name) in 
the current namespace.

The **Camel K Operator** is the component that makes all this possible by configuring all Kubernetes resources needed for running your integration.
I'll talk more about it later.

There exists also a **dev mode** that allow users to create integration incrementally and with **immediate "buildless" redeploys**.
I think a demo is worth 1000 words. 

## Demo

The following video shows an example of what you can do with Camel K.
It starts **from the installation** on a Minishift dev cluster, showing how to run a basic quickstart.
Then it proceeds with a **more complex example**.

The second integration shown in the video will connect a **Telegram** bot (you can interact with it through the Telegram app on your 
mobile phone) to a **Kafka** topic that will be used to buffer messages and to throttle them.
Messages will be received from Kafka again, filtered through one of the basic
**enterprise integration patterns** available out-of-the box in Apache Camel,
then **forwarded to a external HTTPS** endpoint.

All this is done in few minutes of video, and all integrations are created **incrementally**,
leveraging the new build engine behind Camel K that requires **just 1 second to redeploy** the integration 
after each change.

Take a look: 

<iframe width="560" height="315" src="https://www.youtube.com/embed/9Y5JfYiiBwM" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>


## Bringing "operators" to the next level

[Operator SDK](https://github.com/operator-framework/operator-sdk) is the framework
that makes it possible to create all Kubernetes resources needed for running the "Camel DSL script".

Operators are commonly used to install and configure applications or platforms on Kubernetes and Openshift.
They are the digital version of the "human operator" that once installed the application in legacy environments, 
making sure that everything is in place for the application to run.

We brought this concept to the **next level** in Camel K. The operator is **"intelligent"** and knows what you want to run. It **can understand the Camel DSL**.

So, **for example**, if you define a REST endpoint using the Camel REST DSL, the operator will make sure that 
your integration is exposed to the outside and it will create a Service and a Route on OpenShift to expose it, or a Ingress on vanilla Kubernetes.

And in the future, it will have more administrative responsibilities:
- It will expose webhooks on Kubernetes and activate the webhook registration with the external provider to receive data
- It will subscribe to feeds that you need to manage in your integration
- It will convert your "polling" routes into Kubernetes Cronjobs for optimizing resource utilization
- It will use a optimized Camel runtime platform under the hood if your routes support it
- It will... do a lot of useful things!

The operator will make sure that all you should do is writing your integration in a "Camel DSL Script", without caring about
any administrative operation. That's what we mean with **"intelligent operator"**.


## What's next

There are a lot of things coming. We keep updated the [projects section](https://github.com/apache/camel-k/projects) in the
github repository with the current areas we're working on. Those include the already mentioned work on **Knative** and also a 
**Web UI** for Camel K that will really rock!

We love contributions! If you're interested in the project, there are a lot of ways to contribute.

Meet us in our [dedicated Gitter room](https://gitter.im/apache/camel-k?utm_source=share-link&utm_medium=link&utm_campaign=share-link) for more information.

