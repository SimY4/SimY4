+++
title = "We Began to Forget: Demystifying Spring Framework"
date = "2026-01-15"
tags = ["java", "we-began-to-forget"]
categories = ["dev-blog"]
+++

We began to forget how to pass parameters to functions.

Before `@Bean`, before `@Configuration` or `@PropertySource` and certainly way before `WebSecurityConfigurerAdapter`
objects were responsible not only for their own behaviour, but also for creating everything they depended on.

```java
public class OrderService {
    private final PaymentProcessor processor;
    public OrderService() {
        this.processor = new StripePaymentProcessor();
    }
}
```

This code works, but it also quietly locks many decisions inside the class, because `OrderService` now decides which 
payment processor is used, how it is created, and when its lifecycle starts, and once this happens, changing that
decision means changing the code itself. Configuration and behaviour start to mix, breaking single responsibility 
principle.

This is where the idea of inversion of control slowly appears.

## Inversion of Control

Inversion of control is simply the idea that a class should not decide how its dependencies are created, and instead
it should only describe what it needs in order to work.

```java
public class OrderService {
    private final PaymentProcessor processor;
    public OrderService(PaymentProcessor processor) {
        this.processor = processor;
    }
}
```

The simplest Spring context you can build today looks like this:

```java
var context = new GenericApplicationContext();
context.registerBean(PaymentProcessor.class, StripePaymentProcessor::new);
context.registerBean(OrderService.class, 
        () -> new OrderService(context.getBean(PaymentProcessor.class)));
context.refresh();

var service = context.getBean(OrderService.class);
```

This is the moment where a container enters the picture, and dependency injection becomes the practical way to apply
inversion of control.

{{<figure src="/images/posts/its-time-for-some-awesome-stories.png" alt="Rule of Cool" width="400">}}

## Declarative Era: When XML Took the Stage

{{<details "Spring don't even..." >}}
Spring did not invent the idea of describing components in XML, because enterprise Java was already full of it.

Before Spring, J2EE and EJB used XML heavily, but those XML files were not only describing wiring, they were enforcing
a very strict container model.

```xml
<ejb-jar>
    <enterprise-beans>
        <session>
            <ejb-name>OrderService</ejb-name>
            <ejb-class>com.example.OrderServiceBean</ejb-class>
            <session-type>Stateless</session-type>
        </session>
    </enterprise-beans>
</ejb-jar>
```

EJB at the time was forcing you into a worldview of application containers: JBoss, WebSphere, GlassFish, and others.
At that time there was a joke, which says that Java is a language that converts XML files into stack traces.

Spring offered you a library. It was about decoupling enterprise concerns from enterprise infrastructure. A breath of
fresh air, just plain objects wired by a container you controlled.
{{</details>}}

In the example above you explicitly declare:

- what beans exist
- how they are constructed
- how they depend on each other

Spring is a boundary between *what your code does* and *how your code is assembled*. And yes, it feels verbose.
Verbosity invited abstraction and abstraction invited XML. Spring’s story began when object graphs were declared,
not coded:

```xml
<beans>
    <bean id="paymentProcessor" class="com.example.StripePaymentProcessor"/>
    <bean id="orderService" class="com.example.OrderService">
        <constructor-arg ref="paymentProcessor"/>
    </bean>
</beans>
...
var context = new ClassPathXmlApplicationContext("beans.xml");
var service = context.getBean(OrderService.class);
```

Object wiring became declarative. But at the same time XML grew dense, IDE support lagged behind, refactoring was
fragile. But it worked and it was extremely popular.

{{<details "XML was not the only option available" >}}
There were other attempts at declarative DSLs. Still supported but rarely mentioned: Groovy based bean definitions

```groovy
beans {
    paymentProcessor(StripePaymentProcessor)
    orderService(OrderService, paymentProcessor)
}
...
var context = new GenericGroovyApplicationContext("beans.groovy");
```

works only if you have Groovy on your classpath. It never became mainstream.
{{</details>}}

## Context of Contexts

Not only beans form a graph, but contexts themselves can also form a graph. A Spring `ApplicationContext` is not always
alone, because it can have a parent, and together they form a hierarchy where visibility flows in one direction.

```java
var parent = new ClassPathXmlApplicationContext("parent-context.xml");

var child = new ClassPathXmlApplicationContext(
        new String[]{"child-context.xml"}, parent);
```

A child context can see all beans defined in its parent, and it can use them as dependencies as if they were local, but
the parent context cannot see anything defined in the child. This means that the dependency graph can be layered, and
resolution will always walk upward.

