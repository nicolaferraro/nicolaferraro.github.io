---
title:  "Hot Reconfiguration of Microservices on Kubernetes"
modified: 2016-10-23 11:00:00 +0200
last_modified_at: 2016-10-23 11:00:00 +0200
tags: [Spring Boot, Fabric8, Kubernetes, Apache Camel, Fuse]
categories: [Dev]
header:
    image: post-logo-spring-boot.png
    teaser: post-logo-spring-boot.png
---
Spring Cloud Kubernetes is a fantastic project from the Fabric8 team that contains a lot of useful tools for building spring-boot based microservices. 
Version _0.1.3_ includes a new feature that allows changing the configuration of a microservice at runtime using Kubernetes config maps.

Spring-boot applications are much appreciated for their simple configuration mechanism, based on properties or yaml files. 
There is plenty of ways to override the default settings of an application, usually provided in the bundled "application.properties" file. 
Environment variables take precedence over the provided configuration and this is particularly useful to automatically configure hosts and ports of external services inside Kubernetes.
Other ways, like command line options or external configuration files, are supported [out of the box](http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html).

By just including Spring Cloud Kubernetes into the microservice, the application configuration can be also provided using **Kubernetes config maps**. 
What's new is that the application can be configured to **listen for changes in the Kubernetes namespace** and apply the new configuration at runtime as soon as the config map is updated.
Reconfiguration is applied immediately and only the configurable beans are reloaded to reflect the new settings. 
Respect to other solutions, hot reload **does not require downtime** because the application is not restarted (nor the application context).

I'll show the usage of this new feature with a quickstart. Let's create a _Camel_ microservice (the feature works also without _Camel_) with a simple route:
 
```java
@Component
public class Route extends RouteBuilder {

    @Autowired
    private RouteConfig config;

    @Override
    public void configure() throws Exception {

        from("timer://tick?period=3000")
                .id("generation")
                .transform().simple("message-${header.CamelTimerCounter}")
                .recipientList().method(config, "getRecipients");


        from("direct:async-queue")
                .id("async-queue")
                .log("${body} pushed to the async queue");

        from("direct:mail")
                .id("mail")
                .log("${body} sent via mail");

        from("direct:file")
                .id("file")
                .log("${body} written to a file");


    }
}
```

I'll show here only the relevant parts, but you can just clone or fork my [spring-boot-configmap-quickstart](https://github.com/nicolaferraro/spring-boot-configmap-quickstart) repository and run it.

```bash
git clone git@github.com:nicolaferraro/spring-boot-configmap-quickstart.git
```

The route generates every _3_ seconds a dummy message and pushes it to a list of recipents (this EIP is indeed called "recipent list").
Here there are 3 available endpoints where the message can be pushed to: `direct:async-queue`, `direct:mail` and `direct:file`.
They are fake endpoints for the purposes of this example, as they just log every message with a different surrounding text.

The list of recipents is chosen using the `getRecipients` method of an external bean. Here's the bean:
 
```java 
@Configuration
@ConfigurationProperties(prefix = "route")
public class RouteConfig {

    /**
     * The recipient endpoint.
     */
    private String recipients;

    public RouteConfig() {
    }

    public String getRecipients() {
        return recipients;
    }

    public void setRecipients(String recipients) {
        this.recipients = recipients;
    }
}
```

The `RouteConfig` bean is a typical spring-boot configuration bean with a property of type `String` named `recipients`
that contains the list of target endpoints for every message. 

I've included the configuration in a bean **annotated with _@ConfigurationProperties_** because all the beans with such annotation **will be refreshed automatically**.

The list of recipients is given as **comma separated value string**, as shown in the `application.properties` file below.

```properties
# Application data
spring.application.name=configmap-example

# Enable auto-reload
spring.cloud.kubernetes.reload.enabled=true


# Using an internal port for health checks. 
# Always a good choice to avoid exposing sensitive paths.
management.port=8000

# Camel recipients default configuration.
# Can be overridden using configmaps.
route.recipients=direct:async-queue,direct:mail
```

