#### Exchanger

  
Exchanger (обменник) может понадобиться, для того, чтобы обменяться данными между двумя потоками в определенной точки работы обоих потоков. Обменник — обобщенный класс, он параметризируется типом объекта для передачи.  
  
![](https://habrastorage.org/files/947/ef3/f47/947ef3f47ff843a099059006b30ea54d.gif)  
  
Обменник является точкой синхронизации пары потоков: поток, вызывающий у обменника метод `exchange()` блокируется и ждет другой поток. Когда другой поток вызовет тот же метод, произойдет обмен объектами: каждая из них получит аргумент другой в методе `exchange()`. Стоит отметить, что обменник поддерживает передачу `null` значения. Это дает возможность использовать его для передачи объекта в одну сторону, или, просто как точку синхронизации двух потоков.  
  
[Официальная документация по Exchanger.](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Exchanger.html)  
  

**Пример использования Exchanger**

Рассмотрим следующий пример. Есть два грузовика: один едет из пункта A в пункт D, другой из пункта B в пункт С. Дороги AD и BC пересекаются в пункте E. Из пунктов A и B нужно доставить посылки в пункты C и D. Для этого грузовики в пункте E должны встретиться и обменяться соответствующими посылками.  
  
```java
import java.util.concurrent.Exchanger;

public class Delivery {
    //Создаем обменник, который будет обмениваться типом String
    private static final Exchanger<String> EXCHANGER = new Exchanger<>();

    public static void main(String[] args) throws InterruptedException {
        String[] p1 = new String[]{"{посылка A->D}", "{посылка A->C}"};//Формируем груз для 1-го грузовика
        String[] p2 = new String[]{"{посылка B->C}", "{посылка B->D}"};//Формируем груз для 2-го грузовика
        new Thread(new Truck(1, "A", "D", p1)).start();//Отправляем 1-й грузовик из А в D
        Thread.sleep(100);
        new Thread(new Truck(2, "B", "C", p2)).start();//Отправляем 2-й грузовик из В в С
    }

    public static class Truck implements Runnable {
        private int number;
        private String dep;
        private String dest;
        private String[] parcels;

        public Truck(int number, String departure, String destination, String[] parcels) {
            this.number = number;
            this.dep = departure;
            this.dest = destination;
            this.parcels = parcels;
        }

        @Override
        public void run() {
            try {
                System.out.printf("В грузовик №%d погрузили: %s и %s.\n", number, parcels[0], parcels[1]);
                System.out.printf("Грузовик №%d выехал из пункта %s в пункт %s.\n", number, dep, dest);
                Thread.sleep(1000 + (long) Math.random() * 5000);
                System.out.printf("Грузовик №%d приехал в пункт Е.\n", number);
                parcels[1] = EXCHANGER.exchange(parcels[1]);//При вызове exchange() поток блокируется и ждет
                //пока другой поток вызовет exchange(), после этого произойдет обмен посылками
                System.out.printf("В грузовик №%d переместили посылку для пункта %s.\n", number, dest);
                Thread.sleep(1000 + (long) Math.random() * 5000);
                System.out.printf("Грузовик №%d приехал в %s и доставил: %s и %s.\n", number, dest, parcels[0], parcels[1]);
            } catch (InterruptedException e) {
            }
        }
    }
}
```
  

**Результат работы программы**

В грузовик №1 погрузили: {посылка A->D} и {посылка A->C}.  
Грузовик №1 выехал из пункта A в пункт D.  
В грузовик №2 погрузили: {посылка B->C} и {посылка B->D}.  
Грузовик №2 выехал из пункта B в пункт C.  
Грузовик №1 приехал в пункт Е.  
Грузовик №2 приехал в пункт Е.  
В грузовик №2 переместили посылку для пункта C.  
В грузовик №1 переместили посылку для пункта D.  
Грузовик №2 приехал в C и доставил: {посылка B->C} и {посылка A->C}.  
Грузовик №1 приехал в D и доставил: {посылка A->D} и {посылка B->D}.  

  
Как мы видим, когда один грузовик (один поток) приезжает в пункт Е (достигает точки синхронизации), он ждет пока другой грузовик (другой поток) приедет в пункт Е (достигнет точки синхронизации). После этого происходит обмен посылками (String) и оба грузовика (потока) продолжают свой путь (работу).  

  
  
