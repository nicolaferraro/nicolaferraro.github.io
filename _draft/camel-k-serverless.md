Apache Camel K is now progressing towards the 1.0 release and consolidating its serverless superpowers by providing a clean integration with Knative.
Camel K integrations can scale automatically to zero and also interact with Knative eventing components for creating event-based applications. But how are you going to use Camel K in this kind of architectures? And why is Camel K a good choice for integrating external systems in the serverless world? In this post I'm going to provide some answers, starting from the main aspects that made Apache Camel great during all these years: enterprise integration patterns.

Apache Camel was created with enterprise integration patterns in mind and so also Camel K is able to leverage them. If you want to learn more about Camel K, I suggest you to read the previous articles LINK that explain the basics.

For the rest of the article, I'll often talk generically about functions, without giving them specific traits. The term "function" is often confusing because it has different meanings in different contexts. But it's important to remember that a "serverless function" is NOT the same as a "function in math or functional programming". In particular, a serverless function don't necessarily need to be "pure": it can have side effects. More importantly, it can have some form of state, persisted in local storage or elsewhere in the cloud. A "pure serverless function" has little application in a serverless architecture.

I'd talk about "services" and not "functions", because this concept of function resembles more a "smaller microservice" than the mathematical concept of function, but I'm sure most people refer to them as functions and I'll stick with that definition. Fortunately, the Knative building block for deploying a function is called "Service".

Let's start with some patterns!

## API Gateway

The API gateway is one of the most important patterns for today's workloads, because it allows to give a REST shape to a series of functions, so that they can be consumed by client or web applications.

PICTURE

The SYMBOL in the picture 



Reminder: use advanced routing to let functions decide processing paths.