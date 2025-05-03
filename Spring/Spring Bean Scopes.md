---
title: "Quick Guide to Spring Bean Scopes"
source: "https://www.baeldung.com/spring-bean-scopes"
author:
  - "[[Baeldung]]"
published: 2016-06-03
created: 2025-05-03
description: "Learn how to quickly navigate the available bean scopes in the Spring framework."
tags:
  - "clippings"
---
## 1\. Overview

In this quick tutorial, we’ll learn about the different types of bean scopes in the Spring framework.

The scope of a bean defines the life cycle and visibility of that bean in the contexts we use it.

The latest version of the Spring framework defines 6 types of scopes:

- singleton
- prototype
- request
- session
- application
- websocket

The last four scopes mentioned, *request, session, application* and *websocket*, are only available in a web-aware application.

## What Is a Spring Bean?

A quick and practical explanation of what a Spring Bean is.

[Read more](https://www.baeldung.com/?post_type=post&p=41330) →

## Spring Bean Annotations

Learn how and when to use the standard Spring bean annotations - @Component, @Repository, @Service and @Controller.

[Read more](https://www.baeldung.com/?post_type=post&p=8176) →

## 2\. Singleton Scope

When we define a bean with the *singleton* scope, the container creates a single instance of that bean; all requests for that bean name will return the same object, which is cached. Any modifications to the object will be reflected in all references to the bean. This scope is the default value if no other scope is specified.

Let’s create a *Person* entity to exemplify the concept of scopes:

```java
public class Person {
    private String name;

    // standard constructor, getters and setters
}
```

Afterwards, we define the bean with the *singleton* scope by using the *@Scope* annotation:

```java
@Bean
@Scope("singleton")
public Person personSingleton() {
    return new Person();
}
```

We can also use a constant instead of the *String* value in the following manner:

```java
@Scope(value = ConfigurableBeanFactory.SCOPE_SINGLETON)
```

Now we can proceed to write a test that shows that two objects referring to the same bean will have the same values, even if only one of them changes their state, as they are both referencing the same bean instance:

```java
private static final String NAME = "John Smith";

@Test
public void givenSingletonScope_whenSetName_thenEqualNames() {
    ApplicationContext applicationContext = 
      new ClassPathXmlApplicationContext("scopes.xml");

    Person personSingletonA = (Person) applicationContext.getBean("personSingleton");
    Person personSingletonB = (Person) applicationContext.getBean("personSingleton");

    personSingletonA.setName(NAME);
    Assert.assertEquals(NAME, personSingletonB.getName());

    ((AbstractApplicationContext) applicationContext).close();
}
```

The *scopes.xml* file in this example should contain the xml definitions of the beans used:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans 
    http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="personSingleton" class="org.baeldung.scopes.Person" scope="singleton"/>    
</beans>
```

## 3\. Prototype Scope

A bean with the *prototype*  scope will return a different instance every time it is requested from the container. It is defined by setting the value *prototype* to the *@Scope* annotation in the bean definition:

```java
@Bean
@Scope("prototype")
public Person personPrototype() {
    return new Person();
}
```

We can also use a constant like we did for the *singleton* scope:

```java
@Scope(value = ConfigurableBeanFactory.SCOPE_PROTOTYPE)
```

We will now write a similar test as before that shows two objects requesting the same bean name with the *prototype* scope. They will have different states as they are no longer referring to the same bean instance:

```java
private static final String NAME = "John Smith";
private static final String NAME_OTHER = "Anna Jones";

@Test
public void givenPrototypeScope_whenSetNames_thenDifferentNames() {
    ApplicationContext applicationContext = 
      new ClassPathXmlApplicationContext("scopes.xml");

    Person personPrototypeA = (Person) applicationContext.getBean("personPrototype");
    Person personPrototypeB = (Person) applicationContext.getBean("personPrototype");

    personPrototypeA.setName(NAME);
    personPrototypeB.setName(NAME_OTHER);

    Assert.assertEquals(NAME, personPrototypeA.getName());
    Assert.assertEquals(NAME_OTHER, personPrototypeB.getName());

    ((AbstractApplicationContext) applicationContext).close();
}
```

The *scopes.xml* file is similar to the one presented in the previous section while adding the xml definition for the bean with the *prototype* scope:

```xml
<bean id="personPrototype" class="org.baeldung.scopes.Person" scope="prototype"/>
```

## 4\. Web Aware Scopes

As previously mentioned, there are four additional scopes that are only available in a web-aware application context. We use these less often in practice.

The *request* scope creates a bean instance for a single HTTP request, while the s *ession* scope creates a bean instance for an HTTP Session.

The *application* scope creates the bean instance for the lifecycle of a *ServletContext*, and the *websocket* scope creates it for a particular *WebSocket* session.

Let’s create a class to use for instantiating the beans:

```java
public class HelloMessageGenerator {
    private String message;
    
    // standard getter and setter
}
```

### 4.1. Request Scope

We can define the bean with the *request* scope using the *@Scope* annotation:

```java
@Bean
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public HelloMessageGenerator requestScopedBean() {
    return new HelloMessageGenerator();
}
```

The *proxyMode* attribute is necessary because at the moment of the instantiation of the web application context, there is no active request. Spring creates a proxy to be injected as a dependency, and instantiates the target bean when it is needed in a request.

We can also use a *@RequestScope* composed annotation that acts as a shortcut for the above definition:

```java
@Bean
@RequestScope
public HelloMessageGenerator requestScopedBean() {
    return new HelloMessageGenerator();
}
```

Next we can define a controller that has an injected reference to the *requestScopedBean*. We need to access the same request twice in order to test the web specific scopes.

If we display the *message* each time the request is run, we can see that the value is reset to *null*, even though it is later changed in the method. This is because of a different bean instance being returned for each request.

```java
@Controller
public class ScopesController {
    @Resource(name = "requestScopedBean")
    HelloMessageGenerator requestScopedBean;

    @RequestMapping("/scopes/request")
    public String getRequestScopeMessage(final Model model) {
        model.addAttribute("previousMessage", requestScopedBean.getMessage());
        requestScopedBean.setMessage("Good morning!");
        model.addAttribute("currentMessage", requestScopedBean.getMessage());
        return "scopesExample";
    }
}
```

### 4.2. Session Scope

We can define the bean with the *session* scope in a similar manner:

```java
@Bean
@Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
public HelloMessageGenerator sessionScopedBean() {
    return new HelloMessageGenerator();
}
```

There’s also a dedicated composed annotation we can use to simplify the bean definition:

```java
@Bean
@SessionScope
public HelloMessageGenerator sessionScopedBean() {
    return new HelloMessageGenerator();
}
```

Next we define a controller with a reference to the *sessionScopedBean*. Again, we need to run two requests in order to show that the value of the *message* field is the same for the session.

In this case, when the request is made for the first time, the value *message* is *null.* However, once it is changed, that value is retained for subsequent requests as the same instance of the bean is returned for the entire session.

```java
@Controller
public class ScopesController {
    @Resource(name = "sessionScopedBean")
    HelloMessageGenerator sessionScopedBean;

    @RequestMapping("/scopes/session")
    public String getSessionScopeMessage(final Model model) {
        model.addAttribute("previousMessage", sessionScopedBean.getMessage());
        sessionScopedBean.setMessage("Good afternoon!");
        model.addAttribute("currentMessage", sessionScopedBean.getMessage());
        return "scopesExample";
    }
}
```

### 4.3. Application Scope

The *application* scope creates the bean instance for the lifecycle of a *ServletContext.*

This is similar to the *singleton* scope, but there is a very important difference with regards to the scope of the bean.

When beans are *application* scoped, the same instance of the bean is shared across multiple servlet-based applications running in the same *ServletContext*, while *singleton* scoped beans are scoped to a single application context only.

Let’s create the bean with the *application* scope:

```java
@Bean
@Scope(
  value = WebApplicationContext.SCOPE_APPLICATION, proxyMode = ScopedProxyMode.TARGET_CLASS)
public HelloMessageGenerator applicationScopedBean() {
    return new HelloMessageGenerator();
}
```

Analogous to the *request* and *session* scopes, we can use a shorter version:

```java
@Bean
@ApplicationScope
public HelloMessageGenerator applicationScopedBean() {
    return new HelloMessageGenerator();
}
```

Now let’s create a controller that references this bean:

```java
@Controller
public class ScopesController {
    @Resource(name = "applicationScopedBean")
    HelloMessageGenerator applicationScopedBean;

    @RequestMapping("/scopes/application")
    public String getApplicationScopeMessage(final Model model) {
        model.addAttribute("previousMessage", applicationScopedBean.getMessage());
        applicationScopedBean.setMessage("Good afternoon!");
        model.addAttribute("currentMessage", applicationScopedBean.getMessage());
        return "scopesExample";
    }
}
```

In this case, once set in the *applicationScopedBean*, the value *message* will be retained for all subsequent requests, sessions and even for different servlet applications that will access this bean, provided it is running in the same *ServletContext.*

### 4.4. WebSocket Scope

Finally, let’s create the bean with the *websocket* scope:

```java
@Bean
@Scope(scopeName = "websocket", proxyMode = ScopedProxyMode.TARGET_CLASS)
public HelloMessageGenerator websocketScopedBean() {
    return new HelloMessageGenerator();
}
```

When first accessed, *WebSocket* scoped beans are stored in the *WebSocket* session attributes. The same instance of the bean is then returned whenever that bean is accessed during the entire *WebSocket* session.

We can also say that it exhibits singleton behavior, but limited to a *W* *ebSocket* session only.

## 5\. Conclusion

In this article, we discussed the different bean scopes provided by Spring and what their intended uses are.

The code backing this article is available on GitHub. Once you're **logged in as a [Baeldung Pro Member](https://www.baeldung.com/members/)**, start learning and coding on the project.

Follow the Spring Category

Follow the Spring category to get regular info about the new articles and tutorials we publish here.