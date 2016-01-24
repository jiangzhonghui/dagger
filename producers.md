---
layout: default
title: Producers
---

Dagger Producers is an extension to Dagger that implements
asynchronous dependency injection in Java.


## Overview

This document assumes familiarity with the [Dagger 2 API][Dagger 2] and with
Guava’s [ListenableFuture][ListenableFuture].

Dagger Producers introduces new annotations [@ProducerModule][ProducerModule],
[@Produces][Produces], and [@ProductionComponent][ProductionComponent] as
analogues of [@Module][Module], [@Provides][Provides], and
[@Component][Component]. We refer to classes annotated `@ProducerModule` as
**producer modules**, methods annotated `@Produces` as **producer methods**, and
interfaces annotated `@ProductionComponent` as **producer graphs** (analogous to
**modules**, **provider methods**, and **object graphs**).

The key difference from ordinary Dagger is that producer methods may return
`ListenableFuture<T>` and subsequent producer methods may depend on the
unwrapped `T`; the framework handles scheduling the subsequent methods when the
future is available.

Here is a simple example that mimics a server’s request-handling flow:

```java
@ProducerModule(includes = UserModule.class)
final class UserResponseModule {
  @Produces
  static ListenableFuture<UserData> lookUpUserData(
      User user, UserDataStub stub) {
    return stub.lookUpData(user);
  }

  @Produces
  static Html renderHtml(UserData data, UserHtmlTemplate template) {
    return template.render(data);
  }
}
```

In this example, the computation we’re describing here says:

  * call the `lookUpUserData` method
  * when the future is available, call the `renderHtml` method with the result

(Note that we haven’t explicitly indicated where `User`, `UserDataStub`, or
`UserHtmlTemplate` come from; we’re assuming that the Dagger module `UserModule`
provides those types.)

To build this graph, we use a `ProductionComponent`:

```java
@ProductionComponent(modules = UserResponseModule.class)
interface UserResponseComponent {
  ListenableFuture<Html> html();
}

// ...

UserResponseComponent component = Dagger_UserResponseComponent.builder()
    .executor(executor)
    .build();

ListenableFuture<Html> htmlFuture = component.html();
```

Dagger generates an implementation of the interface `UserResponseComponent`,
whose method `html()` does exactly what we described above: first it calls
`lookUpUserData`, and when that future is available, it calls `renderHtml`.

Both producer methods are scheduled on the provided executor, so the execution
model is entirely user-specified.

## Features

### Works with ordinary Dagger

As in the above example, producer modules can be used seamlessly with ordinary
modules, subject to the restriction that provided types cannot depend on
produced types.

### Exception handling

By default, if a producer method throws an exception, or the future that it
returns failed, then any dependent producer methods will be skipped - this
models “propagating” an exception up a call stack.

If a producer method would like to “catch” that exception, the method can
request a [Produced&lt;T>][Produced] instead of a T. For example:

```java
@Produces
static Html renderHtml(
    Produced<UserData> data,
    UserHtmlTemplate template,
    ErrorHtmlTemplate errorTemplate) {
  try {
    return template.render(data.get());
  } catch (ExecutionException e) {
    return errorTemplate.render("user data failed", e.getCause());
  }
}
```

In this example, if the production of `UserData` threw an exception (either by a
producer method throwing an exception, or by a future failing), then the
`renderHtml` method catches it and returns an error template.

If an exception propagates all the way up to the component’s entry point
without any producer method catching it, then the future returned from the
component will fail with an exception.

### Lazy execution

Producer methods can request a [`Producer<T>`][Producer], which is analogous to
a [`Provider<T>`][Provider]: it delays the computation of the associated binding
until a `get()` method is called. `Producer<T>` is non-blocking; its `get()`
method returns a `ListenableFuture`, which can then be fed to the framework. For
example:

```java
@Produces
static ListenableFuture<UserData> lookUpUserData(
    Flags flags,
    @Standard Producer<UserData> standardUserData,
    @Experimental Producer<UserData> experimentalUserData) {
  return flags.useExperimentalUserData()
      ? experimentalUserData.get()
      : standardUserData.get();
}
```

In this example, if the experimental user data is requested, then the standard
user data is never computed. Note that the `Flags` may be a request-time flag,
or even the result of an RPC, which lets users build very flexible conditional
graphs.

### Multibindings

Several bindings of the same type can be collected into a set or map, just like
in [ordinary Dagger](multibindings.md). For example:

```java
@ProducerModule
final class UserDataModule {
  @Produces(type = SET) static ListenableFuture<Data> standardData(…) { … }
  @Produces(type = SET) static ListenableFuture<Data> extraData(…) { … }
  @Produces(type = SET) static Data synchronousData(…) { … }
  @Produces(type = SET_VALUES) static Set<ListenableFuture<Data>> rest(…) { … }

  @Produces static … collect(Set<Data> data) { … }
}
```

In this example, when all the producer methods that contribute to this set have
completed futures, the `Set<Data>` is constructed and the collect() method is
called.

#### Map multibindings

