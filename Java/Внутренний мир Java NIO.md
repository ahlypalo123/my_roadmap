---
title: "Внутренний мир: Java NIO"
source: "https://habr.com/ru/articles/801811/"
author:
  - "[[Хабр]]"
published: 2024-03-21
created: 2025-05-03
description: "Привет, Хабр! Парадигма «неблокируемого ввода/вывода» заинтересовала меня с того момента, как я о ней услышал. Идея возможности вызвать операцию чтения без блокировки вызывающего потока довольно..."
tags:
  - "clippings"
---
**Привет, Хабр!**

Парадигма «неблокируемого ввода/вывода» заинтересовала меня с того момента, как я о ней услышал. Идея возможности вызвать операцию чтения без блокировки вызывающего потока довольно привлекающая сама по себе.

Неблокируемый ввод/вывод был реализован в пакете `java.nio` Java SE 1.4. К сожалению, в ежедневной практике нечасто приходится иметь дело с низкоуровневым I/O, и намного чаще при необходимости используются стримы из `java.io`. В этой статье будет описано содержание Java NIO, несколько примеров и принцип работы неблокируемого I/O.

*Прим.: данная статья не является руководством по использованию или собранием best-practices. Она направлена в первую очередь на обзор существующих в Java NIO каналов и принцип работы неблокируемого I/O.*

