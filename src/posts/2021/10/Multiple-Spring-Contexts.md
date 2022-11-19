---
title: "Multiple Hierarchical Contexts in Spring Boot"
date: "2021-10-07"
tags:
- springframework
- springboot
- java
- maven
---

<!-- excerpt -->
### TL;DR

In addition to creating **only one** [`ApplicationContext`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/ApplicationContext.html) in a typical _Spring Boot_ application, there is a possibility to create **multiple** instances of `ApplicationContext` in a single application. These contexts can form parent-child relationships between them. This way we don't have any longer a single `ApplicationContext` containing all _beans_ defined in the application. We can rather have a hierarchy of contexts each containing _beans_ defined within themselves. Forming the parent-child relationships between contexts, accessibility of _beans_ defined in a context can be controlled.

### Introduction

In a typical _Spring Boot_ application written in _Java_, our main method will look similar like the one below.

```java
@SpringBootApplication
public class MainApplication {

    public static void main(String[] args) {
        SpringApplication.run(MainApplication.class, args);
    }

}
```

Static method `run` from [`SpringApplication`](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/SpringApplication.html) class will create a single `ApplicationContext`. The context will contain all custom _beans_ defined in the application, as well as _beans_ which will be created during _Spring_'s auto-configuration process. This means that we can request any _bean_ from the context to be auto-wired, with help of _Spring_'s dependency injection mechanism, wherever in our application. In most cases this will be preferred behavior.

However, there could be situations where we don't want that all defined _beans_ are accessible within the whole application. We might like to restrict access to some specific _beans_ because a part of the application shouldn't know about them. The real-world example could be that we have a _multi-modular_ project, in which we don't want that _beans_ defined in one module know about _beans_ defined in other modules. Another example could be that in a typical _3-layer architecture_ (controller-service-repository) we would like to restrict access to controller _beans_ from repository or service related _beans_.

