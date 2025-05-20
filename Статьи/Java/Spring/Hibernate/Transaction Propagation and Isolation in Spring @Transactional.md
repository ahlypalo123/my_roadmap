---
title: "Transaction Propagation and Isolation in Spring @Transactional"
source: "https://www.baeldung.com/spring-transactional-propagation-isolation"
author:
  - "[[Baeldung]]"
published: 2019-10-20
created: 2025-05-03
description: "Learn about the isolation and propagation settings in Spring's @Transactional"
tags:
  - "clippings"
---
## 1\. Introduction

In this tutorial, we’ll cover the *@Transactional* annotation, as well as its *isolation* and *propagation* settings.

## Introduction to Transactions in Java and Spring

A quick and practical guide to transactions in Java and Spring.

[Read more](https://www.baeldung.com/?post_type=post&p=84843) →

## Programmatic Transaction Management in Spring

Learn to manage transactions programmatically in Spring and why this approach is sometimes better than simply using the declarative Transactional annotation.

[Read more](https://www.baeldung.com/?post_type=post&p=62777) →

## Detecting If a Spring Transaction Is Active

Let's look at how we can find if a Spring transaction is active.

[Read more](https://www.baeldung.com/?post_type=post&p=88408) →

## 2\. What Is @Transactional?

We can use *[@Transactional](https://www.baeldung.com/transaction-configuration-with-jpa-and-spring)* to wrap a method in a database transaction.

It allows us to set propagation, isolation, timeout, read-only, and rollback conditions for our transaction. We can also specify the transaction manager.

### 2.1. @Transactional Implementation Details

Spring creates a proxy, or manipulates the class byte-code, to manage the creation, commit, and rollback of the transaction. In the case of a proxy, Spring ignores *@Transactional* in internal method calls.

Simply put, if we have a method like *callMethod* and we mark it as *@Transactional,* Spring will wrap some transaction management code around the invocation *@Transactional* method called:

```java
createTransactionIfNecessary();
try {
    callMethod();
    commitTransactionAfterReturning();
} catch (exception) {
    completeTransactionAfterThrowing();
    throw exception;
}
```

### 2.2. How to Use @Transactional

We can put the annotation on definitions of interfaces, classes, or directly on methods. They override each other according to the priority order; from lowest to highest we have: interface, superclass, class, interface method, superclass method, and class method.

**Spring applies the class-level annotation to all public methods of this class that we did not annotate with *@Transactional*.**

**However, if we put the annotation on a private or protected method, Spring will ignore it without an error.**

Let’s start with an interface sample:

```java
@Transactional
public interface TransferService {
    void transfer(String user1, String user2, double val);
}
```

Usually it’s not recommended to set *@Transactional* on the interface; however, it is acceptable for cases like *@Repository* with Spring Data. We can put the annotation on a class definition to override the transaction setting of the interface/superclass:

```java
@Service
@Transactional
public class TransferServiceImpl implements TransferService {
    @Override
    public void transfer(String user1, String user2, double val) {
        // ...
    }
}
```

Now let’s override it by setting the annotation directly on the method:

```java
@Transactional
public void transfer(String user1, String user2, double val) {
    // ...
}
```

## 3\. Transaction Propagation

Propagation defines our business logic’s transaction boundary. Spring manages to start and pause a transaction according to our *propagation* setting.

Spring calls *TransactionManager::getTransaction* to get or create a transaction according to the propagation. It supports some of the propagations for all types of *TransactionManager*, but there are a few of them that are only supported by specific implementations of *TransactionManager*.

Let’s go through the different propagations and how they work.

 - [[3.1. REQUIRED Propagation]]
 - [[3.2. SUPPORTS Propagation]]
 - [[3.3. MANDATORY Propagation]]
 - [[3.4. NEVER Propagation]]
 - [[3.5. NOT_SUPPORTED Propagation]]
 - [[3.6. REQUIRES_NEW Propagation]]
 - [[3.7. NESTED Propagation]]
## 4\. Transaction Isolation

Isolation is one of the common ACID properties: Atomicity, Consistency, Isolation, and Durability. Isolation describes how changes applied by concurrent transactions are visible to each other.

Each isolation level prevents zero or more concurrency side effects on a transaction:

- **Dirty read:** read the uncommitted change of a concurrent transaction
- **Nonrepeatable read**: get different value on re-read of a row if a concurrent transaction updates the same row and commits
- **Phantom read:** get different rows after re-execution of a range query if another transaction adds or removes some rows in the range and commits

We can set the isolation level of a transaction by *@Transactional::isolation.* It has these five enumerations in Spring: *DEFAULT*, *READ\_UNCOMMITTED*, *READ\_COMMITTED*, *REPEATABLE\_READ*, *SERIALIZABLE.*

### 4.1. Isolation Management in Spring

The default isolation level is *DEFAULT*. As a result, when Spring creates a new transaction, the isolation level will be the default isolation of our RDBMS. Therefore, we should be careful if we change the database.

We should also consider cases when we call a chain of methods with different isolation*.* In the normal flow, the isolation only applies when a new transaction is created. Thus, if for any reason we don’t want to allow a method to execute in different isolation, we have to set *TransactionManager::setValidateExistingTransaction* to true.

Then the pseudo-code of transaction validation will be:

```java
if (isolationLevel != ISOLATION_DEFAULT) {
    if (currentTransactionIsolationLevel() != isolationLevel) {
        throw IllegalTransactionStateException
    }
}
```

Now let’s get deep in different isolation levels and their effects.

[[Уровни изоляции транзакций в БД]]
## 5\. Conclusion

In this article, we explored the propagation property of *@Transaction* in detail. We then learned about concurrency side effects and isolation levels.

The code backing this article is available on GitHub. Once you're **logged in as a [Baeldung Pro Member](https://www.baeldung.com/members/)**, start learning and coding on the project.