==As of January 2016, not implemented yet==

Map multibindings are similar to set multibindings:

```java
@MapKey @interface DispatchPath {
  String value();
}

@ProducerModule
final class DispatchModule {
  @Produces(type = MAP) @DispatchPath("/user")
  static ListenableFuture<Html> dispatchUser(…) { … }

  @Produces(type = MAP) @DispatchPath("/settings")
  static ListenableFuture<Html> dispatchSettings(…) { … }

  @Produces
  static ListenableFuture<Html> dispatch(
      Map<DispatchPath, Producer<Html>> dispatchers, Url url) {
    return dispatchers.get(url.path()).get();
  }
}
```

Note that here, `dispatch()` is requesting
`Map<DispatchPath, Producer<Html>>`; this ensures that only the dispatch
handler that was requested will be executed.

### Caching

Each producer method will only be executed once within the context of a given
component, and its result will be cached. This gives complete control over the
lifetime of each binding &mdash; it is the same as the lifetime of its enclosing
component instance.

### Component dependencies

Like ordinary `Component`s, `ProductionComponent`s may depend on other
interfaces:

```java
interface RequestComponent {
  ListenableFuture<Request> request();
}

@ProducerModule
final class UserDataModule {
  @Produces static ListenableFuture<UserData> userData(Request request, …) { … }
}

@ProductionComponent(
    modules = UserDataModule.class,
    dependencies = RequestComponent.class)
interface UserDataComponent {
  ListenableFuture<UserData> userData();
}
```

Since the `UserDataComponent` depends on the `RequestComponent`, Dagger will
require that an instance of the `RequestComponent` be provided when building the
`UserDataComponent`; and then that instance will be used to satisfy the bindings
of the getter methods that it offers:

```java
ListenableFuture<UserData> userData = Dagger_UserDataComponent.builder()
    .executor(executor)
    .requestComponent(/* a particular RequestComponent */)
    .build()
    .userData();
```

### Subcomponents

==As of April 2015, not implemented yet==

Dagger Producers also introduces a new annotation @ProductionSubcomponent as an
analogue to [@Subcomponent][Subcomponent]. Production subcomponents may be
subcomponents of either components or production components.

A subcomponent inherits all bindings from its parent component, and so it is
often a simpler way of building nested scopes.

### Monitoring

[ProducerMonitor][ProducerMonitor] can be used to monitor the execution of
producer methods; its methods correspond to various places in a producer's
lifecycle.

To install a `ProducerMonitor`, contribute into a set binding of
[ProductionComponentMonitor.Factory][ProductionComponentMonitorFactory]. For
example:

```java
@Module
final class MyMonitorModule {
  @Provides(type = SET)
  static ProductionComponentMonitor.Factory provideMonitorFactory(
      MyProductionComponentMonitor.Factory monitorFactory) {
    return monitorFactory;
  }
}

@ProductionComponent(modules = {MyMonitorModule.class, MyProducerModule.class})
interface MyComponent {
  ListenableFuture<SomeType> someType();
}
```

When the component is created, each monitor factory contributed to the set will
be asked to create a monitor for the component. The resulting (single) instance
will be held for the lifetime of the component, and will be used to create
individual monitors for each producer method.

### Timing, Logging and Debugging

==As of January 2016, not implemented yet==

Since graphs are constructed at compile-time, the graph will be able to be
viewed immediately after compiling, likely via writing its structure to json
or a proto.

Statistics from producer execution (CPU usage of producer method invocations and
latency of futures) will be available to export to many endpoints.

While a graph is running, its current state will be able to be dumped.

[Dagger 2]: http://google.github.io/dagger/

[Component]: http://google.github.io/dagger/api/latest/dagger/Component.html
[ListenableFuture]: http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/util/concurrent/ListenableFuture.html
[Module]: http://google.github.io/dagger/api/latest/dagger/Module.html
[Produced]: http://google.github.io/dagger/api/latest/dagger/producers/Produced.html
[Producer]: http://google.github.io/dagger/api/latest/dagger/producers/Producer.html
[ProducerModule]: http://google.github.io/dagger/api/latest/dagger/producers/ProducerModule.html
[ProducerMonitor]: http://google.github.io/dagger/api/latest/dagger/producers/monitoring/ProducerMonitor.html
[Produces]: http://google.github.io/dagger/api/latest/dagger/producers/Produces.html
[ProductionComponent]: http://google.github.io/dagger/api/latest/dagger/producers/ProductionComponent.html
[ProductionComponentMonitor]: http://google.github.io/dagger/api/latest/dagger/producers/monitoring/ProductionComponentMonitor.html
[ProductionComponentMonitorFactory]: http://google.github.io/dagger/api/latest/dagger/producers/monitoring/ProductionComponentMonitor.Factory.html
[Provider]: http://docs.oracle.com/javaee/6/api/javax/inject/Provider.html
[Provides]: http://google.github.io/dagger/api/latest/dagger/Provides.html
[Subcomponent]: http://google.github.io/dagger/api/latest/dagger/Subcomponent.html