_Spring Boot_ enables creating multiple contexts with its [Fluent Builder API](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#features.spring-application.fluent-builder-api), with the core class [`SpringApplicationBuilder`](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/builder/SpringApplicationBuilder.html) in this API.

### Single context application

We will start with a simple _Spring Boot_ application having only one context. The application will be written in _Java_ and will use _Maven_ as building tool.

The application will contain only core _Spring_ and _Spring Boot_ dependencies with a help of _spring-boot-starter_. Resulting `pom.xml` file should look like the one on the [link](https://github.com/nikolamicic21/spring-boot-multiple-hierarchical-contexts/blob/main/pom.xml).

There are two different packages in the application

- **web** package
- **data** package

both containing relevant _beans_ (`WebService` and `DataService`).

Source code of the single context application where `WebService` has dependency on `DataService` can be found [here](https://github.com/nikolamicic21/spring-boot-multiple-hierarchical-contexts/tree/single-context-webService-dependsOn-dataService).

If we look into the _main_ method and retrieve the context from the `SpringApplication`'s _run_ method we can query for _beans_ in the context like in the code snippet below.

```java
@SpringBootApplication
public class MainApplication {

    public static void main(String[] args) {
        final var context = SpringApplication.run(MainApplication.class, args);
        final var webService = context.getBean(WebService.class);
        final var dataService = context.getBean(DataService.class);
    }

}
```

_Spring_ will resolve both _beans_ and return singleton instances (by default) for both classes (`WebService` and `DataService`). This means that both _beans_ are accessible from the context via _getBean_ method and _Spring_ can auto-wire an instance of `DataService` into the instance of `WebService` via dependency injection mechanism.

We could achieve reversed scenario where `DataService` has dependency on `WebService`. _Spring_ will be able to auto-wire _bean_ of type `WebService` into the `DataService` _bean_.

The source code with reversed dependency direction can be found [here](https://github.com/nikolamicic21/spring-boot-multiple-hierarchical-contexts/tree/single-context-dataService-dependsOn-webService).

All this is possible because all _beans_ are part of **the same context**.

### Multiple hierarchical contexts

With a help of _SpringApplicationBuilder_ we can create multiple contexts in our application and form parent-child relationships between them. This way we can restrict access to the `WebService` _bean_ in the way that it cannot be injected into `DataService`.

The rule for multiple context hierarchy with parent-child relationship is that **_beans_ from parent context are accessible in child contexts**, but not the other way around. In our case, a context which defines `WebService` will be a child context of the context which defines `DataService`. Another rule is that a context can have **only one parent context**.

This could be achieved in a couple of steps

- Step One would be to create context for web related _beans_ in a new class, annotated with [`@Configuration`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/Configuration.html)
  , in the web package. This context will scan all _beans_ defined within the web package with a help of [`@ComponentScan`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/ComponentScan.html). Annotation [`@EnableAutoConfiguration`](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/autoconfigure/EnableAutoConfiguration.html) will take care of triggering _Spring_'s auto-configuration process.

```java
@ComponentScan
@EnableAutoConfiguration
@Configuration(proxyBeanMethods = false)
public class WebContextConfiguration {
}
```

- Step Two would be to create a context for data related _beans_ in a new class, annotated with `@Configuration` in the data package. _Beans_ defined within the data package will be discovered the same way as the one defined in the web package.

```java
@ComponentScan
@EnableAutoConfiguration
@Configuration(proxyBeanMethods = false)
public class DataContextConfiguration {
}
```

- Step Three is to use only [`@SpringBootConfiguration`](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/SpringBootConfiguration.html) annotation on our main class instead of a well-known [`@SpringBootApplication`](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/autoconfigure/SpringBootApplication.html) annotation in order to prevent component scanning and auto-configuration. As we saw before, both processes (component scanning and auto-configuration) will be done in each child context separately.

- Step Four is to use `SpringApplicationBuilder` class instead of `SpringApplication` class as in the code snippet below. Method _web_ in `SpringApplicationBuilder` specifies which type of web application the specific context will define (possible values are NONE, SERVLET, and REACTIVE defined in [`WebApplicationType`](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/WebApplicationType.html)).

```java
@SpringBootConfiguration
public class MainApplication {

    public static void main(String[] args) {
        new SpringApplicationBuilder()
                .sources(MainApplication.class)
                    .web(NONE)
                .child(DataContextConfiguration.class)
                    .web(NONE)
                .child(WebContextConfiguration.class)
                    .web(NONE)
                .run(args);
    }

}
```

Context hierarchy defined above in the `SpringApplicationBuilder` can be represented by the following diagram.

![main-data-web-contexts-hierarchy](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/r297od09hs5um3cwenui.png)

If we now try to autowire `WebService` into `DataService` _bean_ we will get [`NoSuchBeanDefinitionException`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/NoSuchBeanDefinitionException.html) stating that `WebService` _bean_ cannot be found. This comes from the statement that _beans_ defined in child contexts are not accessible from the parent context. The source code with these changes can be found [here](https://github.com/nikolamicic21/spring-boot-multiple-hierarchical-contexts/tree/multiple-contexts-dataService-cannot-access-webService).

There will be no exception thrown in the case when we try to auto-wire `DataService` into `WebService` as _beans_ defined in parent context are visible to _beans_ defined in child contexts. The source code of the application with multiple contexts where `WebService` depends on `DataService` can be found [here](https://github.com/nikolamicic21/spring-boot-multiple-hierarchical-contexts/tree/multiple-contexts-webService-can-access-dataService).

### Parent-child-sibling context hierarchy

Another interesting hierarchy which can be applied is when we have a single parent context with multiple children contexts, where each child context forms a sibling relationship with other child contexts.

The rule for sibling contexts is that **_beans_ in one sibling context cannot access _beans_ defined in other sibling contexts**.

Let us see how our main class looks like for this case.

```java
@SpringBootConfiguration
public class MainApplication {

    public static void main(String[] args) {
        new SpringApplicationBuilder()
                .sources(MainApplication.class)
                    .web(NONE)
                .child(DataContextConfiguration.class)
                    .web(NONE)
                .sibling(EventContextConfiguration.class)
                    .web(NONE)
                .sibling(WebContextConfiguration.class)
                    .web(NONE)
                .run(args);
    }

}
```

As we can see, we have introduced additional context `EventContextConfiguration` in a separate package **event**. The context is defined in a similar way as the other child contexts.

```java
@ComponentScan
@EnableAutoConfiguration
@Configuration(proxyBeanMethods = false)
public class EventContextConfiguration {
}
```
Diagram for this kind of context hierarchy is shown below.

![main-data-web-event-contexts-hierarchyt](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/sz3u1hna93rs1xkdu0g7.png)

As we can see from the diagram, all child contexts share the same parent and form sibling relationship.

If we retain the same dependency hierarchy, where `WebService` depends on `DataService`, we would get `NoSuchBeanDefinitionException` exception, because it is not accessible from the  sibling context. The source code for this stage of the application can be found [here](https://github.com/nikolamicic21/spring-boot-multiple-hierarchical-contexts/tree/multiple-sibling-contexts-webService-can-access-dataService).

Thing to note is that child contexts can still access _beans_ defined in the parent context.

### Summary

In this installment we have seen how we can create a _Spring Boot_ application containing multiple contexts and how they can form parent-child relationships.

We have also mentioned a couple of real-world scenarios where we could structure contexts in hierarchy and how.

Also, we have seen that there are multiple ways to form parent-child relationships including sibling relationships.

### References

- [Spring Boot Reference Documentation - Fluent Builder API](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#features.spring-application.fluent-builder-api)
- [Domain-Driven Design by Examples - Library Project](https://github.com/ddd-by-examples/library)
- [Context Hierarchy with the Spring Boot Fluent Builder API](https://www.baeldung.com/spring-boot-context-hierarchy)
