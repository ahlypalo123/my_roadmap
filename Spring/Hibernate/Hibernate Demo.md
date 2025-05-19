# Hibernate и Transactional

1. [Примеры](#examples)
2. [Ссылки на доклады](#%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D0%B8-%D0%BD%D0%B0-%D0%B4%D0%BE%D0%BA%D0%BB%D0%B0%D0%B4%D1%8B)

## [](#%D0%BF%D1%80%D0%B8%D0%BC%D0%B5%D1%80%D1%8B)Примеры

1. [Пример транзакции](#transactional_example)
2. [Конкурирующий доступ к одной строке БД несколькими транзакциями](#transactional_problem)
3. [Пример оптимистической блокировки](#optimistic_lock_example)
4. [Используйте пессимистические блокировки с осторожностью, так как они могут породить deadlock](#dead_lock_example)
5. [Кеширование в Spring](#spring_caching)
6. [Кеширование 1 уровня в Hibernate](#first_level_cache_problem)
7. [Кеширование 2 уровня](#second_level_cache_problem)
8. [Кеширование уровня запроса](#query_cache)
9. [Проблема N+1](#n_plus_1_problem)
10. [Использование Hibernate JCache](#using_jcache)

### [](#%D0%BF%D1%80%D0%B8%D0%BC%D0%B5%D1%80-1-%D0%BF%D1%80%D0%B8%D0%BC%D0%B5%D1%80-%D1%82%D1%80%D0%B0%D0%BD%D0%B7%D0%B0%D0%BA%D1%86%D0%B8%D0%B8)_Пример 1_. Пример транзакции

Метод `transactionalExample` с аннотацией `@Transactional` вынесен в отдельный сервис. Если бы метод был в текущем классе, то при вызове `this.transactionalExample`, транзакция бы не сработала. Происходит это из-за того, что для класса, где есть `@Transactional` создается proxy бин, а при вызове this вызывается оригинальный класс

[GrabliTest.java](/learning/hibernate_demo/-/blob/main/src/test/java/ru/obteh/learning/hibernate/demo/GrabliTest.java):

```java
    @Test
    void transactional_example() {
        employeeRepository.deleteAll();
        try {
            transactionalService.transactionalExample();
            //  transactionalExample();
    } catch (Exception e) {
        log.error(e.getMessage(), e);
    }
    Assertions.assertEquals(0, employeeRepository.count());
}
```

[TransactionalService.java](/learning/hibernate_demo/-/blob/main/src/test/java/ru/obteh/learning/hibernate/demo/TransactionalService.java):

```java
    @Transactional
    public void transactionalExample() {
        employeeRepository.save(EmployeeEntity.builder()
                .jobTitle("Java-developer")
                .build());
        throw new RuntimeException("Я упал((((");
    }
```

### [](#%D0%BF%D1%80%D0%B8%D0%BC%D0%B5%D1%80-2-%D0%BA%D0%BE%D0%BD%D0%BA%D1%83%D1%80%D0%B8%D1%80%D1%83%D1%8E%D1%89%D0%B8%D0%B9-%D0%B4%D0%BE%D1%81%D1%82%D1%83%D0%BF-%D0%BA-%D0%BE%D0%B4%D0%BD%D0%BE%D0%B9-%D1%81%D1%82%D1%80%D0%BE%D0%BA%D0%B5-%D0%B1%D0%B4-%D0%BD%D0%B5%D1%81%D0%BA%D0%BE%D0%BB%D1%8C%D0%BA%D0%B8%D0%BC%D0%B8-%D1%82%D1%80%D0%B0%D0%BD%D0%B7%D0%B0%D0%BA%D1%86%D0%B8%D1%8F%D0%BC%D0%B8)_Пример 2_. Конкурирующий доступ к одной строке БД несколькими транзакциями

По умолчанию в PostgreSQL используется уровень изоляции транзакций `READ_COMMITTED`, в таком случае возможны чтения данных, которые были уже изменены другой транзакцией

Таким образом может произойти следующая ситуация:

```
         thread 1
   получить баланс (0)         thread 2
    сложить (0 + 10)      получить баланс (0)
   сохранить в БД (10)     сложить (0 + 10)
                          сохранить в БД (10)
```

Оба потока сохранили в базу значение 10, а ожидалось 20. Решением проблемы будет использования уровня изоляции `REPEATABLE_READ` или `SERIALIZABLE`, либо пессимистической/оптимистической блокировки.

Обратите внимание, на метод `addToBalanceByIdRepeatableRead` повешен `@Retryable`, потому что при уровне изоляции `REPEATABLE_READ` возникает исключение при попытке доступа к одной строчке конкурентными транзакциями.

[GrabliTest.java](/learning/hibernate_demo/-/blob/main/src/test/java/ru/obteh/learning/hibernate/demo/GrabliTest.java):

```java
    @Test
    void transactional_problem() {
        peopleRepository.deleteAll();
        long id = peopleRepository.save(PeopleEntity.builder()
            .name("Tom Hardy")
            .balance(0)
            .build()).getId();
        Random r = new Random();
        List<Integer> numbers = Stream.generate(() -> r.nextInt(0, 100)).limit(10).toList(); // Сгенерируем 10 рандомных чисел
        int expected = numbers.stream().reduce(Integer::sum).get(); // Вычислим сумму
        numbers.parallelStream().forEach(n -> { // Параллельно запустим поток для каждого числа
            // transactionalService.addToBalanceById(id, n); // Проблема
            transactionalService.addToBalanceByIdRepeatableRead(id, n); // Вариант решения с уровнем изоляции  REPEATABLE_READ
            // transactionalService.addToBalanceByIdSerializable(id, n); // // Вариант решения с уровнем изоляции  SERIALIZABLE (Может быть deadlock)
            // transactionalService.addToBalanceByIdPessimisticLock(id, n); // Вариант решения с пессимистической блокировкой (Может быть deadlock)
        });
        long actual = peopleRepository.findById(id).get().getBalance();
        log.info("expected: {}, actual: {}", expected, actual);
    }
```

[TransactionalService.java](/learning/hibernate_demo/-/blob/main/src/test/java/ru/obteh/learning/hibernate/demo/TransactionalService.java):

```java
    @Transactional
    public void addToBalanceById(long id, int n) {
        PeopleEntity p = peopleRepository.findById(id).get();
        long balance = p.getBalance() + n;
        p.setBalance(balance);
        peopleRepository.save(p);
    }

    @Retryable(maxAttempts = 10)
    @Transactional(isolation = Isolation.REPEATABLE_READ)
    public void addToBalanceByIdRepeatableRead(long id, int n) {
        PeopleEntity p = peopleRepository.findById(id).get();
        long balance = p.getBalance() + n;
        p.setBalance(balance);
        peopleRepository.save(p);
    }

    @Transactional
    public void addToBalanceByIdPessimisticLock(long id, int n) {
        PeopleEntity p = peopleRepository.findByIdPessimistic(id);
        long balance = p.getBalance() + n;
        p.setBalance(balance);
        peopleRepository.save(p);
    }

    @Transactional(isolation = Isolation.SERIALIZABLE) // Можно обойтись без Retryable
    public void addToBalanceByIdSerializable(long id, int n) {
        PeopleEntity p = peopleRepository.findById(id).get();
        long balance = p.getBalance() + n;
        p.setBalance(balance);
        peopleRepository.save(p);
    }
```

Для пессимистической блокировки на методе репозитория указывается аннотация `@Lock(LockModeType.PESSIMISTIC_WRITE)`

[PeopleRepository.java](/learning/hibernate_demo/-/blob/main/src/main/java/ru/obteh/learning/hibernate/demo/repository/PeopleRepository.java):

```java
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM PeopleEntity p WHERE p.id = :id")
    PeopleEntity findByIdPessimistic(long id);
```

### [](#%D0%BF%D1%80%D0%B8%D0%BC%D0%B5%D1%80-3-%D0%BF%D1%80%D0%B8%D0%BC%D0%B5%D1%80-%D0%BE%D0%BF%D1%82%D0%B8%D0%BC%D0%B8%D1%81%D1%82%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%BE%D0%B9-%D0%B1%D0%BB%D0%BE%D0%BA%D0%B8%D1%80%D0%BE%D0%B2%D0%BA%D0%B8)_Пример 3_. Пример оптимистической блокировки

Тут все то же самое, что и в примере 2, только с оптимистической блокировкой. Тут вообще не нужна транзакция, но нужны retry

[GrabliTest.java](/learning/hibernate_demo/-/blob/main/src/test/java/ru/obteh/learning/hibernate/demo/GrabliTest.java):

```java
    @Test
    void optimistic_lock_example() {
        purchaseRepository.deleteAll();
        long id = purchaseRepository.save(PurchaseEntity.builder()
            .amount(100)
            .product("За огурцы")
            .build()).getId();
        Random r = new Random();
        List<Integer> numbers = Stream.generate(() -> r.nextInt(0, 100)).limit(10).toList(); // Сгенерируем 10 рандомных чисел
        int expected = numbers.stream().reduce(Integer::sum).get() + 100; // Вычислим сумму
        numbers.parallelStream().forEach(n -> {
            transactionalService.addToBalanceByIdOptimistic(id, n);
        });
        long actual = purchaseRepository.findById(id).get().getAmount();
    }
```

[TransactionalService.java](/learning/hibernate_demo/-/blob/main/src/test/java/ru/obteh/learning/hibernate/demo/TransactionalService.java):

```java
    @Retryable(maxAttempts = 10)
    public void addToBalanceByIdOptimistic(long id, int n) {
        PurchaseEntity p = purchaseRepository.findById(id).get();
        long amount = p.getAmount() + n;
        p.setAmount(amount);
        purchaseRepository.save(p);
    }
```

Для оптимистической блокировки указывается аннотация `@Version` на поле сущности, для которой применяется блокировка. Это поле должно быть уникальным для каждого обновления строки в БД.

[PurchaseEntity.java](/learning/hibernate_demo/-/blob/main/src/main/java/ru/obteh/learning/hibernate/demo/model/PurchaseEntity.java):

```java
    @Version
    @UpdateTimestamp
    @Column(nullable = false)
    private LocalDateTime updateTime;
```

### [](#%D0%BF%D1%80%D0%B8%D0%BC%D0%B5%D1%80-4-%D0%B8%D1%81%D0%BF%D0%BE%D0%BB%D1%8C%D0%B7%D1%83%D0%B9%D1%82%D0%B5-%D0%BF%D0%B5%D1%81%D1%81%D0%B8%D0%BC%D0%B8%D1%81%D1%82%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%B8%D0%B5-%D0%B1%D0%BB%D0%BE%D0%BA%D0%B8%D1%80%D0%BE%D0%B2%D0%BA%D0%B8-%D1%81-%D0%BE%D1%81%D1%82%D0%BE%D1%80%D0%BE%D0%B6%D0%BD%D0%BE%D1%81%D1%82%D1%8C%D1%8E-%D1%82%D0%B0%D0%BA-%D0%BA%D0%B0%D0%BA-%D0%BE%D0%BD%D0%B8-%D0%BC%D0%BE%D0%B3%D1%83%D1%82-%D0%BF%D0%BE%D1%80%D0%BE%D0%B4%D0%B8%D1%82%D1%8C-deadlock)_Пример 4_. Используйте пессимистические блокировки с осторожностью, так как они могут породить deadlock

Здесь используется тот же метод `findByIdPessimistic`, что и в прошлом примере. Метод `addToBalanceByIdPessimisticLock` аннотирован `@Transactional` с типом распространения `REQUIRES_NEW`, это значит что при вызове метода будет всегда открываться новая транзакция. По умолчанию он бы присоединялся к предыдущей.

`prepareDeadLock` подготавливает данные для теста. Для него тоже открывается отдельная транзакция.

На последней строчке теста выводится сообщение в лог, но мы его никогда не увидим, так как при вызове `addToBalanceByIdPessimisticLock` произойдет dead lock. Метод `dead_lock_example` открывает транзакцию и методом `findByIdPessimistic` блокирует строку в БД. Когда вызывается метод `addToBalanceByIdPessimisticLock`, он открывает новую транзакцию и пытается заблокировать ту же строку, но не может, так как она уже заблокирована.

[GrabliTest.java](/learning/hibernate_demo/-/blob/main/src/test/java/ru/obteh/learning/hibernate/demo/GrabliTest.java):

```java
    @Test
    @Transactional
    void dead_lock_example() {
        long id = transactionalService.prepareDeadLock();
        PeopleEntity p = peopleRepository.findByIdPessimistic(id);
        log.info("Добавляем баланс пользователю {}", p.getName());
        transactionalService.addToBalanceByIdPessimisticLock(id, 100); // Вызов этого метода породит дэдлок, так как ресурс уже занят другой транзакцией
        p = peopleRepository.findById(id).get();
        log.info("Баланс пользователя {} изменен на {}", p.getName(), p.getBalance()); // Мы никогда не увидим это сообщение(
    }
```

[TransactionalService.java](/learning/hibernate_demo/-/blob/main/src/test/java/ru/obteh/learning/hibernate/demo/TransactionalService.java):

```java
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public long prepareDeadLock() {
        peopleRepository.deleteAll();
        return peopleRepository.save(PeopleEntity.builder()
            .name("Ryan Gosling")
            .balance(0)
            .build()).getId();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void addToBalanceByIdPessimisticLock(long id, int n) {
        PeopleEntity p = peopleRepository.findByIdPessimistic(id);
        long balance = p.getBalance() + n;
        p.setBalance(balance);
        peopleRepository.save(p);
    }
```

### [](#%D0%BF%D1%80%D0%B8%D0%BC%D0%B5%D1%80-5-%D0%BA%D0%B5%D1%88%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5-%D0%B2-spring)_Пример 5_. Кеширование в Spring

Spring поддерживает кеширование результатов выполнения методов с помощью аннотаций `@Cacheable`, `@Caching`, `@CachePut`. По умолчанию используется реализация кеширования с помощью `java.util.concurrent.ConcurrentMap`, но можно также подключить свою реализацию.

Подробнее про Cache Abstraction в Spring: [https://docs.spring.io/spring-framework/docs/4.3.x/spring-framework-reference/html/cache.html](https://docs.spring.io/spring-framework/docs/4.3.x/spring-framework-reference/html/cache.html)

[GrabliTest.java](/learning/hibernate_demo/-/blob/main/src/test/java/ru/obteh/learning/hibernate/demo/GrabliTest.java):

```java
    @Test
    void springCaching() {
        peopleService.deleteAll();

        List<Long> ids = new ArrayList<>();

        ids.add(peopleService.save(PeopleEntity.builder()
            .name("Tony Stark")
            .balance(99999999999999L)
            .build()).getId());
        ids.add(peopleService.save(PeopleEntity.builder()
            .name("Tyler Durden")
            .balance(100)
            .build()).getId());

        for (Long id : ids) {
            log.info("People with id {}: {}", id, peopleService.findById(id));
        }
        ids.add(peopleService.save(PeopleEntity.builder()
            .name("Tom Hardy")
            .balance(100)
            .build()).getId());
        for (Long id : ids) {
            log.info("People with id {}: {}", id, peopleService.findById(id));
        }
    }
```

[PeopleService](/learning/hibernate_demo/-/blob/main/src/main/java/ru/obteh/learning/hibernate/demo/service/PeopleService.java):

```java
@Slf4j
@Service
@AllArgsConstructor
public class PeopleService {

    private final PeopleRepository peopleRepository;

    public Iterable<PeopleEntity> findAll(PeopleEntity peopleEntity) {
        log.info("findAll method called");
        return peopleRepository.findAll();
    }

    @CachePut(value = "people", key = "#result.id")
    public PeopleEntity save(PeopleEntity peopleEntity) {
        log.info("save method called");
        return peopleRepository.save(peopleEntity);
    }

    @Cacheable("people")
    public PeopleEntity findById(long id) {
        log.info("findById method called");
        return peopleRepository.findById(id).get();
    }

    @CacheEvict("people")
    public void deleteAll() {
        log.info("deleteAll method called");
        peopleRepository.deleteAll();
    }
}
```

### [](#%D0%BF%D1%80%D0%B8%D0%BC%D0%B5%D1%80-6-%D0%BA%D0%B5%D1%88%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5-1-%D1%83%D1%80%D0%BE%D0%B2%D0%BD%D1%8F-%D0%B2-hibernate)_Пример 6_. Кеширование 1 уровня в Hibernate

В Hibernate по умолчанию активно кеширование 1 уровня. При этом данные кешируются в пределах сессии. Сессия активна в пределах метода, аннотированного `@Transactional`. Проблема в том, что данные могут быть изменены за пределами JPA приложения, например с помощью JdbcTemplate, из-за чего кэш может устаревать.

В методе `withCaching` при вызове `findAll` данные записываются в кэш первого уровня. Таким образом при вызове `withCaching` возвращается баланс из кэша, а при вызове `withoutCaching` из БД.

> Также при большой выборке данных необходимо следить за потреблением памяти и по необходимости чистить кэш

[GrabliTest.java](/learning/hibernate_demo/-/blob/main/src/test/java/ru/obteh/learning/hibernate/demo/GrabliTest.java):

```java
    @Test
    void first_level_cache_problem() {
        peopleRepository.deleteAll();
        // Нужно также следить за потреблением памяти
        peopleRepository.save(PeopleEntity.builder()
            .name("Ryan Gosling")
            .balance(100)
            .build());
        Assertions.assertEquals(100, transactionalService.withCaching());
        Assertions.assertEquals(0, transactionalService.withoutCaching());
    }
```

[TransactionalService.java](/learning/hibernate_demo/-/blob/main/src/test/java/ru/obteh/learning/hibernate/demo/TransactionalService.java):

```java
    @Transactional
    public long withoutCaching() {
        jdbcTemplate.update("UPDATE people SET balance = 0");
        return peopleRepository.findAll().iterator().next().getBalance();
    }

    @Transactional
    public long withCaching() {
        peopleRepository.findAll();
        jdbcTemplate.update("UPDATE people SET balance = 0");
        return peopleRepository.findAll().iterator().next().getBalance();
    }
```

### [](#%D0%BF%D1%80%D0%B8%D0%BC%D0%B5%D1%80-7-%D0%BA%D0%B5%D1%88%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5-2-%D1%83%D1%80%D0%BE%D0%B2%D0%BD%D1%8F)_Пример 7_. Кеширование 2 уровня

Особенность второго уровня кэша в том, что он активен за пределами сессии. Данный уровень кэша по умолчанию не активен в Hibernate. Для его активации необходимо добавить зависимость ehcache

[pom.xml](/learning/hibernate_demo/-/blob/main/pom.xml):

```xml
<dependency>
	<groupId>org.hibernate</groupId>
	<artifactId>hibernate-ehcache</artifactId>
</dependency>
```

Чтобы Hibernate не скачал со временем всю базу себе в память и не упал с OutOfMemoryError, нужно настроить ehcache

[ehcache.xml](/learning/hibernate_demo/-/blob/main/src/main/resources/ehcache.xml)

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<ehcache>
    <defaultCache
            maxElementsInMemory="100000"
            eternal="false"
            timeToIdleSeconds="1200"
            timeToLiveSeconds="1200"
            diskExpiryThreadIntervalSeconds="1200" />
</ehcache>
```

Кэш 2 уровня активен для сущности с аннотацией `@Cacheable`

[EmployeeEntity.java](/learning/hibernate_demo/-/blob/main/src/main/java/ru/obteh/learning/hibernate/demo/model/EmployeeEntity.java)

```java
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CacheConcurrencyStrategy;

import javax.persistence.*;

@Data
@Table(name = "employee")
@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class EmployeeEntity {
    // ...
}
```

[Подробнее про стратегии кеширования](https://docs.jboss.org/hibernate/core/3.3/reference/en/html/performance.html#performance-cache-mapping)

Проблема здесь та же, что и в примере 5. Данные могут быть изменены за пределами JPA приложения, поэтому кэш может устареть.

[GrabliTest.java](/learning/hibernate_demo/-/blob/main/src/test/java/ru/obteh/learning/hibernate/demo/GrabliTest.java):

```java
    @Test
    void second_level_cache_problem() {
        employeeRepository.deleteAll();
        long id = employeeRepository.save(EmployeeEntity.builder()
            .jobTitle("Java-developer")
            .build()).getId();
        employeeRepository.findAll();
        jdbcTemplate.update("UPDATE employee SET job_title = ''");
        String jobTitle = employeeRepository.findById(id).get().getJobTitle();
        log.info("Job title: {}", jobTitle);
    }
```

### [](#%D0%BF%D1%80%D0%B8%D0%BC%D0%B5%D1%80-8-%D0%BA%D0%B5%D1%88%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5-%D1%83%D1%80%D0%BE%D0%B2%D0%BD%D1%8F-%D0%B7%D0%B0%D0%BF%D1%80%D0%BE%D1%81%D0%B0)_Пример 8_. Кеширование уровня запроса

Для включения кеширования уровня запроса необходимо добавить в `application.yml` следующую конфигурацию [application.yml](/learning/hibernate_demo/-/blob/main/src/main/resources/application.yml):

```yml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_query_cache: true
          region:
            factory_class: org.hibernate.cache.ehcache.EhCacheRegionFactory
```

И на метод репозитория добавить аннотацию `@QueryHints`

[PeopleRepository.java](/learning/hibernate_demo/-/blob/main/src/main/java/ru/obteh/learning/hibernate/demo/repository/PeopleRepository.java):

```java
    @QueryHints(value = {
        @QueryHint(name = "org.hibernate.cacheable", value = "true")
    })
    List<EmployeeEntity> findAll();
```

### [](#%D0%BF%D1%80%D0%B8%D0%BC%D0%B5%D1%80-9-%D0%BF%D1%80%D0%BE%D0%B1%D0%BB%D0%B5%D0%BC%D0%B0-n1)_Пример 9_. Проблема N+1

Проблема N + 1 возникает, когда Hibernate выполняет N дополнительных SQL-запросов для получения тех же данных, которые можно получить при выполнении одного SQL-запроса.

[GrabliTest.java](/learning/hibernate_demo/-/blob/main/src/test/java/ru/obteh/learning/hibernate/demo/GrabliTest.java):

```java
    @Test
    void n_plus_1_problem() {
        // preparing
        peopleRepository.deleteAll();
        Set<PeopleEntity> friends = Set.of(
        peopleRepository.save(PeopleEntity.builder()
            .name("Tony Stark")
            .balance(99999999999999L)
            .build()),
        peopleRepository.save(PeopleEntity.builder()
            .name("Tyler Durden")
            .balance(100)
            .build()),
        peopleRepository.save(PeopleEntity.builder()
            .name("Tom Hardy")
            .balance(100)
            .build()));
    
        long id = peopleRepository.save(PeopleEntity.builder()
            .name("Andrey")
            .balance(0)
            .friends(friends)
            .build()).getId();
    
        // problem
        PeopleEntity me = peopleRepository.findById(id).get();
        log.info("friends: {}", Arrays.toString(me.getFriends().toArray())); // При вызове me.getFriends() выполняется целая куча запросов
    
        // solution
        me = peopleRepository.findByIdFetchFriends(id); // всего один запрос
        log.info("friends: {}", Arrays.toString(me.getFriends().toArray()));
    }
}
```

[PeopleEntity.java](/learning/hibernate_demo/-/blob/main/src/main/java/ru/obteh/learning/hibernate/demo/model/PeopleEntity.java):

```java
@Data
@Table(name = "people")
@Entity
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PeopleEntity {
    @Id
    @GeneratedValue(strategy = IDENTITY)
    private long id;
    private String name;
    private long balance;
    @ManyToMany
    private Set<PeopleEntity> friends;
    @OneToOne(mappedBy = "people")
    private EmployeeEntity employee;
    @OneToMany(mappedBy = "people")
    private Set<PurchaseEntity> purchaseEntity;
}
```

[PeopleRepository.java](/learning/hibernate_demo/-/blob/main/src/main/java/ru/obteh/learning/hibernate/demo/repository/PeopleRepository.java):

```java
@Repository
public interface PeopleRepository extends CrudRepository<PeopleEntity, Long> {

    @Query("SELECT p FROM PeopleEntity p WHERE p.id = :id")
    PeopleEntity findByIdFetchFriends(long id);
}
```

### [](#%D0%BF%D1%80%D0%B8%D0%BC%D0%B5%D1%80-10-%D0%B8%D1%81%D0%BF%D0%BE%D0%BB%D1%8C%D0%B7%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5-hibernate-jcache)_Пример 10_. Использование Hibernate JCache

`hibernate-jcache` является более предпочтительным, чем `hibernate-ehcache`

Для подключения `hibernate-jcache` нужно добавить следующие зависимости

[pom.xml](/learning/hibernate_demo/-/blob/main/pom.xml):

```xml
    <dependency>
        <groupId>org.ehcache</groupId>
        <artifactId>ehcache</artifactId>
    </dependency>
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-jcache</artifactId>
    </dependency>
```

Добавить в `application.yml` следующую конфигурацию

[application.yml](/learning/hibernate_demo/-/blob/main/src/main/resources/application.yml):

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true # Для включения кеширования второго уровня
          use_query_cache: true # Для включения кеширования уровня запросов
          region:
            factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
      javax:
        cache:
          provider: org.ehcache.jsr107.EhcacheCachingProvider
          uri: classpath:jcache.xml
```

Затем настроить JCache

[jcache.xml](/learning/hibernate_demo/-/blob/main/src/main/resources/jcache.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns="http://www.ehcache.org/v3"
        xmlns:jsr107="http://www.ehcache.org/v3/jsr107"
        xsi:schemaLocation="
            http://www.ehcache.org/v3 http://www.ehcache.org/schema/ehcache-core-3.0.xsd
            http://www.ehcache.org/v3/jsr107 http://www.ehcache.org/schema/ehcache-107-ext-3.0.xsd">

    <service>
        <jsr107:defaults enable-management="true" enable-statistics="true"/>
    </service>

    <cache-template name="default">
        <expiry>
            <ttl unit="minutes">2</ttl>
        </expiry>
        <heap>1024</heap>
    </cache-template>

    <cache alias="ru.obteh.learning.hibernate.demo.model.EmployeeEntity" uses-template="default">
        <expiry>
            <none/>
        </expiry>
    </cache>
    <cache alias="ru.obteh.learning.hibernate.demo.model.PeopleEntity" uses-template="default"/>
    <cache alias="ru.obteh.learning.hibernate.demo.model.PurchaseEntity" uses-template="default"/>

    <cache alias="default-query-results-region">
        <expiry>
            <ttl>30</ttl>
        </expiry>
        <heap>10</heap>
    </cache>
</config>
```

В данной конфигурации настраиваются **регионы** кеширования. Регион кэша — это логический разделитель памяти вашего кеша. Для каждого региона можно настроить свою политику кеширования

```xml
<cache-template name="default">
    <expiry>
        <ttl unit="minutes">2</ttl>
    </expiry>
    <heap>1024</heap>
</cache-template>
```

Эта запись обозначает создание шаблона кэша `default`. Он создается чтобы каждый раз не прописывать настройки для каждого региона. Шаблон описывает максимальное время жизни записи и максимальный размер региона

```xml
<cache alias="ru.obteh.learning.hibernate.demo.model.EmployeeEntity" uses-template="default">
    <expiry>
        <none/>
    </expiry>
</cache>
<cache alias="ru.obteh.learning.hibernate.demo.model.PeopleEntity" uses-template="default"/>
<cache alias="ru.obteh.learning.hibernate.demo.model.PurchaseEntity" uses-template="default"/>
```

Здесь мы создаем регионы кеширования, которые наследуют шаблон `default`. В hibernate названия регионов соответствуют полному названию entity с пакетом. В этом примере у EmployeeEntity переопределяется срок жизни кэша

Чтобы каждый раз не создавать регион для каждой Entity можно создать регион по-умолчанию и указывать ссылку на него в аннотации `@org.hibernate.annotations.Cache`

```xml
<cache alias="default-query-results-region">
    <expiry>
        <ttl>30</ttl>
    </expiry>
    <heap>10</heap>
</cache>
```

```java
@Data
@Table(name = "employee")
@Entity
@ToString(exclude = "people")
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_ONLY, region = "default-query-results-region")
public class EmployeeEntity {
    ...
}
```

## [](#%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D0%B8-%D0%BD%D0%B0-%D0%B4%D0%BE%D0%BA%D0%BB%D0%B0%D0%B4%D1%8B)Ссылки на доклады

- [Никита Летов — Используем @Transactional like a Pro](https://www.youtube.com/watch?v=QZ9rXZT0DlQ&t=471s)
- [Вячеслав Круглов — Введение в Hibernate: что, зачем, и где стандартные ловушки](https://www.youtube.com/watch?v=C-wEZjEOhWc&t=496s)
- [Николай Алименков — Сделаем Hibernate снова быстрым](https://www.youtube.com/watch?v=b52Qz6qlic0)
- [«HIBERNATE CACHING»: разбор всех уровней на примере чужих ошибок](https://www.youtube.com/watch?v=0s48OsEbIU0)