![Поток ожидает чтения](https://habrastorage.org/r/w1560/getpro/habr/upload_files/d96/5c5/6c0/d965c56c00502c4c5c08f88e47d128d3.png)

Поток ожидает чтения

---

В начале статьи приведу [цитату](https://habr.com/ru/articles/235585/), описывающую разницу между подходами в Java IO и Java NIO:

> *“Основное отличие между двумя подходами к организации ввода/вывода в том, что Java IO является потокоориентированным, а Java NIO – буфер-ориентированным. Разберем подробней.*
> 
> *Потокоориентированный ввод/вывод подразумевает чтение/запись из потока/в поток одного или нескольких байт в единицу времени поочередно. Данная информация нигде не кэшируются. Таким образом, невозможно произвольно двигаться по потоку данных вперед или назад. Если вы хотите произвести подобные манипуляции, вам придется сначала кэшировать данные в буфере.*
> 
> *Подход, на котором основан Java NIO немного отличается. Данные считываются в буфер для последующей обработки. Вы можете двигаться по буферу вперед и назад. Это дает немного больше гибкости при обработке данных. В то же время, вам необходимо проверять содержит ли буфер необходимый для корректной обработки объем данных. Также необходимо следить, чтобы при чтении данных в буфер вы не уничтожили ещё не обработанные данные, находящиеся в буфере.”*

В Java NIO присутствуют три важные для его понимания сущности: `Buffer`, `Channel` и `Selector`. Рассмотрим их по порядку.

***Buffer*** – это контейнер для данных примитивного типа. Является более функциональной и удобной заменой массивам примитивов. В Java NIO используется как объект, который хранит фиксированный объем данных, подлежащих отправке или получению из службы ввода-вывода. Он находится между приложением и каналом, который записывает данные в буффер или считывает из него данные.

***Channel*** – связующее звено для операций ввода/вывода. Представляет собой открытое соединение с объектом, таким как аппаратное устройство, файл, сетевой сокет или программный компонент, который способен выполнять одну или несколько различных операций ввода-вывода, например чтение или запись. Остановимся на них поподробнее.

Java NIO имеет множество реализаций каналов. Ниже представлена иерархия интерфейсов.

![Рис. 1: Иерархия интерфейсов семейства Channel в Java NIO.](https://habrastorage.org/r/w1560/getpro/habr/upload_files/f03/c21/d11/f03c21d1163905f65ccfd62c4519b8f0.jpg)

Рис. 1: Иерархия интерфейсов семейства Channel в Java NIO.

Краткое описание особенностей интерфейсов семейства Channel.

- `Channel` – родительский класс для всего семейства
- `ReadableByteChannel` – канал, читающий байты из источника данных.
- `ScatteringByteChannel` – канал, читающий байты из источника данных в массив буфферов.
- `WritableByteChannel` – канал, записывающий байты в приемник данных.
- `GatheringByteChannel` – канал, записывающий байты в приемник данных из массива буфферов.
- `ByteChannel` – канал, который может как считывать, так и записывать данные. Интерфейс, объединяющий `ReadableByteChannel` и `WritableByteChannel`.
- `SeekableByteChannel` – канал, запоминающий текущую позицию чтения и имеющий возможность ее изменять. Иными словами, позволяет перемещаться по источнику данных.
- `AsynchronousChannel` – канал, поддерживающий асинхронные операции ввода/вывода.
- `AsynchronousByteChannel` – канал, поддерживающий асинхронные операции чтения/записи байт.
- `NetworkChannel` – канал, использующий сетевой сокет.
- `MulticastChannel` – канал, поддерживающий многоадресную рассылку по IP протоколу.
- `InterruptibleChannel` – канал, который возможно асинхронно закрыть и прервать операцию.

*Прим.: потоки из Java OI всегда являются однонаправленными: InputStream/OutputStream. Каналы из Java NIO могут быть двунаправленными.*

На следующей диаграмме представлена иерархия классов с абстрактными реализациями.

![Рис. 2: Иерархия интерфейсов и абстрактных реализаций семейства Channel в Java NIO.](https://habrastorage.org/r/w1560/getpro/habr/upload_files/2c8/750/4aa/2c87504aa064f19bf3ee2fc0073c3831.jpg)

Рис. 2: Иерархия интерфейсов и абстрактных реализаций семейства Channel в Java NIO.

На диаграмме интерфейсы (реализации) разделены на 3 группы:

- красные – блокирующие каналы;
- пурпурные – асинхронные каналы;
- зеленые – неблокирующие каналы.

Да-да, оказывается, далеко не все каналы из Java NIO являются неблокирующими! Возможно, именно поэтому NIO - не Non-blockable I/O, а именно New I/O. А еще можно заметить, что асинхронные и неблокирующие каналы имеют разные реализации (и концепции). Рассмотрим все три группы по порядку.

***Блокирующие каналы***

Данная группа каналов работает в стандартной парадигме – после вызова функции чтения/записи, вызывающий поток блокируется до выполнения операции. Главным отличием от стандартного IO тут является буфер-ориентированный подход.

Листинг 1: чтение файла с помощью канала из Java NIO

```java
void nio_blocking_readFile() throws IOException, URISyntaxException {
    URL fileUrl = NioTest.class.getResource(testFilePath);
    var filepath = Path.of(fileUrl.toURI());
  
    try (ReadableByteChannel inputChannel = FileChannel.open(filepath)) {
        var buffer = ByteBuffer.allocate(300_000);
        int readByteCount = inputChannel.read(buffer);
      
        var resultBytes = new byte[readByteCount];
        //Записываем считанные данные в resultBytes
        //Если просто вызвать buffer.array(), то если массив больше считываемого файла,
        //в конце он будет заполнен нулями
        buffer.get(0, resultBytes);
        var fileString = new String(resultBytes, StandardCharsets.UTF_8);
      
        System.out.println(fileString);
    }
}
```

Для сравнения, чтение файла с помощью потоков будет выглядеть следующим образом:

Листинг 2: чтение файла с помощью потока из Java IO

```java
void io_readFile() throws IOException {
    try (
        InputStream fileStream = NioTest.class.getResourceAsStream(testFilePath);
        var inputStream = new BufferedInputStream(fileStream)
    ) {
        byte[] fileBytes = inputStream.readAllBytes();
        var fileString = new String(fileBytes, StandardCharsets.UTF_8);
        System.out.println(fileString);
    }
}
```

Стоит отметить, что в листинге 1 мы не сможем полностью прочесть файл, если он больше размером, чем выделенный буффер. А так же, нам приходится "обрезать" буффер, что бы убрать лишние нули в нем.

Чтение файла в Java NIO

Из вышесказанного может показаться, что чтение файлов в Java NIO реализовано довольно неудобно. Однако в нем появился утилитный класс `Files`, который делает взаимодействие с файлами очень удобным. Вот пример, как с помощью него можно прочитать файл:

Листинг 2.1: чтение файла с помощью java.nio.file.Files

```java
void nio_readFile_files() throws IOException, URISyntaxException {
    URL fileUrl = NioTest.class.getResource(testFilePath);
    var filePath = Path.of(fileUrl.toURI());
    var fileString = Files.readString(path);
    System.out.println(fileString);
}
```

***Асинхронные каналы***

Данная группа каналов имеет возможность асинхронных операций чтения/записи. Есть возможность выполнить чтение/запись с каллбэком, или просто получить объект `Future`, а сама операция чтения/записи будет проходить в фоновом режиме.

Листинг 3: асинхронное чтение файла с Future

```java
void nio_async_readFile() throws URISyntaxException, IOException {
    URL fileUrl = NioTest.class.getResource(testFilePath);
    var path = Path.of(fileUrl.toURI());
  
    try (var inputChannel = AsynchronousFileChannel.open(path)) {
        var buffer = ByteBuffer.allocate(300_000);
        Future<Integer> futureResult = inputChannel.read(buffer, 0);
      
        while (!futureResult.isDone()) {
            System.out.println("Файл еще не загружен в буффер");
        }
      
        var fileString = new String(buffer.array(), StandardCharsets.UTF_8);
        System.out.println(fileString);
    }
}
```

Листинг 4: асинхронное чтение файла с использованием каллбэка

```java
void nio_async_readFile() throws URISyntaxException, IOException {
    URL fileUrl = NioTest.class.getResource(testFilePath);
    var path = Path.of(fileUrl.toURI());
  
    try (var inputChannel = AsynchronousFileChannel.open(path)) {
        var buffer = ByteBuffer.allocate(300_000);
        inputChannel.read(buffer, 0, buffer, new CompletionHandler<Integer, ByteBuffer>() {

            @Override
            public void completed(Integer result, ByteBuffer attachment) {
                var fileString = new String(buffer.array(), StandardCharsets.UTF_8);
                System.out.println(fileString);
            }

            @Override
            public void failed(Throwable exc, ByteBuffer attachment) {
                //do nothing
            }
        });
      
        try {
            Thread.sleep(3_000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
}
```

***Неблокирующие каналы***

Данная группа каналов может переключаться между блокирующим и неблокирующим режимом. Можно заметить, что все реализации неблокирующих каналов в Java NIO работают с сокетами.

В статье будут рассмотрены неблокируемые каналы на примере двух классов – `ServerSocketChannel` и `SocketChannel`.

#### ServerSockerChannel

Серверный сокет открывается командой `ServerSocketChannel.open()`. Созданный канал является открытым, но он не привязан к конкретному сокету. Что бы связать его с сокетом, необходимо вызвать `serverSocketChannel.socket().bind().`

По дефолту канал является блокирующим. Что бы перевести его в неблокирующий режим, нужно вызвать `serverSocketChannel.configureBlocking(false)`.

Ловим соединения через вызов `serverSocketChannel.accept()`. Если указан блокирующий режим, то вызывающий поток блокируется до момента, пока не примется соединение. В противном случае (включен неблокирующий режим), немедленно возвращается null, если нет ожидаемых подключений. Возвращаемые этим методом каналы всегда являются блокирующими, независимо от установленного типа канала (но их можно перевести в неблокируемый режим)..

Листинг 5: блокируемый сервер

```java
void nio_server_blockable() throws IOException {
    //Открытие канала. Под капотом вызывается SelectorProvider, реализация которого является платформозависимой
    var ssc = ServerSocketChannel.open();
    //Созданный канал является открытым, но не привязан к конкретному сокету. Что бы связать его с сокетом, необходимо вызвать код из следующей строки
    ssc.socket().bind(new InetSocketAddress(9999));
    //По дефолту канал является блокирующим. Что бы перевести его в неблокирующий режим, нужно в следующей строке передать false
    ssc.configureBlocking(true);
    var responseMessage = "Привет от сервера! : " + ssc.socket().getLocalSocketAddress();
    var sendBuffer = ByteBuffer.wrap(responseMessage.getBytes());
  
    while (true) {
        //Ловим соединения через вызов ssc.accept()
        //Поток блокируется до момента принятия соединения
        try (SocketChannel sc = ssc.accept()) {
            System.out.println("Принято соединение от  " + sc.socket().getRemoteSocketAddress());
            var receivedBuffer = ByteBuffer.allocate(100);
            sc.read(receivedBuffer);
            var requestMessage = new String(receivedBuffer.array());
            System.out.println(requestMessage);
          
            sendBuffer.rewind();
            sc.write(sendBuffer);
        }
    }
}
```

На строке 14 приложение встает в ожидании запроса на подключение.

Листинг 6: неблокируемый сервер

```java
void nio_server_non_blockable() throws IOException {
    var ssc = ServerSocketChannel.open();
    ssc.socket().bind(new InetSocketAddress(9999));
    //Включаем неблокирующий режим канала
    ssc.configureBlocking(false);
    var responseMessage = "Привет от сервера! : " + ssc.socket().getLocalSocketAddress();
    var sendBuffer = ByteBuffer.wrap(responseMessage.getBytes());
  
    while (true) {
        System.out.print(".");
        //Ловим соединения через вызов ssc.accept().
        //Т.к. стоит неблокирующий режим, метод accept немедленно вернет null, если нет ожидающих подключений
        try (SocketChannel sc = ssc.accept()) {
            if (sc != null) {
                System.out.println();
                System.out.println("Принято соединение от  " + sc.socket().getRemoteSocketAddress());
                var receivedBuffer = ByteBuffer.allocate(100);
                sc.read(receivedBuffer);
                var requestMessage = new String(receivedBuffer.array());
                System.out.println(requestMessage);
              
                sendBuffer.rewind();
                sc.write(sendBuffer);
            } else {
                Thread.sleep(100);
            }
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
}
```

В случае, если нет подключений в состоянии ожидания, на строке 13 возвращается `null.`

Как мы видим, «неблокируемость» не является какой то серебряной пулей. `ServerSocketChannel.accept()` просто-напросто не ждет подключения, а немедленно возвращает `null`.

#### SocketChannel

Клиентский сокет открывается командой `SocketChannel.open()`. Если мы включаем неблокируемый режим, то происходит почти то же самое, что и с `ServerSocketChannel` – канал не ждет появления «отправляемых» или «получаемых» данных и просто продвигается дальше. Однако, если данные есть, поток блокируется и канал считывает их. Фактически, если канал еще не готов к чтению или записи на момент вызова операции, мы просто пропускаем эту операцию. В примерах используется сервер, написанный в листинге 6.

Листинг 7: блокируемый клиент

```java
void nio_client_blockable() throws IOException {
    try (SocketChannel sc = SocketChannel.open()) {
        sc.configureBlocking(true);
        sc.connect(new InetSocketAddress("localhost", 9999));
      
        var requestMessage = "Привет от клиента! " + LocalDateTime.now();
        ByteBuffer buffer = ByteBuffer.wrap(requestMessage.getBytes());
        sc.write(buffer);
      
        var receivedBuffer = ByteBuffer.allocate(100);
        //Приложение останавливается в ожидании ответа
        sc.read(receivedBuffer);
        var responseMessage = new String(receivedBuffer.array());
        System.out.println(responseMessage);
    }
}
```

На 10 строке приложение останавливается в ожидании ответа.

Листинг 8: неблокирующий клиент

```java
void nio_client_non_blockable() throws IOException {
    try (SocketChannel sc = SocketChannel.open()) {
        //Включаем неблокирующий режим канала
        sc.configureBlocking(false);
        sc.connect(new InetSocketAddress("localhost", 9999));
      
        while (!sc.finishConnect()) {
            System.out.println("waiting to finish connection");
        }
      
        var requestMessage = "Привет от клиента! " + LocalDateTime.now();
        ByteBuffer buffer = ByteBuffer.wrap(requestMessage.getBytes());
        sc.write(buffer);
      
        var receivedBuffer = ByteBuffer.allocate(100);
        //Ответа еще нет, канал ничего не прочтет, буффер останется пустым
        int receiveReadCount = sc.read(receivedBuffer);
        Thread.sleep(1_000);

        var resultBytes = new byte[receiveReadCount];
        buffer.get(0, resultBytes);
        var responseMessage = new String(resultBytes, StandardCharsets.UTF_8);
      
        //Консоль выведет пустую строку
        System.out.println(responseMessage);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
}
```

На строке 17 клиент ничего не читает из сокета, поскольку ответа еще нет. Операция чтения пропускается. Можно заметить, что в фоновом режиме ничего не читается, поскольку в консоль выводится пустая строка, несмотря на ожидание после вызова операции чтения.

Листинг 9: неблокируемый клиент, ожидающий ответа

```java
void nio_client_non_blockable() throws IOException {
    try (SocketChannel sc = SocketChannel.open()) {
        sc.configureBlocking(false);
        sc.connect(new InetSocketAddress("localhost", 9999));

        while (!sc.finishConnect()) {
            System.out.println("waiting to finish connection");
        }

        ByteBuffer buffer = ByteBuffer.wrap(("Привет от клиента! " + LocalDateTime.now()).getBytes());
        sc.write(buffer);
        Thread.sleep(1_000);

        var receivedBuffer = ByteBuffer.allocate(100);
        int receiveReadCount = sc.read(receivedBuffer);
      
        var resultBytes = new byte[receiveReadCount];
        buffer.get(0, resultBytes);
        var responseMessage = new String(resultBytes, StandardCharsets.UTF_8);
      
        //Консоль выведет ответ
        System.out.println(responseMessage);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
}
```

Поскольку мы установили задержку в секунду между записью запроса и чтением ответа, к моменту вызова операции чтения канал уже имеет данные для чтения, и в консоль выведется ответ.

На первый взгляд, «неблокируемость» выглядит не в лучше свете – мы просто не находимся в состоянии ожидания, если операция еще не готова к выполнению, и возвращаем управление вызывающей функции. Операция пропускается, и мы не получаем (не записываем) никаких данных.

*Прим.: тут можно заметить разницу между «асинхронностью» и «неблокируемостью». При асинхронном подходе приложение все равно ожидает данные, но в фоновом потоке (т.е. поток все равно находится в состоянии ожидания, но не вызывающий), в то время как в неблокируемом подходе мы пропускаем операцию, если она не готова к выполнению (сокет не готов принимать данные, или сокет не имеет данных для чтения).*

Появляется вопрос – что же нам дает такая «неблокируемость»? Вряд ли кого то устроит пропуск операции, если канал еще не готов отдать данные. Для ответа на этот вопрос нам нужно разобраться с классом `Selector`.

#### Selector

`Selector` – это объект, относящийся к группе каналов и определяющий, какой канал готов к записи/чтению/подключению и т.д. Он позволяет одному потоку управлять несколькими каналами (подключениями). Это позволяет уменьшить траты на переключения между потоками.

Я не буду описывать методы рассматриваемых классов, они прекрасно описаны в документации. Но постараюсь описать процесс взаимодействия с селектором и его использования.

После открытия селектора в нем необходимо зарегистрировать используемый канал. *Для использования с селектором можно использовать только неблокируемые каналы. Т.е. мы не сможем зарегистрировать, например,* `FileChannel`*.* Канал сам проверяет, поддерживает ли он переданные для наблюдения операции и регистрирует свой `SelectionKey` в селекторе (по сути, селектор добавляет ключ в список наблюдаемых объектов). `Selector.select()` возвращает количество готовых к использованию каналов. Как он понимает, сколько каналов готовы к использованию? `SelectionKey` содержит ссылку на канал. Селектор проходит по каждому каналу и опрашивает, готов ли он к записи/чтению/и т.д., и считает количество таких каналов.

Далее, если есть каналы, ожидающие обработки, мы вытаскиваем множество готовых `SelectedKey` и обрабатываем их. Кроме этого, есть возможность напрямую передать каллбэк в функцию `select()`, и селектор применит его к готовым для взаимодействия каналам.

Для понимания доступных каналу операций существуют обозначающие их константы. Экземпляр `SelectionKey` содержит готовые к выполнению операции в канале. Ниже представлен список доступных каналам операций. Они могут сочетаться любым образом.

Доступные операции

| OP\_READ | Канал имеет данные, доступные для чтения |
| --- | --- |
| OP\_WRITE | Канал доступен для записи |
| OP\_CONNECT | Канал готов завершить подключение или ожидает сообщение об ошибке |
| OP\_ACCEPT | Канал готов к приему подключения (только для `ServerSocketChannel`) |

Листинг 10: реализация неблокируемого сервера с использованием селектора

```java
void nio_non_blockable_selector_server() throws IOException {
    try (ServerSocketChannel channel = ServerSocketChannel.open();
         //Открытие селектора. Под капотом вызывается SelectorProvider, реализация которого является платформозависимой
         Selector selector = Selector.open()) {
        channel.socket().bind(new InetSocketAddress(9999));
        channel.configureBlocking(false);
        //Регистрируем серверный канал в селекторе с интересующим типом операции - принятие подключения
        SelectionKey registeredKey = channel.register(selector, SelectionKey.OP_ACCEPT);

        while (true) {
            //Получаем количество готовых к обработке каналов.
            int numReadyChannels = selector.select();
            if (numReadyChannels == 0) {
                continue;
            }
            //Получаем готовые к обработке каналы
            Set<SelectionKey> selectedKeys = selector.selectedKeys();
            Iterator<SelectionKey> keyIterator = selectedKeys.iterator();

            //Обрабатываем каналы в соответствии с типом доступной каналу операции
            while (keyIterator.hasNext()) {
                SelectionKey key = keyIterator.next();

                if (key.isAcceptable()) {
                    //Принятие подключения серверным сокетом
                    ServerSocketChannel server = (ServerSocketChannel) key.channel();
                    SocketChannel client = server.accept();
                    if (client == null) {
                        continue;
                    }
                    client.configureBlocking(false);
                    //Регистрируем принятое подключение в селекторе с интересующим типом операции - чтение
                    client.register(selector, SelectionKey.OP_READ);
                }

                if (key.isReadable()) {
                    //Тут происходит обработка принятых подключений
                    SocketChannel client = (SocketChannel) key.channel();
                    ByteBuffer requestBuffer = ByteBuffer.allocate(100);
                    int r = client.read(requestBuffer);
                    if (r == -1) {
                        client.close();
                    } else {
                        //В этом блоке происходит обработка запроса
                        System.out.println(new String(requestBuffer.array()));
                        String responseMessage = "Привет от сервера! : " + client.socket().getLocalSocketAddress();
                        //Несмотря на то, что интересующая операция, переданная в селектор - чтение, мы все равно можем писать в сокет
                        client.write(ByteBuffer.wrap(responseMessage.getBytes()));
                    }
                }
                //Удаляем ключ после обработки. Если канал снова будет доступным, его ключ снова появится в selectedKeys
                keyIterator.remove();
            }
        }
    }
}
```

Листинг 11: реализация неблокируемого сервера с использованием селектора и регистрацией каллбэка

```java
void nio_non_blockable_selector_server() throws IOException {
    try (ServerSocketChannel channel = ServerSocketChannel.open();
         Selector selector = Selector.open()) {
        channel.socket().bind(new InetSocketAddress(9999));
        channel.configureBlocking(false);
        SelectionKey registeredKey = channel.register(selector, SelectionKey.OP_ACCEPT);

        while (true) {
            //Обрабатываем доступные к ожиданию подключения с использованием каллбэка
            selector.select(key -> {

                if (key.isAcceptable()) {
                    try {
                        //Принятие подключения серверным сокетом
                        ServerSocketChannel server = (ServerSocketChannel) key.channel();
                        SocketChannel client = server.accept();
                        client.configureBlocking(false);
                        //Регистрируем принятое подключение в селекторе с интересующим типом операции - чтение
                        client.register(selector, SelectionKey.OP_READ);
                    } catch (IOException e) {
                        throw new RuntimeException(e);
                    }
                }

                if (key.isReadable()) {
                    try {
                        //Тут происходит обработка принятых подключений
                        SocketChannel client = (SocketChannel) key.channel();
                        ByteBuffer requestBuffer = ByteBuffer.allocate(100);
                        int r = client.read(requestBuffer);
                        if (r == -1) {
                            client.close();
                        } else {
                            //В этом блоке происходит обработка запроса
                            System.out.println(new String(requestBuffer.array()));
                            String responseMessage = "Привет от сервера! : " + client.socket().getLocalSocketAddress();
                            //Несмотря на то, что интересующая операция, переданная в селектор - чтение, мы все равно можем писать в сокет
                            client.write(ByteBuffer.wrap(responseMessage.getBytes()));
                        }
                    } catch (IOException e) {
                        throw new RuntimeException(e);
                    }
                }
            });
        }
    }
}
```

Примерно так же работает и неблокирующий клиент: мы отправляем сообщение и вместо ожидания ответа регистрируем его в селекторе. При этом мы можем создать отдельный поток, который проверяет селектор и обрабатывает готовые к работе потоки.

Листинг 12 – неблокируемый клиент с использованием селектора и обработкой ответов в отдельном потоке

```java
void nio_clientSocket_non_blockable_selector_1() throws IOException {
    try (SocketChannel sc = SocketChannel.open();
         Selector selector = Selector.open()) {
        sc.configureBlocking(false);
        //Регистрируем канал в селекторе с интересующим типом операции - чтение
        SelectionKey registeredKey = sc.register(selector, SelectionKey.OP_READ);

        //Создаем поток, который будет опрашивать селектор и обрабатывать ответы на наши запросы
        var selectorThread = new Thread(() -> {
            while (true) {
                try {
                    int numReadyChannels = selector.select();
                    if (numReadyChannels == 0) {
                        continue;
                    }

                    Set<SelectionKey> selectionKeys = selector.selectedKeys();
                    Iterator<SelectionKey> keyIterator = selectionKeys.iterator();
                    
                    while (keyIterator.hasNext()) {
                        SelectionKey key = keyIterator.next();
                        if (key.isReadable()) {
                            //Этот тот канал, который мы открыли в начале функции
                            //Мы отловили его дя чтения ответа
                            SocketChannel client = (SocketChannel) key.channel();
                            var received = ByteBuffer.allocate(100);
                            client.read(received);
                            System.out.println(new String(received.array()));
                        }
                        keyIterator.remove();
                    }
                } catch (IOException e) {
                    throw new RuntimeException(e);
                }
            }
        });
        selectorThread.setDaemon(true);
        selectorThread.start();

        sc.connect(new InetSocketAddress("localhost", 9999));
        while (!sc.finishConnect()) {
            System.out.println("waiting to finish connection");
        }
        
        String requestMessage = "Привет от клиента! " + LocalDateTime.now();
        ByteBuffer requestBuffer = ByteBuffer.wrap(requestMessage.getBytes());
        sc.write(requestBuffer);

        Thread.sleep(2000);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
}
```

Таким образом, основной поток не блокируется при ожидании ответа. Однако, мы можем создать отдельный ограниченный пулл потоков, которые будут обрабатывать наши селекторы и передавать обработку полученных запросов/ответов в исполняющие потоки.