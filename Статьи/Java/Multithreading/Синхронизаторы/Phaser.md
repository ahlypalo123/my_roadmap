#### Phaser

  
Phaser (фазер), как и CyclicBarrier, является реализацией шаблона синхронизации [Барьер](https://ru.wikipedia.org/wiki/%D0%91%D0%B0%D1%80%D1%8C%D0%B5%D1%80%D0%BD%D0%B0%D1%8F_%D1%81%D0%B8%D0%BD%D1%85%D1%80%D0%BE%D0%BD%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D1%8F), но, в отличии от CyclicBarrier, предоставляет больше гибкости. Этот класс позволяет синхронизировать потоки, представляющие отдельную фазу или стадию выполнения общего действия. Как и CyclicBarrier, Phaser является точкой синхронизации, в которой встречаются потоки-участники. Когда все стороны прибыли, Phaser переходит к следующей фазе и снова ожидает ее завершения.  
  
Если сравнить Phaser и CyclicBarrier, то можно выделить следующие важные особенности Phaser:  
- Каждая фаза (цикл синхронизации) имеет номер;
- Количество сторон-участников жестко не задано и может меняться: поток может регистрироваться в качестве участника и отменять свое участие;
- Участник не обязан ожидать, пока все остальные участники соберутся на барьере. Чтобы продолжить свою работу достаточно сообщить о своем прибытии;
- Случайные свидетели могут следить за активностью в барьере;
- Поток может и не быть стороной-участником барьера, чтобы ожидать его преодоления;
- У фазера нет опционального действия.
  
Объект Phaser создается с помощью одного из конструкторов:  
  
```java
Phaser()
Phaser(int parties)
```
  
Параметр parties указывает на количество сторон-участников, которые будут выполнять фазы действия. Первый конструктор создает объект Phaser без каких-либо сторон, при этом барьер в этом случае тоже «закрыт». Второй конструктор регистрирует передаваемое в конструктор количество сторон. Барьер открывается когда все стороны прибыли, или, если снимается последний участник. (У класса Phaser еще есть конструкторы, в которые передается родительский объект Phaser, но мы их рассматривать не будем.)  
  
Основные методы:  
- **int register()** — регистрирует нового участника, который выполняет фазы. Возвращает номер текущей фазы;
- **int getPhase()** — возвращает номер текущей фазы;
- **int arriveAndAwaitAdvance()** — указывает что поток завершил выполнение фазы. Поток приостанавливается до момента, пока все остальные стороны не закончат выполнять данную фазу. Точный аналог `CyclicBarrier.await()`. Возвращает номер текущей фазы;
- **int arrive()** — сообщает, что сторона завершила фазу, и возвращает номер фазы. При вызове данного метода поток не приостанавливается, а продолжает выполнятся;
- **int arriveAndDeregister()** — сообщает о завершении всех фаз стороной и снимает ее с регистрации. Возвращает номер текущей фазы;
- **int awaitAdvance(int phase)** — если phase равно номеру текущей фазы, приостанавливает вызвавший его поток до её окончания. В противном случае сразу возвращает аргумент.
![](https://habrastorage.org/files/086/6a4/b7a/0866a4b7acdf416384d4e7372b49a34b.gif)  
[Официальная документация по Phaser.](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Phaser.html)  

**Пример использования Phaser**

Рассмотрим следующий пример. Есть пять остановок. На первых четырех из них могут стоять пассажиры и ждать автобуса. Автобус выезжает из парка и останавливается на каждой остановке на некоторое время. После конечной остановки автобус едет в парк. Нам нужно забрать пассажиров и высадить их на нужных остановках.  
```java
import java.util.ArrayList;
import java.util.concurrent.Phaser;

public class Bus {
    private static final Phaser PHASER = new Phaser(1);//Сразу регистрируем главный поток
    //Фазы 0 и 6 - это автобусный парк, 1 - 5 остановки

    public static void main(String[] args) throws InterruptedException {
        ArrayList<Passenger> passengers = new ArrayList<>();

        for (int i = 1; i < 5; i++) {           //Сгенерируем пассажиров на остановках
            if ((int) (Math.random() * 2) > 0)
                passengers.add(new Passenger(i, i + 1));//Этот пассажир выходит на следующей

            if ((int) (Math.random() * 2) > 0)
                passengers.add(new Passenger(i, 5));    //Этот пассажир выходит на конечной
        }

        for (int i = 0; i < 7; i++) {
            switch (i) {
                case 0:
                    System.out.println("Автобус выехал из парка.");
                    PHASER.arrive();//В фазе 0 всего 1 участник - автобус
                    break;
                case 6:
                    System.out.println("Автобус уехал в парк.");
                    PHASER.arriveAndDeregister();//Снимаем главный поток, ломаем барьер
                    break;
                default:
                    int currentBusStop = PHASER.getPhase();
                    System.out.println("Остановка № " + currentBusStop);

                    for (Passenger p : passengers)          //Проверяем, есть ли пассажиры на остановке
                        if (p.departure == currentBusStop) {
                            PHASER.register();//Регистрируем поток, который будет участвовать в фазах
                            p.start();        // и запускаем
                        }

                    PHASER.arriveAndAwaitAdvance();//Сообщаем о своей готовности
            }
        }
    }

    public static class Passenger extends Thread {
        private int departure;
        private int destination;

        public Passenger(int departure, int destination) {
            this.departure = departure;
            this.destination = destination;
            System.out.println(this + " ждёт на остановке № " + this.departure);
        }

        @Override
        public void run() {
            try {
                System.out.println(this + " сел в автобус.");

                while (PHASER.getPhase() < destination) //Пока автобус не приедет на нужную остановку(фазу)
                    PHASER.arriveAndAwaitAdvance();     //заявляем в каждой фазе о готовности и ждем

                Thread.sleep(1);
                System.out.println(this + " покинул автобус.");
                PHASER.arriveAndDeregister();   //Отменяем регистрацию на нужной фазе
            } catch (InterruptedException e) {
            }
        }

        @Override
        public String toString() {
            return "Пассажир{" + departure + " -> " + destination + '}';
        }
    }
}
```
  

**Результат работы программы**

Пассажир{1 -> 2} ждёт на остановке № 1  
Пассажир{1 -> 5} ждёт на остановке № 1  
Пассажир{2 -> 3} ждёт на остановке № 2  
Пассажир{2 -> 5} ждёт на остановке № 2  
Пассажир{3 -> 4} ждёт на остановке № 3  
Пассажир{4 -> 5} ждёт на остановке № 4  
Пассажир{4 -> 5} ждёт на остановке № 4  
Автобус выехал из парка.  
Остановка № 1  
Пассажир{1 -> 5} сел в автобус.  
Пассажир{1 -> 2} сел в автобус.  
Остановка № 2  
Пассажир{2 -> 3} сел в автобус.  
Пассажир{1 -> 2} покинул автобус.  
Пассажир{2 -> 5} сел в автобус.  
Остановка № 3  
Пассажир{2 -> 3} покинул автобус.  
Пассажир{3 -> 4} сел в автобус.  
Остановка № 4  
Пассажир{4 -> 5} сел в автобус.  
Пассажир{3 -> 4} покинул автобус.  
Пассажир{4 -> 5} сел в автобус.  
Остановка № 5  
Пассажир{1 -> 5} покинул автобус.  
Пассажир{2 -> 5} покинул автобус.  
Пассажир{4 -> 5} покинул автобус.  
Пассажир{4 -> 5} покинул автобус.  
Автобус уехал в парк.  

  

  
Кстати, функционалом фазера можно воспроизвести работу CountDownLatch.  

**Пример из CountDownLatch с использованием Phaser**

```java
import java.util.concurrent.Phaser;

public class NewRace {
    private static final Phaser START = new Phaser(8);
    private static final int trackLength = 500000;

    public static void main(String[] args) throws InterruptedException {
        for (int i = 1; i <= 5; i++) {
            new Thread(new Car(i, (int) (Math.random() * 100 + 50))).start();
            Thread.sleep(100);
        }

        while (START.getRegisteredParties() > 3) //Проверяем, собрались ли все автомобили
            Thread.sleep(100);                  //у стартовой прямой. Если нет, ждем 100ms

        Thread.sleep(100);
        System.out.println("На старт!");
        START.arriveAndDeregister();
        Thread.sleep(100);
        System.out.println("Внимание!");
        START.arriveAndDeregister();
        Thread.sleep(100);
        System.out.println("Марш!");
        START.arriveAndDeregister();
    }

    public static class Car implements Runnable {
        private int carNumber;
        private int carSpeed;

        public Car(int carNumber, int carSpeed) {
            this.carNumber = carNumber;
            this.carSpeed = carSpeed;
        }

        @Override
        public void run() {
            try {
                System.out.printf("Автомобиль №%d подъехал к стартовой прямой.\n", carNumber);
                START.arriveAndDeregister();
                START.awaitAdvance(0);
                Thread.sleep(trackLength / carSpeed);
                System.out.printf("Автомобиль №%d финишировал!\n", carNumber);
            } catch (InterruptedException e) {
            }
        }
    }
}
```
  

  
  

---

  
Если кому-нибудь пригодилось, то я очень рад=)  
  
Более подробно о Phaser [здесь](https://habrahabr.ru/post/117185/).  
Почитать ещё о синхронизаторах и посмотреть примеры можно [здесь](http://www.quizful.net/post/java-parallel-tools).  
  
Отличный обзор java.util.concurrent смотрите [здесь](https://habrahabr.ru/company/luxoft/blog/157273/).