Parent and child contexts don’t have to share the same lifecycle. A child context can be shorter-lived, can be created
later, attached to an existing parent, used for some period of time, and then closed, without affecting the parent
context at all. When a child context is destroyed, only the beans defined in that child are destroyed. Beans in the
parent remain untouched and continue to exist.

This means that Spring allows part of the dependency graph to appear and disappear, while the rest stays stable. This
model allowed Spring applications to share infrastructure without forcing everything into one large flat configuration,
and it also made it possible to separate concerns cleanly between layers. But in modern era, when annotations took over,
layered contexts are rarely present.

## The Modern Era: Configuration as Code

Over time, XML started to feel heavy, and Spring slowly moved configuration back into Java code, using `@Configuration`
and `@Bean` methods.

```java
@Configuration
public class AppConfig {
    @Bean
    public PaymentProcessor paymentProcessor() {
        return new StripePaymentProcessor();
    }
    @Bean
    public OrderService orderService(PaymentProcessor processor) {
        return new OrderService(processor);
    }
}
...
var context = new AnnotationConfigApplicationContext(AppConfig.class);
```

This was more pleasant to write, easier to refactor, and felt closer to the language developers already knew, but it
also changed something important under the surface. With XML, Spring could see the entire dependency graph before
creating any objects, because everything was described as data. With Java methods, the graph became executable, and
Spring could no longer fully know what a bean was without touching the code.

```java
@Bean
public PaymentProcessor paymentProcessor() {
    return new StripePaymentProcessor();
}
```

The declared return type is `PaymentProcessor`. The actual runtime type is `StripePaymentProcessor`. Because Java
allows covariant return types, the declared return type of a `@Bean` method is not always the real type that will be
created, and sometimes the only way for Spring to know is to actually create the object.

Spring cannot answer simple questions like:

- “What beans of type X exist?”
- “Which implementations satisfy this injection point?”
- “Is this dependency ambiguous?”

unless the method is invoked, side-effects triggered and object instantiation occurred. Spring worked around this with
🌈 more engineering 🌈 : `@Configuration` classes are now enhanced with CGLIB, all methods are intercepted to avoid
double instantiation, optionally beans can also be CGLIB enhanced to provide laziness. But there’s an escape hatch for
all this overhead, if you don’t lie to spring and you can guarantee that your beans are resolvable purely based on
return type information available, you can eliminate the unwanted overhead with
`@Configuration(proxyBeanMethods = false)`.

## The Boot Era: When *it Got Real

Spring Boot arrived and changed the feeling of Spring more than any other step before it. With Boot, beans stopped being
mostly declared and started being discovered, because classpath scanning, starters, and auto-configuration began to
assemble the context implicitly.

Adding a dependency was no longer just adding code, it was also adding behaviour, because auto-configurations could
activate based on what classes were present, what properties were set, and what beans already existed. Conditional
configuration, ordering rules, and missing-bean checks made the container very flexible, but also much harder to fully
see. Understanding what your application context actually contains can now require reading auto-configuration classes
and mentally simulating condition evaluation.

The level of reflection involved skyrocketed to extremes never seen before. My favourite example of this level of
insanity in Spring is the `@ConditionalOnClass` annotation.

```java
@ConditionalOnClass(DataSource.class)
public class DataSourceAutoConfiguration {
    ...
}
```

A natural question appears here, especially for people who think carefully about how Java works, which is how Spring
can evaluate `@ConditionalOnClass` when the class is not even present on the classpath? Normally, referring to a missing
class in Java would cause a `ClassNotFoundException`, or even fail at class loading time, but Spring must avoid this,
otherwise auto-configuration would break exactly in the cases where it is supposed to stay silent.

The trick is that Spring does not load the class in order to check its presence. Instead, Spring reads class metadata
without actually loading the class into the JVM, using bytecode inspection. When Spring processes auto-configuration
classes, it scans annotations using a metadata reader, which works directly on .class files as resources, and it checks
whether a type name exists on the classpath without resolving it. If the class file for `DataSource` is found, the
condition passes. If it is not found, the configuration is simply skipped, without an exception and without noise.
This means that Spring can safely ask the question “does this class exist?” without ever touching the class itself.

## Fin

From manual contexts, to XML, to annotations, to auto-configuration, the container became quieter, robust and more
confident, and we accepted that because it usually made our work faster.

But when something feels strange, or when a bean appears without being invited, or when order suddenly matters, it
helps to remember that Spring is still doing the same old thing. It is still building a dependency graph. 

It just stopped showing it to us unless we really insist on looking.
