---
title:  "Strategy Pattern using CDI"
date:   2016-02-24 17:29:32 +0200
tags: [CDI, Design Pattern, Java EE]
---
The strategy pattern is one of the most famous patterns by the GoF and, for sure, one of the most useful in the Java EE world. 
Implementing it using Context and Dependency Injection (CDI) may appear difficult, but it’s just a matter of following some simple steps.

![Strategy Pattern](/images/strategy-pattern.png){:width="300px" .align-center}

The strategy pattern allows a strategy to be selected dynamically among a collection of possible strategies. The client will be completely unaware of the selection and the choice will be done programmatically, on the basis of contextual information.

In order to implement the pattern using CDI, the following set of artifacts should be created:

```java
public interface Algorithm {
 
    String compute(String input);
 
}
```

First of all, the *Algorithm* interface. 
It can be any custom interface and will be referenced by the client to obtain an implementation of the strategy. 
You may need to create a *MyDomainObjectManager*, a *DAO* object, etc.. 
You just have to declare the interface of your bean.

```java
public class Client {
 
  @Inject
  private Algorithm algorithm;
 
  // ...
  // code that uses the algorithm
 
}
```

The client will inject an instance of the *Algorithm* interface and nothing else. 
The choice of which instance will be injected will vary depending on a **producer methods**. More details below.

In order to define multiple instances of an *Algorithm*, a qualifier needs to be defined. 
We will need a custom one. **The *@Named* qualifier cannot be used** in this case as objects having the *@Named* 
qualifier get also the *@Default* qualifier for free and this causes trouble when you try to inject a *@Default* instance. 
A generic qualifier will be sufficient for this purpose:

```java
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({TYPE, CONSTRUCTOR, FIELD, PARAMETER})
@Documented
public @interface Implementation {
}
```

Once the qualifier is defined, two different classes implementing the *Algorithm* interface can be declared:

```java
// First declaration
@Implementation
public class FirstAlgorithm implements Algorithm {
 
    public String compute(String input) {
        return "first";
    }
}
 
// Second declaration
@Implementation
public class SecondAlgorithm implements Algorithm {
 
    public String compute(String input) {
        return "second";
    }
}
```

At this point, these two classes will never be injected into the client code, as they define a qualifier 
that is different from the one expected by the client, that is *@Default* (it is implicitly required when 
you don’t declare other qualifiers at the injection point).

In order to glue the whole puzzle, we need the last piece, the *@Produces* method:

```java
public class AlgorithmProducer {
 
    @Produces
    public Algorithm getInstance(@Any @Implementation Instance<Algorithm> instances, InjectionPoint injectionPoint) {
 
        Map<Class, Algorithm> algorithms = new HashMap<>();
        for(Algorithm a : instances) {
            algorithms.put(a.getClass(), a);
        }
 
        return algorithms.get(SecondAlgorithm.class);
    }
 
}
```

The *AlgorithmProducer* class contains the method that decides which implementation should be used for the 
*Algorithm* (in this case, the *SecondAlgorithm* has been chosen statically).

In the producer method, CDI will inject all the instances of the *Algorithm* interface that have our custom 
*@Implementation* annotation.

Note that the second argument of the *@Produces* method is the *InjectionPoint*. 
It is optional, but it is much powerful as it provides access to contextual information, 
that can be used to choose a different implementation of the strategy depending on the context where it is needed.

If you want to change the implementation of the *Algorithm* at runtime for a particular client, 
the client should not inject the *Algorithm* directly, but a **Instance**:

```java
public class Client {
 
  @Inject
  private Instance<Algorithm> algorithmInstance;
 
  // ...
  // code that uses the algorithm
  //
  // Algorithm algorithm = algorithmInstance.get();
 
}
```

This way, every time the client needs the algorithm, the producer code is re-evaluated.