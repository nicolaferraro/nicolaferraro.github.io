---
title:  "Serverless"
modified: 2018-10-26 15:00:00 +0200
last_modified_at: 2018-10-26 15:00:00 +0200
tags: [Serverless]
categories: [Dev]
header:
    image: question-mark.png
    teaser: question-mark.png
---

# [DRAFT] Random notes
# ------ TODO ------

Is "serverless" just an empty buzzword, or there's something behind it?

## The Confusion

If you ask two random (IT) folks what "serverless" means, it's likely that they will give you two completely different responses.
And if they give you the same one, it's probably wrong.

The root cause of the confusion is in the word itself: "serverless" literally means "without servers". So it's easier for
people to associate the term with the absence of infrastructure. I've seen people entering an infinite loop while thinking to this.

At the beginning of the journey, each time someone mentioned the word "serverless", I always
though to the [Hacker News Experiment: "This page exists only if someone is looking at it"](http://ephemeralp2p.durazo.us/2bbbf21959178ef2f935e90fc60e5b6e368d27514fe305ca7dcecc32c0134838).
That was **really "without servers"**: the idea of a web page that is served only by connected browsers in a kind of P2P network is **amazing!**
But that's **not what "serverless" means**, even if it was an interesting experiment.

For **uninitiated**, but also negative thinking people in general, there's no such thing as a "serverless" architecture: **"serverless" is just someone else's computer!**.
It's **"just a new way to spend more money"**.
It reminds me of the current political majority party in Italy, that explains every complex phenomenon as a **conspiracy** created by *[put-a-big-company-here]* to steal money from poor people
(and considering how many votes they got, I think that a lot of people support this interpretation of the "serverless" term).

But the way **"initiated"** people think about "serverless" is also often wrong. I call "initiated" people those devs and IT folks in general
that started to experiment with platforms that allow executing custom code in response to events. Those custom pieces of code are
called **functions** and platforms running them are known as **function-as-a-service platforms (FaaS)**. Now, there's something important to remember: **"FaaS" is not a synonym of "serverless"**.
But the "FaaS" term has started appearing recently, [especially in combination with "BaaS"](https://blog.symphonia.io/revisiting-serverless-architectures-29f0b831303c),
that is one of the other constituents of the "serverless area".

The unification of the terms "FaaS" and "serverless" in the mind of many people has also been accomplished with a series of announcements
since the end of 2016 to the beginning of 2018 of a **lot of Open Source projects claiming that "they were serverless"** (OpenWhisk, Fission, Kubeless, Funktion, ... Fn, Riff).
Actually, it's true that they were "serverless", but all them belong to the sub-category of FaaS platforms, increasing the confusion around the terms.

To complete the **madness** of the whole picture, since all these "pieces of code" that are executed on a FaaS platform are called **"functions"**,
I've seen also people confusing the **"function" in the FaaS context** with the concept of **"function" in functional programming**.
This is dangerous because functional programming has an unconditioned love for "pure functions", i.e. functions ("give me a input, I'll return a output") without
internal state and without side effects (e.g. writing data to a database is a "side effect" in functional programming).

So, some people started to invent new "serverless" architectures where the functions were only "pure functions". They soon realized that
the only use cases for such kind of functions in the "serverless" context were the creation of a **"hello world function"** that always returns "Hello World!" to any input
(and it was actually used a lot in the demos!) and a **hypothetical image recognition function** that given a photo returns you the name of the people contained in it.
It's difficult to find examples of a **useful** single pure function.
Furthermore, given that you pay "per function execution" in the FaaS model, as we'll see, using **function composition is not an option**.

As additional drawback, the "pure function" thing increased the common **(mis)understanding that "serverless" applications are unuseful**.

Let's **try to put some order**...

## What does "serverless" mean, then?

I see "serverless" architectures as a necessary evolution of the "microservice" architectures. And I'm going to explain why.
But, first of all: let's try to define what serverless is.

I like the definition of "serverless" given by [Jeff Hollan](https://hollan.io/): **"Serverless is a billing model"**. It is a good starting point
to understand what it is. It allows to distinguish immediately what "serverless" is from what it's not.

Let's drill it down. At the core of "serverless" there's the concept of **not caring about the infrastructure**, instead focusing on other aspects of the applications you are developing.
So "serverless" as "I don't care about servers" is another interpretation of the term. That does not mean that there aren't servers.
It just means that you don't see them. Especially: **you don't pay for servers in the monthly bills**.

But if you don't pay for servers, **what do you pay for?**

You pay for what you actually **"use"**. Let's take the _example_ of the FaaS platform: you may be **billed** depending on how many times your **functions are invoked**.

**What about BaaS?**

BaaS is a shorthand for "backend as a service". While a FaaS is expected to provide services such as scalable execution of user code,
a BaaS is expected to provide a place where the application state can be stored. And it's expected to be scalable.

Just to give some context, a typical example of BaaS is [Firebase](https://firebase.google.com/), a well known "database-as-a-service" that allows to
develop a wide set of web and mobile applications literally without a "application layer".

Well, this kind of BaaS services apply a similar billing strategy to the one used in FaaS: they let you pay for how many GB you store and/or the
number of concurrent users you need to handle.

This is what "serverless" platform have in common: they **allow billing strategies based on actual usage**.
And this is also the **rule of thumb** to recognize a "serverless" platform.

I want to put the mark on the **"they allow"** statement, related to pay per usage:
- (they enable) It means that operational costs of running the platform will be **proportional** to the metrics used in the billing phase (more on this later).
- (they grant) It means that pay-per-usage is not a required model, but a potential.

## And it's cheap, right?

In many circumstances, this model is expected to have costs that are proportional to earnings of the company using it.
So, especially when there's a **free start tier**, costs are expected to be lower than in traditional models for the users.

But I don't want to go into the details of the billing strategies. That is not my job.

I want just to mention that a strategy that allows you to pay-per-usage can be dangerous.
Just think what happens if you accidentally DDOS yourself (["Serverless: A lesson learned. The hard way."](https://sourcebox.be/blog/2017/08/07/serverless-a-lesson-learned-the-hard-way/)).

A reasonable strategy is to always allow per-usage-billing, but with a cap on the total cost...

## What about "serverless architectures"?

So far I've talked about billing. But the architecture of a "serverless" platform, unfortunately, is tightly linked to the billing model.

For this model to be sustainable, a company that offer a "serverless" product should have management costs that are proportional to the earnings.
So, if the earnings depends on a specific "metric", also operational costs should depend on the same metric.

**A practical example:**
you develop a FaaS platform and your customers pay per function invocation. Now, suppose that a customer deploys his application in your
FaaS platform as a series of functions, but nobody uses that application for an entire month. He obviously pays 0 at the end of the months.
But a "traditional" application, e.g. a web application, needs to be kept alive to serve requests in case they arrive. Te costs of keeping that application alive
are much higher than the earnings (that are 0 in this case). A "traditional" architecture, even if container-based, is not suitable for this billing model.

This is the basic reason why "serverless" platforms usually need different architectures.





