Apache Camel K is now progressing towards the 1.0 release and consolidating its serverless superpowers by providing a clean integration with Knative.
Camel K integrations can scale automatically to zero and also interact with Knative eventing components for creating event-based applications. But how are you going to use Camel K in this kind of architectures? And why is Camel K a good choice for integrating external systems in the serverless world? In this post I'm going to provide some answers, starting from the main aspects that made Apache Camel great during all these years: enterprise integration patterns.

Apache Camel was created with enterprise integration patterns in mind and so also Camel K is able to leverage them. If you want to learn more about Camel K, I suggest you to read the previous articles LINK that explain the basics.

For the rest of the article, I'll often talk generically about functions, without giving them specific traits. The term "function" is often confusing because it has different meanings in different contexts. But it's important to remember that a "serverless function" is NOT the same as a "function in math or functional programming". In particular, a serverless function don't necessarily need to be "pure": it can have side effects. More importantly, it can have some form of state, persisted in local storage or elsewhere in the cloud. A "pure serverless function" has little application in a serverless architecture.

I'd talk about "services" and not "functions", because this concept of function resembles more a "smaller microservice" than the mathematical concept of function, but I'm sure most people refer to them as functions and I'll stick with that definition. Fortunately, the Knative building block for deploying a function is called "Service".

Let's start with some patterns!

## API Gateway

The API gateway is one of the most important patterns for today's workloads, because it allows to give a REST shape to a series of functions, so that they can be consumed by client or web applications.

PICTURE

The SYMBOL in the picture depicts services that can automatically scale thanks to Knative technology. As you see, the Camel K API gateway is able to scale automatically because it's rendered as a Knative serving service.

Given two functions called fun-a and fun-b, the Camel K code to create the API gateway is:

api.groovy
'''
rest().post("/path")
  .to("knative:endpoint/fun-a")
  .to("knative:endpoint/fun-b")
'''

The output of function fun-a is given as input of fun-b and the result of fun-b is returned to the REST API caller. Of course Camel is able to do any kind of transformation between those steps.

The script can be simply deployed on Kubernetes with:
'''
kamel run api.groovy
'''

Just make sure you've previously install the camel k operator in the namespace DOC.

Let's see now some other Knative building blocks in action.

## Importer

The importer pattern has a special role because importers are top level Knative resources (you may have heard about them, they were called Knative Sources in previous releases). An importer is simply a resource that brings events inside the serverless environment.

This is where Apache Camel and Camel K shine, because you can really bring events from any external system.

PICTURE




Reminder: use advanced routing to let functions decide processing paths.