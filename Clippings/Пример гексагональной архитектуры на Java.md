---
title: "Пример гексагональной архитектуры на Java"
source: "https://habr.com/ru/companies/otus/articles/492076/"
author:
  - "[[Хабр]]"
published: 2020-03-12
created: 2025-05-03
description: "Перевод статьи подготовлен специально для студентов курса «Разработчик Java» . Как разработчикам нам часто приходится сталкиваться с легаси кодом, который тяжело поддерживать. Вы знаете как бывает..."
tags:
  - "clippings"
---
[Автор оригинала: Animesh Gaitonde](https://itnext.io/hexagonal-architecture-principles-practical-example-in-java-364bb2e50075)

***Перевод статьи подготовлен специально для студентов курса [«Разработчик Java»](https://otus.pw/vUX0/).***  
  
![](https://habrastorage.org/r/w1560/webt/gd/2f/ap/gd2fapnfvwthsglrmjmhnn1wyvk.png)  
  

---

  
Как разработчикам нам часто приходится сталкиваться с легаси кодом, который тяжело поддерживать. Вы знаете как бывает сложно понять простую логику в большом запутанном спагетти-коде. Улучшение кода или разработка новой функциональности становятся ночным кошмаром для разработчика.  
  
Одна из основных целей проектирования программного обеспечения — это удобство сопровождения. Код, который плохо поддерживать, становится сложными в управлении. Его не только трудно масштабировать, но становится проблемой привлечь новых разработчиков.  
  
В мире ИТ все движется быстрыми темпами. Если вас попросят срочно реализовать новую функциональность или вы захотите перейти с реляционной базы данных на NoSQL, то какая будет ваша первая реакция?  
  
![](https://habrastorage.org/webt/wk/lj/8b/wklj8bigqatvwl-qo7cjlapjl_8.gif)  
  
Хорошее тестовое покрытие повышает уверенность разработчиков в том, что с новым релизом не будет проблем. Однако если ваша бизнес-логика переплетена с инфраструктурной логикой, то с ее тестированием могут быть проблемы.  
  
![](https://habrastorage.org/webt/mo/vc/hh/movchhizk5zg4m56jhtnkdsrvdy.gif)  
*Почему я?*  
  
Но хватит пустой болтовни, давайте посмотрим на гексагональную архитектуру. Использование этого шаблона поможет вам улучшить сопровождаемость, тестируемость и получить другие преимущества.  
  

### Введение в гексагональную архитектуру

  
Термин *Hexagonal Architecture (гексагональная, шестиугольная архитектура)* придумал в 2006 году Алистер Коберн (Alistair Cockburn). Этот архитектурный стиль также известен как *Ports And Adapters Architecture (архитектура портов и адаптеров)*. Говоря простыми словами, компоненты вашего приложения взаимодействуют через множество конечных точек (портов). Для обработки запросов у вас должны быть адаптеры, соответствующие портам.  
  
Здесь можно провести аналогию с USB-портами на компьютере. Вы можете ими воспользоваться, если у вас есть совместимый адаптер (зарядное устройство или флешка).  
  
![](https://habrastorage.org/r/w1560/webt/tn/om/vd/tnomvd4i3cngipfc8olfqpyec9g.jpeg)  
  
Эту архитектуру можно схематически представить в виде шестиугольника с бизнес-логикой в самом центре (в ядре), окруженную объектами с которыми она взаимодействует, и компонентами, которые ею управляют, предоставляя входные данные.  
  
В реальной жизни с вашим приложением взаимодействуют и предоставляют входные данные пользователи, вызовы API, автоматизированные скрипты и модульное тестирование. Если ваша бизнес-логика смешана с логикой пользовательского интерфейса, то вы столкнетесь с многочисленными проблемами. Например, будет сложно переключить ввод данных с пользовательского интерфейса на модульные тесты.  
  
Также приложение взаимодействует с внешними объектами, такими как базы данных, очереди сообщений, веб-серверы (через вызовы HTTP API) и т. д. При необходимости мигрировать базу данных или выгрузить данные в файл у вас должна быть возможность это сделать, не затрагивая бизнес-логику.  
  
Как следует из названия “ **порты и адаптеры** ”, есть ”порты”, через которые происходит взаимодействие и “адаптеры” — компоненты, обрабатывающие пользовательский ввод и преобразующие его на “язык” домена. Адаптеры инкапсулируют логику взаимодействия с внешними системами, такими как базы данных, очереди сообщений и др., облегчают связь между бизнес-логикой и внешними объектами.  
  
На диаграмме ниже показаны слои, на которые разделено приложение.  
  
![](https://habrastorage.org/r/w1560/webt/yu/zd/k-/yuzdk-bppoignfpz98ck7ld-a4q.jpeg)  
  
Гексагональная архитектура выделяет в приложении три слоя: домен (domain), приложение (application) и инфраструктура (framework):  
  
- **Домен**. Слой содержит основную бизнес-логику. Он не должен знать детали реализации внешних слоев.
- **Приложение**. Слой действует как связующее звено между слоями домена и инфраструктуры.
- **Инфраструктура**. Реализация взаимодействия домена с внешним миром. Внутренние слои выглядят для него как черный ящик.
  
Согласно этой архитектуре, с приложением взаимодействуют два типа участников: основные (driver) и вторичные (driven). Основные действующие лица отправляют запросы и управляют приложением (например, пользователи или автоматизированные тесты). Вторичные обеспечивают инфраструктуру для связи с внешним миром (это адаптеры базы данных, TCP- или HTTP-клиенты).  
Это можно представить так:  
  
![](https://habrastorage.org/r/w1560/webt/9j/yd/l-/9jydl-rnrtzuy8dk23gexjitq_g.jpeg)  
  
Левая сторона шестиугольника состоит из компонент, обеспечивающих ввод для домена (они “управляют” приложением), а правая — из компонент, которые управляются нашим приложением.  
  

#### Пример

  
Давайте спроектируем приложение, которое будет хранить обзоры фильмов. У пользователя должна быть возможность отправить запрос с названием фильма и получить пять случайных отзывов.  
  
Для простоты сделаем консольное приложение с хранением данных в оперативной памяти. Ответ пользователю будем выводить на консоль.  
  
У нас есть пользователь (User), который отправляет запрос приложению. Таким образом, пользователь становится “управляющим” (driver). Приложение должно уметь получать данные из любого типа хранилища и выводить результаты на консоль или в файл. Управляемыми (driven) объектами будут “хранилище данных” (`IFetchMovieReviews`) и “принтер ответов” (`IPrintMovieReviews`).  
  
На следующем рисунке показаны основные компоненты нашего приложения.  
  
![](https://habrastorage.org/r/w1560/webt/yo/kz/cj/yokzcjkqbwbvklf1bcxgqnhfx74.jpeg)  
  
Слева находятся компоненты, которые обеспечивают ввод данных в приложение. Справа — компоненты, которые позволяют взаимодействовать с базой данных и консолью.  
  
Давайте рассмотрим код приложения.  
  
**Управляющий порт**  
  
```java
public interface IUserInput {
    public void handleUserInput(Object userCommand);
}
```
  
**Управляемые порты**  
  
```java
public interface IFetchMovieReviews {
    public List<MovieReview> fetchMovieReviews(MovieSearchRequest movieSearchRequest);
}

public interface IPrintMovieReviews {
    public void writeMovieReviews(List<MovieReview> movieReviewList);
}
```
  
**Адаптеры управляемого порта**  
  
Фильмы будем получить из репозитория фильмов (MovieReviewsRepo). Выводить обзоры фильмов на консоль будет класс `ConsolePrinter`. Давайте реализуем два вышеупомянутых интерфейса.  
  
```java
public class ConsolePrinter implements IPrintMovieReviews {
    @Override
    public void writeMovieReviews(List<MovieReview> movieReviewList) {
        movieReviewList.forEach(movieReview -> {
            System.out.println(movieReview.toString());
        });
    }
}

public class MovieReviewsRepo implements IFetchMovieReviews {
    private Map<String, List<MovieReview>> movieReviewMap;

    public MovieReviewsRepo() {
        initialize();
    }

    public List<MovieReview> fetchMovieReviews(MovieSearchRequest movieSearchRequest) {

        return Optional.ofNullable(movieReviewMap.get(movieSearchRequest.getMovieName()))
            .orElse(new ArrayList<>());
    }

    private void initialize() {
        this.movieReviewMap = new HashMap<>();
        movieReviewMap.put("StarWars", Collections.singletonList(new MovieReview("1", 7.5, "Good")));
        movieReviewMap.put("StarTreck", Arrays.asList(new MovieReview("1", 9.5, "Excellent"), new MovieReview("1", 8.5, "Good")));
    }
}
```
  

### Домен

  
Основная задача нашего приложения — обрабатывать запросы пользователей. Необходимо получить фильмы, обработать их и передать результаты “принтеру”. На данный момент у нас есть только одна функциональность — поиск фильмов. Для обработки пользовательских запросов будем использовать стандартный интерфейс `Consumer`.  
  
Давайте посмотрим на основной класс `MovieApp`.  
  
```java
public class MovieApp implements Consumer<MovieSearchRequest> {
    private IFetchMovieReviews fetchMovieReviews;
    private IPrintMovieReviews printMovieReviews;
    private static Random rand = new Random();

    public MovieApp(IFetchMovieReviews fetchMovieReviews, IPrintMovieReviews printMovieReviews) {
        this.fetchMovieReviews = fetchMovieReviews;
        this.printMovieReviews = printMovieReviews;
    }

    private List<MovieReview> filterRandomReviews(List<MovieReview> movieReviewList) {
        List<MovieReview> result = new ArrayList<MovieReview>();
        // logic to return random reviews
        for (int index = 0; index < 5; ++index) {
            if (movieReviewList.size() < 1)
                break;
            int randomIndex = getRandomElement(movieReviewList.size());
            MovieReview movieReview = movieReviewList.get(randomIndex);
            movieReviewList.remove(movieReview);
            result.add(movieReview);
        }
        return result;
    }

    private int getRandomElement(int size) {
        return rand.nextInt(size);
    }

    public void accept(MovieSearchRequest movieSearchRequest) {
        List<MovieReview> movieReviewList = fetchMovieReviews.fetchMovieReviews(movieSearchRequest);
        List<MovieReview> randomReviews = filterRandomReviews(new ArrayList<>(movieReviewList));
        printMovieReviews.writeMovieReviews(randomReviews);
    }
}
```
  
Теперь определим класс `CommandMapperModel`, который будет сопоставлять команды с обработчиками.  
  
```java
public class CommandMapperModel {
    private static final Class<MovieSearchRequest> searchMovies = MovieSearchRequest.class;

    public static Model build(Consumer<MovieSearchRequest> displayMovies) {
        Model model = Model.builder()
            .user(searchMovies)
            .system(displayMovies)
            .build();

        return model;
    }
}
```
  

### Адаптеры управляющего порта

  
Пользователь будет взаимодействовать с нашей системой через интерфейс `IUserInput`. Реализация будет использовать `ModelRunner` и делегировать выполнение.  
  
```java
public class UserCommandBoundary implements IUserInput {
    private Model model;

    public UserCommandBoundary(IFetchMovieReviews fetchMovieReviews, IPrintMovieReviews printMovieReviews) {
        MovieApp movieApp = new MovieApp(fetchMovieReviews, printMovieReviews);
        model = CommandMapperModel.build(movieApp);
    }

    public void handleUserInput(Object userCommand) {
        new ModelRunner().run(model)
            .reactTo(userCommand);
    }
}
```
  
Теперь посмотрим на пользователя, который использует вышеупомянутый интерфейс.  
  
```java
public class MovieUser {
    private IUserInput userInputDriverPort;

    public MovieUser(IUserInput userInputDriverPort) {
        this.userInputDriverPort = userInputDriverPort;
    }

    public void processInput(MovieSearchRequest movieSearchRequest) {
        userInputDriverPort.handleUserInput(movieSearchRequest);
    }
}
```
  

#### Приложение

  
Далее, создадим консольное приложение. Управляемые адаптеры добавляем в качестве зависимостей. Пользователь будет создавать и отправлять запрос в приложение. Приложение будет получать данные, обрабатывать и выводить ответ на консоль.  
  
```java
public class Main {

    public static void main(String[] args) {
        IFetchMovieReviews fetchMovieReviews = new MovieReviewsRepo();
        IPrintMovieReviews printMovieReviews = new ConsolePrinter();
        IUserInput userCommandBoundary = new UserCommandBoundary(fetchMovieReviews, printMovieReviews);
        MovieUser movieUser = new MovieUser(userCommandBoundary);
        MovieSearchRequest starWarsRequest = new MovieSearchRequest("StarWars");
        MovieSearchRequest starTreckRequest = new MovieSearchRequest("StarTreck");

        System.out.println("Displaying reviews for movie " + starTreckRequest.getMovieName());
        movieUser.processInput(starTreckRequest);
        System.out.println("Displaying reviews for movie " + starWarsRequest.getMovieName());
        movieUser.processInput(starWarsRequest);
    }

}
```
  

### Что можно улучшить, изменить

  
- В нашей реализации можно легко переключиться с одного хранилища данных на другое. Реализация хранилища может быть внедрена (inject) в код без изменения бизнес-логики. Например, можно перенести данные из памяти в базу данных, написав адаптер базы данных.
- Вместо вывода на консоль можно реализовать “принтер”, который будет записывать данные в файл. В таком многослойном приложении становится проще добавлять функциональность и исправлять ошибки.
- Для проверки бизнес-логики можно написать комплексные тесты. Адаптеры можно тестировать изолированно. Таким образом, можно увеличить общее тестовое покрытие.
  

### Заключение

  
Можно отметить следующие преимущества гексагональной архитектуры:  
  
- **Сопровождаемость** — слабосвязанные и независимые слои. Становится легко добавлять новые функции в один слой, не затрагивая других слоев.
- **Тестируемость** — модульные тесты пишутся просто и быстро выполняются. Можно написать тесты для каждого слоя с использованием объектов-заглушек, имитирующих зависимости. Например, мы можем убрать зависимость от базы данных, сделав хранилище данных в памяти.
- **Адаптивность** — основная бизнес-логика становится независимой от изменений во внешних объектах. Например, если потребуется мигрировать на другую базу данных то, нам не нужно вносить изменения в домен. Мы можем сделать соответствующий адаптер для базы данных.
  

### Ссылки

  
- [Hexagonal Architecture (Alistair Cockburn)](https://alistair.cockburn.us/hexagonal-architecture/)
- [Hexagonal Architecture-Wiki](https://wiki.c2.com/?HexagonalArchitecture)
- [Ports And Adapters Architecture](https://softwarecampament.wordpress.com/portsadapters/)
- [Giphy](https://giphy.com/)
  
На этом все. До встречи [на курсе](https://otus.pw/vUX0/)!