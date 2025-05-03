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

### 3.1. REQUIRED Propagation

*REQUIRED* is the default propagation. Spring checks if there is an active transaction, and if nothing exists, it creates a new one. Otherwise, the business logic appends to the currently active transaction:

```java
@Transactional(propagation = Propagation.REQUIRED)
public void requiredExample(String user) { 
    // ... 
}
```

Furthermore, since *REQUIRED* is the default propagation, we can simplify the code by dropping it:

```java
@Transactional
public void requiredExample(String user) { 
    // ... 
}
```

Let’s see the pseudo-code of how transaction creation works for *REQUIRED* propagation:

```java
if (isExistingTransaction()) {
    if (isValidateExistingTransaction()) {
        validateExisitingAndThrowExceptionIfNotValid();
    }
    return existing;
}
return createNewTransaction();
```

### 3.2. SUPPORTS Propagation

For *SUPPORTS*, Spring first checks if an active transaction exists. If a transaction exists, then the existing transaction will be used. If there isn’t a transaction, it is executed non-transactional:

```java
@Transactional(propagation = Propagation.SUPPORTS)
public void supportsExample(String user) { 
    // ... 
}
```

Let’s see the transaction creation’s pseudo-code for *SUPPORTS*:

```java
if (isExistingTransaction()) {
    if (isValidateExistingTransaction()) {
        validateExisitingAndThrowExceptionIfNotValid();
    }
    return existing;
}
return emptyTransaction;
```

### 3.3. MANDATORY Propagation

When the propagation is *MANDATORY*, if there is an active transaction, then it will be used. If there isn’t an active transaction, then Spring throws an exception:

```java
@Transactional(propagation = Propagation.MANDATORY)
public void mandatoryExample(String user) { 
    // ... 
}
```

Let’s again see the pseudo-code:

```java
if (isExistingTransaction()) {
    if (isValidateExistingTransaction()) {
        validateExisitingAndThrowExceptionIfNotValid();
    }
    return existing;
}
throw IllegalTransactionStateException;
```

### 3.4. NEVER Propagation

For transactional logic with *NEVER* propagation, Spring throws an exception if there’s an active transaction:

```java
@Transactional(propagation = Propagation.NEVER)
public void neverExample(String user) { 
    // ... 
}
```

Let’s see the pseudo-code of how transaction creation works for *NEVER* propagation:

```java
if (isExistingTransaction()) {
    throw IllegalTransactionStateException;
}
return emptyTransaction;
```

### 3.5. NOT\_SUPPORTED Propagation

If a current transaction exists, first Spring suspends it, and then the business logic is executed without a transaction:

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void notSupportedExample(String user) { 
    // ... 
}
```

**The *JTATransactionManager* supports real transaction suspension out-of-the-box. Others simulate the suspension by holding a reference to the existing one and then clearing it from the thread context**

### 3.6. REQUIRES\_NEW Propagation

When the propagation is *REQUIRES\_NEW*, Spring suspends the current transaction if it exists, and then creates a new one:

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void requiresNewExample(String user) { 
    // ... 
}
```

**Similar to *NOT\_SUPPORTED*, we need the *JTATransactionManager* for actual transaction suspension.**

The pseudo-code looks like so:

```java
if (isExistingTransaction()) {
    suspend(existing);
    try {
        return createNewTransaction();
    } catch (exception) {
        resumeAfterBeginException();
        throw exception;
    }
}
return createNewTransaction();
```

### 3.7. NESTED Propagation

For *NESTED* propagation, Spring checks if a transaction exists, and if so, it marks a save point. This means that if our business logic execution throws an exception, then the transaction rollbacks to this save point. If there’s no active transaction, it works like *REQUIRED*.

***DataSourceTransactionManager* supports this propagation out-of-the-box. Some implementations of *JTATransactionManager* may also support this.**

**[*JpaTransactionManager*](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/orm/jpa/JpaTransactionManager.html) supports *NESTED* only for JDBC connections. However, if we set the *nestedTransactionAllowed* flag to *true*, it also works for JDBC access code in JPA transactions if our JDBC driver supports save points.  
**

Finally, let’s set the *propagation* to *NESTED*:

```java
@Transactional(propagation = Propagation.NESTED)
public void nestedExample(String user) { 
    // ... 
}
```

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

### 4.2. READ\_UNCOMMITTED Isolation

*READ\_UNCOMMITTED* is the lowest isolation level and allows for the most concurrent access.

As a result, it suffers from all three mentioned concurrency side effects. A transaction with this isolation reads uncommitted data of other concurrent transactions. Also, both non-repeatable and phantom reads can happen. Thus we can get a different result on re-read of a row or re-execution of a range query.

We can set the *isolation* level for a method or class:

```java
@Transactional(isolation = Isolation.READ_UNCOMMITTED)
public void log(String message) {
    // ...
}
```

**Postgres does not support *READ\_UNCOMMITTED* isolation and falls back to *READ\_COMMITED* instead.** **Also, Oracle does not support or allow *READ\_UNCOMMITTED*.  
**

### 4.3. READ\_COMMITTED Isolation

The second level of isolation, *READ\_COMMITTED,* prevents dirty reads.

The rest of the concurrency side effects could still happen. So uncommitted changes in concurrent transactions have no impact on us, but if a transaction commits its changes, our result could change by re-querying.

Here we set the *isolation* level:

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void log(String message){
    // ...
}
```

***READ\_COMMITTED* is the default level with Postgres, SQL Server, and Oracle.**

### 4.4. REPEATABLE\_READ Isolation

The third level of isolation, *REPEATABLE\_READ,* prevents dirty, and non-repeatable reads. So we are not affected by uncommitted changes in concurrent transactions.

Also, when we re-query for a row, we don’t get a different result. However, in the re-execution of range-queries, we may get newly added or removed rows.

Moreover, it is the lowest required level to prevent the lost update. The lost update occurs when two or more concurrent transactions read and update the same row. *REPEATABLE\_READ* does not allow simultaneous access to a row at all. Hence the lost update can’t happen.  

Here is how to set the *isolation* level for a method:

```java
@Transactional(isolation = Isolation.REPEATABLE_READ) 
public void log(String message){
    // ...
}
```

***REPEATABLE\_READ* is the default level in Mysql.** **Oracle does not support *REPEATABLE\_READ*.**

### 4.5. SERIALIZABLE Isolation

*SERIALIZABLE* is the highest level of isolation. It prevents all mentioned concurrency side effects, but can lead to the lowest concurrent access rate because it executes concurrent calls sequentially.

In other words, concurrent execution of a group of serializable transactions has the same result as executing them in serial.

Now let’s see how to set *SERIALIZABLE* as the *isolation* level:

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void log(String message){
    // ...
}
```

## 5\. Conclusion

In this article, we explored the propagation property of *@Transaction* in detail. We then learned about concurrency side effects and isolation levels.

The code backing this article is available on GitHub. Once you're **logged in as a [Baeldung Pro Member](https://www.baeldung.com/members/)**, start learning and coding on the project.