---
title:  "Knative Camel Sources: Bringing Data to your Serverless Applications"
modified: 2019-11-09 12:30:00 +0100
last_modified_at: 2019-11-09 12:30:00 +0100
tags: [Knative, Apache Camel, Openshift, Kubernetes, Serverless, JBoss Fuse]
categories: [Dev]
header:
    image: post-logo-apache-camel-k-native.png
    teaser: post-logo-apache-camel-k-native.png
---

Knative is progressively becoming a fundamental platform for running serverless workloads on Kubernetes.
The secret sauce is most likely its flexibility. Differently from many other serverless platforms, Knative allows users to run 
generic containers and thus it's less opinionated on what the users are supposed to write their application code.
Users can write their services with their framework of choice and also pre-existing REST applications can obtain serverless features, 
such as auto-scaling and scaling to zero, without any change.

But the flexibility of Knative is not only related to what users can run on top of it. Knative has been developed from the 
ground up with extensibility in mind. Third-party projects are allowed and encouraged to extend Knative, through the many interfaces that it
defines. And this is what we've been doing in [Apache Camel K](https://camel.apache.org/camel-k/latest/) since the start of the 
Knative project: providing new features by extending the platform with the huge component base that only Apache Camel owns.

The work on [Knative Camel Sources](https://knative.dev/development/eventing/samples/apache-camel-source/) is now becoming mature, 
together with Knative itself. This article will explain Camel Sources and why they are fundamental for building modern real-world
serverless applications.

## What are Knative Sources, btw?

You may already know that Knative efforts can be roughly divided in two major components:
- [Knative Serving](https://knative.dev/development/serving/): provides Kubernetes primitives for defining auto-scaling container-based services.
- [Knative Eventing](https://knative.dev/development/serving/): provides Kubernetes primitives for creating event-based applications by binding event sources and their consumers.

Knative Sources are a fundamental **part of Knative Eventing** and they are the units responsible for bringing events inside the platform, 
so that they can be consumed by downstream services in the end.

[figure showing source and possible destinations]

A "Source" is a Kubernetes object that is capable of generating or relaying events to an "Addressable" endpoint. Examples of "Addressable" endpoints are:
- a Knative Serving Service: in this case the source will deliver events to the service directly as Cloud Events
- a Knative Eventing (Messaging) Channel: in this case, the events will be delivered by the source to the channel and then they will be forwarded
  to each subscriber of the Channel (i.e. any service that creates a Subscription to the channel) 
- the Knative Eventing Broker: in this case, events are pushed to a broker that will deliver them to any service that wants to receive events of a 
  specific type (i.e. declares interest in specific events via Triggers) 

For the reader that wants to look into the details, the list of resources that can receive events from a Source is not closed. 
All Kubernetes resources that adhere to the `Addressable` (TODO link addressable or SinkBinding) [Duck Type](https://www.youtube.com/watch?v=kldVg63Utuw) can receive events from a Source. 
Also, all sources share a common part of their shape, but can define a custom structure and implementation
(i.e. sources are also Duck Types: *"If it walks like a duck and it quacks like a duck, then it must be a duck"*).

Knative Eventing currently provides some sources as part of the main installation. They are:
- CronJobSource: mostly a test source that allows sending text data to an "Addressable" sink
- ContainerSource: allows to specify a user-defined Kubernetes Pod that generates events and forward them to the configured sink

The `ContainerSource` is a generic source and the most flexible one, but requires the user to package the source logic in a container image that 
is able to define his own configuration mechanism and also deal with low level details of cloud events.

The most interesting sources are present in the related [Knative-Contrib](https://github.com/knative/eventing-contrib) repository.
Those sources are more specific than the `ContainerSource`. At the time of writing, there are sources that allow 
to connect to Knative: AWS SQS queues, CouchDB, Kafka topics, Github/BitBucket/Gitlab notifications, GCP PubSub, Google Cloud notifications and Kubernetes events.

Ah, right... there are also [Camel Sources!](https://knative.dev/development/eventing/samples/apache-camel-source/)

As I'll explain in the remainder of the article, a `CamelSource` is something really easy to use but at the same time much powerful.
A `CamelSource` can use any mixture of the [300+ components](https://camel.apache.org/components/latest/) available in Apache Camel, so it can be used
to connect to ~~any~~ almost any system to Knative, including the ones for which there's a specific Knative source (like AWS SQS, CouchDB, Kafka, Github and Kubernetes, just to give you some examples).


## Installing Camel Sources

Camel Sources are released together with Knative in the [Knative-Contrib](https://github.com/knative/eventing-contrib) repository.
You can follow the installation instructions available in the [Knative Documentation](https://knative.dev/development/eventing/samples/apache-camel-source/)
in order to install them on any Kubernetes installation, but if you are running **OpenShift 4**, the best way to install Camel Sources 
is via the Operator Hub:


<p style="text-align: center">
    <img src="/images/post-camel-sources-operator-hub.png" alt="Camel K and Knative Camel Sources in Operator Hub"/>
    <caption align="bottom"><i>Camel K and Knative Camel Sources in Operator Hub</i></caption>
</p>

Camel Sources **require the following** software to be installed on the cluster:
- Knative Serving (or Openshift Serverless in Operator Hub that includes Knative Serving)
- Knative Eventing (installable from OperatorHub)
- Camel K (install it in "global" mode from OperatorHub)

After all dependencies are installed from OperatorHub, the **"Knative Apache Camel Operator" package can be finally installed** and it will
provide support for Camel Sources.

## Setting up a test project

We'll create a namespace named `camel` and install some test resources that will help us understand how Camel Sources work.

Once you're connected to your OpenShift cluster, you can create the namespace with:

```
oc create project camel
```

The command above will also set `camel` as default project for subsequent commands.

Let's now create some resources that will help us understand how CamelSources work.
Test resources have been added to the [resources.yaml](https://gist.githubusercontent.com/nicolaferraro/8dcf344e40d4dad86abf718a5f7a703d/raw/07680229603f3fd21d5f4593cac497ebc43cc169/resources.yaml)
 file.

To create test resources:

```
oc apply -f resources.yaml
```
 
Resources that will be installed include:
- A InMemoryChannel named `camel-test`
- A Knative Service named `camel-event-display`
- A Knative Subscription that will subscribe the service to the channel
- A Camel K Integration platform that will initialize the Camel K engine

We're now ready to experiment with Camel Sources.

## CamelSource #1: "Hello World"

Let's start with the simplest Camel source, the "Hello World".

*camel-timer-source.yaml*
```
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: CamelSource
metadata:
  name: camel-timer-source
spec:
  source:
    flow:
      from:
        uri: timer:tick
        parameters:
          period: 3s
        steps:
          - set-body:
              constant: Hello world!
  sink:
    ref:
      apiVersion: messaging.knative.dev/v1alpha1
      kind: InMemoryChannel
      name: camel-test
```

Let's analyze this source. It's a Kubernetes resource of kind `CamelSource` containing only YAML configuration.

The `spec` part is divided in two major sections:
- `source`: where the user can define the Camel route (TODO link) in YAML DSL that will generate the events. This part is specific for
  Camel sources.
- `sink`: where the user can define which Addressable object will receive the events (i.e. the `InMemoryChannel` named `camel-test` that we've created before).
  This part is present in all Knative sources. 


The most interesting part for our explanation, is the `source` part.

The `uri` field (`flow` -> `from` -> `uri`) determines the starting component. In this case the `timer:tick` scheme belongs to the [Camel Timer Component](https://camel.apache.org/components/latest/timer-component.html).

The `parameters` field (`flow > from > parameters`) allow to configure in a key/value style the options of the consumer endpoint.
In this case, the `period` key is specific to the [Camel Timer Component](https://camel.apache.org/components/latest/timer-component.html) and
allow to configure the period between two executions of the Camel route.

the `steps` section (`flow > from > steps`) allow the user to specify what to do with each Camel exchange generated by the consumer.
In this case, we set the body (`set-body`) of the exchange to the `constant` value `"Hello world!"`.

If all this does not seem new to you, it's because the YAML DSL is not completely new, but it wraps the structure of the Camel route, 
that now can be written in multiple DSLs. Look at the [Camel K Languages Documentation](https://camel.apache.org/camel-k/latest/languages/languages.html).

The `sink` then decides the destination of each message (the `camel-test` channel), after all the steps are executed (in this case only the `set-body` one).

The hello world can be executed with a simple command:

```
kubectl apply -f camel-timer-source.yaml
```

Note that only `kubectl` (or `oc` if you're using OpenShift) is needed to run the source. On OpenShift 4, you can also 
paste the YAML directly on the OpenShift console to run it.

[img paste openshift console]


To see what the source is generating, you can check the logs of the pod that we've subscribed to the `camel-test` Channel:

```
oc logs --todo
``` 

If everything is installed correctly, you should see something like:

```
# logs TODO
```

Now, let's create some more interesting sources.

## CamelSource #2: "Chuck Source"

This is one of my favourite source as I've used a similar concept in many demos this year.
A Chuck `CamelSource` is a source that periodically generates some random quote about [Chuck Norris](https://en.wikipedia.org/wiki/Chuck_Norris).

Apart from the fact that everybody love Chuck Norris, technically, the idea behind this source is to show how it's possible to 
periodically poll data from external system and transform it to make it available as Cloudevents inside the serverless environment.

```
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: CamelSource
metadata:
  name: camel-chuck-source
spec:
  source:
    integration:
      dependencies:
        - camel:jackson
    flow:
      from:
        uri: timer:tick
        parameters:
          period: 10s
        steps:
          - to: http://api.icndb.com/jokes/random?limitTo=[nerdy]
          - unmarshal:
              json: {}
          - transform:
              simple: "${body[value][joke]}"
          - set-header:
              name: ce-type
              constant: chuck.norris
          - set-header:
              name: Content-Type
              constant: text/plain
  sink:
    ref:
      apiVersion: messaging.knative.dev/v1alpha1
      kind: InMemoryChannel
      name: camel-test
```

This seems a bit more complex than previous example, but it's really easy to understand.

First of all, beside `source` a new object named `integration` is present. The `integration` object allows
to customize the underlying Camel K integration resource that is being created for cases where the sole YAML route is not sufficient.
In this case, we've used it to add a dependency to `camel:jackson`, because the route needs to process some JSON data to do transformations.

Additional dependencies are not limited to Camel components like `camel:jackson`. If you use the `mvn:` schema, you can also add your own library
(e.g. `mvn:com.company:artifact:1.0.0`).
The integration object can contain all fields recognized by a Camel K integration ([Json spec here](https://github.com/apache/camel-k/blob/master/assets/json-schema/Integration.json)).

The Chuck source starts from a timer like the previous one, but then the first step retrieves something from URL `http://api.icndb.com/jokes/random?limitTo=[nerdy]`.
The step will use the [Camel 3 HTTP Component](https://camel.apache.org/components/latest/http-component.html) (you no longer need to use the `http4:` scheme as in Camel 2.x).

Data generated from `icndb.com` looks like the following:

```
{
  "type": "success",
  "value": {
    "id": 530,
    "joke": "Chuck Norris can dereference NULL.",
    "categories": [
      "nerdy"
    ]
  }
}
```

Instead of sending the full JSON data to the recipient sink, we can send only the joke contained in it (`"Chuck Norris can dereference NULL."`!).
To do so, we first unmarshal the JSON data, so it becomes navigable (`unmarshal > json`), then we 
navigate into the body using the [Camel Simple Language](https://camel.apache.org/manual/latest/simple-language.html) 
to get the field `joke` contained in the field `value` of the `${body}` (i.e. `${body[value][joke]}`).

The last things we do is to set the Cloudevents header `CE-Type` to the value `chuck.norris` and the `Content-Type` to `text/plain`.

After creating the source with:

```
oc apply -f todo
```

You can now see Chuck Norris quotes in the receiver service:

```
oc logs TODO
```

Sample result:
```
TODO
```

## CamelSource #3: "MQTT Bridge"

Let's say you have some data that a lot of sensors are sending to a MQTT broker and you want to make this data available to the serverless
services. You need something that transfers messages from the MQTT broker to a Knative channel. That is exactly what the following Camel Source
is doing:

```
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: CamelSource
metadata:
  name: camel-mqtt-source
spec:
  source:
    integration:
      # Optionally, to increase throughput
      replicas: 2
    flow:
      from:
        uri: paho:mytopic
        parameters:
          brokerUrl: tcp://mosquitto:1883
          clientId: mqtt-knative-bridge
        steps:
        - log:
            message: "Forwarding: ${body}"
  sink:
    ref:
      apiVersion: messaging.knative.dev/v1alpha1
      kind: InMemoryChannel
      name: camel-test
``` 

We're using the [Camel Paho](https://camel.apache.org/components/latest/paho-component.html) to read from the `mytopic` topic of the MQTT source.
The broker is available at the address `"tcp://mosquitto:1883"`.

We've also set the client id to `"mqtt-knative-bridge"` in order to scale out correctly. To increase throughput, in fact, we've specified a number of
two concurrent pods (`replicas: 2`) reading from the source, and they must have the same client id if you want to avoid duplicate messages.

The only step we've added to the route (you can also omit it) is to log whatever we're sending to the Knative channel, before sending it.

This is how it looks like:


NOTE: you just need to use a different source component to have a similar source for **Kafka**, any **AMQP** source, **ActiveMQ** or any other messaging 
platform. If your messaging platform is not supported by Camel, you're welcome to contribute a new Camel component and you'll be able to use it in Knative sources.  

## CamelSource #4: "Social Source"

Let's be more social and create a source that can forward messages from a Social Network or a public Chat of our choice.
We're going to use Telegram for this example, in particular a Telegram Bot.

You need to create a Bot using the Telegram Botfather. Once you've done it, you'll get an authorization token that will allow you to 
impersonate the Bot.

You should now put the authorization token into a Kubernetes secret with the following command:

```
oc create secret generic telegram-token --from-literal application.properties=camel.component.telegram.authorization-token=put-here-the-auth-token
``` 

The command above creates the `telegram-token` secret containing the Telegram authorization token.
The `camel.component.telegram.authorization-token` property is documented in the [Camel Telegram component manual page](https://camel.apache.org/components/latest/telegram-component.html#_spring_boot_auto_configuration).

You can now create the Telegram source and reference the token secret:

```
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: CamelSource
metadata:
  name: camel-telegram-source
spec:
  source:
    integration:
      configuration:
      - type: secret
        value: telegram-token
    flow:
      from:
        uri: telegram:bots
        steps:
          - set-header:
              name: ce-author
              simple: "${body.from.firstName} ${body.from.lastName}"
          - set-header:
              name: ce-chat
              simple: "${body.chat.id}"
          - set-header:
              name: Content-Type
              constant: text/plain
          - transform:
              simple: "${body.text}"
  sink:
    ref:
      apiVersion: messaging.knative.dev/v1alpha1
      kind: InMemoryChannel
      name: camel-test
```

The configuration part links the Camel Source to the secret containing the token for Telegram.
The starting point of the source is the `telegram:bots` endpoint that belongs to the [Camel Telegram Component](https://camel.apache.org/components/latest/telegram-component.html).
The authorization token is preconfigured in the secret, so the endpoint does not need additional configuration.

We're setting in the `steps` some useful information, for example we're setting the Cloudevents `CE-Author` extension header (i.e. custom header)
to contain the author of the message (first name + last name).
We're also setting the *"chat id"* that has originated the message as `CE-Chat` Cloudevents header and the `Content-Type` to `text/plain`.
We also transform all messages to text using the Camel's built-in type converters (we're only interested in text for this example, but the source can also be complicated to deal with media).

As you see, this source will forward messages to my serverless environment automarically:


NOTE: you can also use **Slack**, **Twitter**, **Facebook** or any other cloud social service for this type of sources. Even the old good **IRC** is supported!


 
## CamelSource #5: "File Source"