The `spring.application.name` property serves to give a name to the application: the name is `configmap-example`.
The reason why this is important is because the application is automatically configured to override the default configuration if a Kubernetes config map named `configmap-example` is present. 
A simple flag named `spring.cloud.kubernetes.reload.enabled` **enables the auto-reload feature on configmap change**.
With this property enabled, the config map can be added at a later time and the application will refresh its status.

The `route.recipients` property is the one we want to override with a configmap. The default value includes two target endpoints for the messages: `direct:async-queue` and `direct:mail`.
 
## Running the example in Kubernetes

You need a running Kubernetes or Openshift cluster to run the example. If you don't already have one, I suggest you using [Minikube](https://github.com/kubernetes/minikube/releases) 
or [Minishift](https://github.com/jimmidyson/minishift/releases), to startup a development Kubernetes or Openshift cluster in 2 minutes.

### Note for Openshift:
If you're using Openshift (any kind of cluster, also Minishift), you need to give permissions to the running application to listen for changes in the project.
This can be obtained by **giving the view permission** to the `default` service account. It is the service account responsible to run all the pods by default.
Just login, switch to the project where you want to deploy the application and type:

```bash
oc policy add-role-to-user view --serviceaccount=default
```

The command above applies to the service account named `default` in the current project only.

Now just execute the following Maven goal to run the quickstart:

```bash
mvn clean install fabric8:run
```

The `fabric8:run` goal will deploy the quickstart to your local Kubernetes/Openshift instance and attach the console to the logs of the created pod.
The output should be something like this (after the initialization):

```
[           main] m.nicolaferraro.quickstarts.Application  : Started Application in 5.902 seconds (JVM running for 9.15)
[ - timer://tick] async-queue                              : message-1 pushed to the async queue
[ - timer://tick] mail                                     : message-1 sent via mail
[ - timer://tick] async-queue                              : message-2 pushed to the async queue
[ - timer://tick] mail                                     : message-2 sent via mail
[ - timer://tick] async-queue                              : message-3 pushed to the async queue
[ - timer://tick] mail                                     : message-3 sent via mail
```

Leave the application running. We are going to change the routes dynamically.

## Changing the configuration

Currently an application named `configmap-example` is running in our Kubernetes and it is listening for config maps with the same name in the current namespace.
Let's create a config map with that name to see what happens.

Create a file named `configmap.yml` with the following content:

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  # Must match the 'spring.application.name' property of the application
  name: configmap-example
data:
  application.properties: |
    # Override the configuration properties here
    route.recipients=direct:async-queue,direct:file,direct:mail

```

As you see, we are going to change the `route.recipients` property to add the `direct:file` recipient. The config map is named after the spring application.
To create it, **open a new terminal** and type the following command if you're using Kubernetes:

```bash
kubectl create -f configmap.yml

# For Openshift
# oc create -f configmap.yml
```

The config map should be created without problems. Now look at the program running in the other terminal. You have triggered a configuration change.

```
[ - timer://tick] async-queue                              : message-22 pushed to the async queue
[ - timer://tick] mail                                     : message-22 sent via mail
[ - timer://tick] async-queue                              : message-23 pushed to the async queue
[ - timer://tick] mail                                     : message-23 sent via mail
[default.svc/...] .r.EventBasedConfigurationChangeDetector : Detected change in config maps
[default.svc/...] .r.EventBasedConfigurationChangeDetector : Reloading using strategy: REFRESH
[default.svc/...] s.c.a.AnnotationConfigApplicationContext : Refreshing org.springframework.context.annotation.AnnotationConfigApplicationContext@71cadf38: startup date [Sun Oct 23 08:44:57 UTC 2016]; root of context hierarchy
[default.svc/...] trationDelegate$BeanPostProcessorChecker : Bean 'configurationPropertiesRebinderAutoConfiguration' of type [class org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration$$EnhancerBySpringCGLIB$$bf2575c9] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
[default.svc/...] b.c.PropertySourceBootstrapConfiguration : Located property source: ConfigMapPropertySource [name='configmap.configmap-example.bbb']
[default.svc/...] b.c.PropertySourceBootstrapConfiguration : Located property source: SecretsPropertySource [name='secrets.configmap-example.bbb']
[default.svc/...] o.s.boot.SpringApplication               : The following profiles are active: kubernetes
[default.svc/...] s.c.a.AnnotationConfigApplicationContext : Refreshing org.springframework.context.annotation.AnnotationConfigApplicationContext@18d995cf: startup date [Sun Oct 23 08:44:57 UTC 2016]; parent: org.springframework.context.annotation.AnnotationConfigApplicationContext@71cadf38
[default.svc/...] o.s.boot.SpringApplication               : Started application in 0.469 seconds (JVM running for 92.617)
[default.svc/...] s.c.a.AnnotationConfigApplicationContext : Closing org.springframework.context.annotation.AnnotationConfigApplicationContext@18d995cf: startup date [Sun Oct 23 08:44:57 UTC 2016]; parent: org.springframework.context.annotation.AnnotationConfigApplicationContext@71cadf38
[ - timer://tick] async-queue                              : message-24 pushed to the async queue
[ - timer://tick] file                                     : message-24 written to a file
[ - timer://tick] mail                                     : message-24 sent via mail
[ - timer://tick] async-queue                              : message-25 pushed to the async queue
[ - timer://tick] file                                     : message-25 written to a file
[ - timer://tick] mail                                     : message-25 sent via mail

```

Now each message is also pushed to the `direct:file` endpoint, as required by the config map. And the change happens **on the fly** as soon as you create the config map and without any downtime.
 
You can also change again the config map from the Openshift console or the command line:

```
kubectl edit configmap configmap-example

# For Openshift
# oc edit configmap configmap-example
```

A **vi-like editing screen** will appear and let you change the contents.

## In a real scenario
 
A common use case for this feature is creating a *per-environment* configuration, allowing you to use eg. different settings in the `testing`, `staging` and `production` environments.
Plus, the feature allows you to change the configuration at runtime.

It's unlikely that you want to change the configuration of an application using `vi` in production (altough I've done that multiple times in the past XD).

Configuration management need a special care. For example, one might want to:

- Track all the changes done to the configuration to be able to revert them;
- Track the users that changed the configuration;
- Authorize only certain users to change the configuration;
- Peer review all the changes to avoid mistakes.

You can obtain this level of control by including the configmap in a **git repository**. There can be a single repository for all the configmaps, or one per configmap, you decide.

For the current example, I've provided an example of git repository with a config map ready to be deployed by the *Fabric8 Maven Plugin*.
Instead of creating the config map file by yourself, you can just have cloned my repo and installed it:

```bash
git clone git@github.com:nicolaferraro/spring-boot-configmap-settings.git
cd spring-boot-configmap-settings
mvn clean install fabric8:deploy
```

Of course, this is not what you're going to do in production. You should instruct Jenkins to deploy the config map to your Kubernetes as soon as it changes.
There are multiple ways to do so. [Fabric8](https://fabric8.io/gitbook/) has a fully featured Jenkins pipeline system that can be configured to accomplish this task with 2 clicks.
Openshift Enterprise 3.3 is also able to use [Jenkins pipelines](https://blog.openshift.com/whats-new-openshift-3-3-developer-experience/).  

## Settings and limitations

The reload feature can also be configured to completely *restart* the application context for the cases where a *refresh* is not enough. 
It can also shutdown completely the pod to let the replication controller restart a new one (for the extreme cases).
Check the [documentation](https://github.com/fabric8io/spring-cloud-kubernetes#propertysource-reload) for the other available options.

When using the *refresh* mode, you should ensure no properties are deleted between two subsequent versions of a config map. Properties can be added, changed but not removed.
[This issue](https://github.com/spring-cloud/spring-cloud-config/issues/421) is related to the *refresh* mechanism of Spring Cloud and prevents you from using lists in configuration (elements cannot be removed from a list).
If you want to use lists or to be able to remove properties, you need to change the reload level in the `application.properties` file:

```
spring.cloud.kubernetes.reload.strategy=restart_context
```

This option causes a short downtime because the application context is restarted whenever the config map changes, but it's still better than restarting the JVM.

The feature can be also used with **Kubernetes secrets**. Take a look at [spring-cloud-kubernetes on Github](https://github.com/fabric8io/spring-cloud-kubernetes).
