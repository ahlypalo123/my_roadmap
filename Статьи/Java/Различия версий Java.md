## Java 8

В Java 8 появилось несколько значительных нововведений, среди которых наиболее заметными являются ==лямбда-выражения, функциональные интерфейсы и Stream API==, значительно расширившие возможности функционального программирования в Java. Также появились новые API для работы с датами и временем, повторные аннотации и статические методы в интерфейсах. 

Основные нововведения Java 8:

- **[Лямбда-выражения](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifOgMPF-pd7GUFXLYcK_iYm7WwPURw%3A1754029844230&q=%D0%9B%D1%8F%D0%BC%D0%B1%D0%B4%D0%B0-%D0%B2%D1%8B%D1%80%D0%B0%D0%B6%D0%B5%D0%BD%D0%B8%D1%8F&sa=X&ved=2ahUKEwi7weqt_uiOAxUSIxAIHRhAHiAQxccNegQIDRAB&mstk=AUtExfBLW2Xq3IiKvKNouZjtbC2Pd89m-9d7S7Ky_Zw4T_DSE7q9QBJgDdokRZBfpFotuitSGixtuldh8Shy7yxmnPYUtxJNj30kpGwrEPlqUemQfZElX3qfzuT88cNPtok3kYdgioWD95MU0pb-vpIY0PJ2S3nIdhs1UiTcwP80EBiBEhOu4dE9cM9UIUQRy8EOPtw7qzoTGlk87GyA7KhmDQP4gDvpD3eMY3eSyrulJuRngsETYpbTkbPjVXr5O5tbCJK-u6lUDuStBLtqzueCoCLa&csui=3):**
    
    Это анонимные функции, которые позволяют писать более компактный и выразительный код, особенно при работе с коллекциями. Они являются своего рода синтаксическим сахаром для анонимных классов с одним методом. 
    
- **[Функциональные интерфейсы](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifOgMPF-pd7GUFXLYcK_iYm7WwPURw%3A1754029844230&q=%D0%A4%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D0%BE%D0%BD%D0%B0%D0%BB%D1%8C%D0%BD%D1%8B%D0%B5+%D0%B8%D0%BD%D1%82%D0%B5%D1%80%D1%84%D0%B5%D0%B9%D1%81%D1%8B&sa=X&ved=2ahUKEwi7weqt_uiOAxUSIxAIHRhAHiAQxccNegQIExAB&mstk=AUtExfBLW2Xq3IiKvKNouZjtbC2Pd89m-9d7S7Ky_Zw4T_DSE7q9QBJgDdokRZBfpFotuitSGixtuldh8Shy7yxmnPYUtxJNj30kpGwrEPlqUemQfZElX3qfzuT88cNPtok3kYdgioWD95MU0pb-vpIY0PJ2S3nIdhs1UiTcwP80EBiBEhOu4dE9cM9UIUQRy8EOPtw7qzoTGlk87GyA7KhmDQP4gDvpD3eMY3eSyrulJuRngsETYpbTkbPjVXr5O5tbCJK-u6lUDuStBLtqzueCoCLa&csui=3):**
    
    Интерфейсы с одним абстрактным методом, которые используются в сочетании с лямбда-выражениями для поддержки функционального программирования. 
    
- **[Stream API](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifOgMPF-pd7GUFXLYcK_iYm7WwPURw%3A1754029844230&q=Stream+API&sa=X&ved=2ahUKEwi7weqt_uiOAxUSIxAIHRhAHiAQxccNegQIEBAB&mstk=AUtExfBLW2Xq3IiKvKNouZjtbC2Pd89m-9d7S7Ky_Zw4T_DSE7q9QBJgDdokRZBfpFotuitSGixtuldh8Shy7yxmnPYUtxJNj30kpGwrEPlqUemQfZElX3qfzuT88cNPtok3kYdgioWD95MU0pb-vpIY0PJ2S3nIdhs1UiTcwP80EBiBEhOu4dE9cM9UIUQRy8EOPtw7qzoTGlk87GyA7KhmDQP4gDvpD3eMY3eSyrulJuRngsETYpbTkbPjVXr5O5tbCJK-u6lUDuStBLtqzueCoCLa&csui=3):**
    
    Потоки данных, которые позволяют выполнять операции над коллекциями в функциональном стиле, такие как фильтрация, отображение и сортировка. Stream API значительно упрощает работу с большими объемами данных и обеспечивает параллельную обработку. 
    
- **[Новый API для работы с датами и временем](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifOgMPF-pd7GUFXLYcK_iYm7WwPURw%3A1754029844230&q=%D0%9D%D0%BE%D0%B2%D1%8B%D0%B9+API+%D0%B4%D0%BB%D1%8F+%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D1%8B+%D1%81+%D0%B4%D0%B0%D1%82%D0%B0%D0%BC%D0%B8+%D0%B8+%D0%B2%D1%80%D0%B5%D0%BC%D0%B5%D0%BD%D0%B5%D0%BC&sa=X&ved=2ahUKEwi7weqt_uiOAxUSIxAIHRhAHiAQxccNegQIERAB&mstk=AUtExfBLW2Xq3IiKvKNouZjtbC2Pd89m-9d7S7Ky_Zw4T_DSE7q9QBJgDdokRZBfpFotuitSGixtuldh8Shy7yxmnPYUtxJNj30kpGwrEPlqUemQfZElX3qfzuT88cNPtok3kYdgioWD95MU0pb-vpIY0PJ2S3nIdhs1UiTcwP80EBiBEhOu4dE9cM9UIUQRy8EOPtw7qzoTGlk87GyA7KhmDQP4gDvpD3eMY3eSyrulJuRngsETYpbTkbPjVXr5O5tbCJK-u6lUDuStBLtqzueCoCLa&csui=3):**
    
    Вместо устаревшего класса `Date` был представлен новый API, который упрощает работу с датами и временем. 
    
- **[Повторяющиеся аннотации](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifOgMPF-pd7GUFXLYcK_iYm7WwPURw%3A1754029844230&q=%D0%9F%D0%BE%D0%B2%D1%82%D0%BE%D1%80%D1%8F%D1%8E%D1%89%D0%B8%D0%B5%D1%81%D1%8F+%D0%B0%D0%BD%D0%BD%D0%BE%D1%82%D0%B0%D1%86%D0%B8%D0%B8&sa=X&ved=2ahUKEwi7weqt_uiOAxUSIxAIHRhAHiAQxccNegQIDxAB&mstk=AUtExfBLW2Xq3IiKvKNouZjtbC2Pd89m-9d7S7Ky_Zw4T_DSE7q9QBJgDdokRZBfpFotuitSGixtuldh8Shy7yxmnPYUtxJNj30kpGwrEPlqUemQfZElX3qfzuT88cNPtok3kYdgioWD95MU0pb-vpIY0PJ2S3nIdhs1UiTcwP80EBiBEhOu4dE9cM9UIUQRy8EOPtw7qzoTGlk87GyA7KhmDQP4gDvpD3eMY3eSyrulJuRngsETYpbTkbPjVXr5O5tbCJK-u6lUDuStBLtqzueCoCLa&csui=3):**
    
    Теперь аннотации можно применять несколько раз к одному элементу. 
    
- **[Статические методы в интерфейсах](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifOgMPF-pd7GUFXLYcK_iYm7WwPURw%3A1754029844230&q=%D0%A1%D1%82%D0%B0%D1%82%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%B8%D0%B5+%D0%BC%D0%B5%D1%82%D0%BE%D0%B4%D1%8B+%D0%B2+%D0%B8%D0%BD%D1%82%D0%B5%D1%80%D1%84%D0%B5%D0%B9%D1%81%D0%B0%D1%85&sa=X&ved=2ahUKEwi7weqt_uiOAxUSIxAIHRhAHiAQxccNegQIDhAB&mstk=AUtExfBLW2Xq3IiKvKNouZjtbC2Pd89m-9d7S7Ky_Zw4T_DSE7q9QBJgDdokRZBfpFotuitSGixtuldh8Shy7yxmnPYUtxJNj30kpGwrEPlqUemQfZElX3qfzuT88cNPtok3kYdgioWD95MU0pb-vpIY0PJ2S3nIdhs1UiTcwP80EBiBEhOu4dE9cM9UIUQRy8EOPtw7qzoTGlk87GyA7KhmDQP4gDvpD3eMY3eSyrulJuRngsETYpbTkbPjVXr5O5tbCJK-u6lUDuStBLtqzueCoCLa&csui=3):**
    
    Позволяют добавлять статические методы в интерфейсы, что расширяет возможности их использования. 
    

В целом, Java 8 значительно упростила написание кода, добавила новые возможности для функционального программирования и улучшила работу с датами и временем, что сделало язык более современным и удобным.

## Java 11

В Java 11 появилось несколько новых функций и улучшений, включая новые методы класса `String`, HTTP-клиент, запуск однофайловых Java-программ, улучшенную поддержку Docker, поддержку Unicode 10, а также новые методы в классе `Files`. Кроме того, были удалены устаревшие модули Java EE и CORBA. 

Основные новшества Java 11:

- **Новые методы класса String:**
    
    - `isBlank()`: Проверяет, является ли строка пустой или содержит только пробельные символы. 
    - `lines()`: Возвращает поток строк, разделенных разделителями строк. 
    - `strip()`: Удаляет начальные и конечные пробельные символы. 
    - `stripLeading()`: Удаляет начальные пробельные символы. 
    - `stripTrailing()`: Удаляет конечные пробельные символы. 
    - `repeat(int)`: Возвращает строку, содержащую заданное количество повторений исходной строки. 
    
- **HTTP-клиент:**
    
    HTTP-клиент, который появился в Java 9 как инкубационный модуль, стал стандартным в Java 11. 
    
- **Запуск однофайловых Java-программ:**
    
    Теперь можно запускать Java-файлы напрямую, без предварительной компиляции, что упрощает тестирование. 
    
- **Улучшенная поддержка Docker:**
    
    В Java 11 улучшена поддержка контейнеров Docker. 
    
- **Поддержка Unicode 10:**
    
    Java 11 поддерживает Unicode 10. 
    
- **Удаление устаревших модулей:**
    
    Модули Java EE и CORBA, которые были помечены как устаревшие в Java 9, были удалены в Java 11. 
    
- **Новые методы в классе Files:**
    
    Добавлено несколько новых методов в класс `java.nio.file.Files`. 
    
- **Синтаксис локальных переменных для лямбда-выражений:**
    
    Ключевое слово `var` можно использовать для объявления переменных в лямбда-выражениях. 
    
- **Epsilon: No-Op Garbage Collector:**
    
    В Java 11 добавлен экспериментальный сборщик мусора Epsilon, который не выполняет сборку мусора. 
    
- **Flight Recorder:**
    
    Утилита Flight Recorder для диагностики и профилирования Java-приложений стала общедоступной частью OpenJDK. 
    

В целом, Java 11 предлагает ряд улучшений и новых функций, которые делают разработку более удобной и эффективной.

## Java 17

Java 17, выпущенная в сентябре 2021 года, является версией с долгосрочной поддержкой (LTS) и принесла несколько значительных улучшений и новых функций. Среди них: запечатанные классы, улучшенный API для работы со случайными числами, предварительный просмотр сопоставления с шаблоном для операторов `switch`, а также [API для работы с внешними функциями и памятью](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifNN4PtUDBn0GqAX-9QAcfWNJG8MOg%3A1754029950009&q=API+%D0%B4%D0%BB%D1%8F+%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D1%8B+%D1%81+%D0%B2%D0%BD%D0%B5%D1%88%D0%BD%D0%B8%D0%BC%D0%B8+%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D1%8F%D0%BC%D0%B8+%D0%B8+%D0%BF%D0%B0%D0%BC%D1%8F%D1%82%D1%8C%D1%8E&sa=X&ved=2ahUKEwjT0r7g_uiOAxUaUlUIHTAiAYAQxccNegQIBBAB&mstk=AUtExfCAVBbqqLHUVeVp6WvO7B0dE7orwIvbkJGQ_UA7ZrODHgLFUZLUCjgtY-D_cVU7nOvVYXIAPs97fGdYECNbNvN7jZ3Wcf6kX-cCXxgyCpexS8htDlc84Eu0UEfmSvpI0FG03a7UiAvg3X4tuDD5Wum_Sr5KJnwO5ajuUjttc1m-TFpaF_h-w5vY8MNt9bz4bqlIRTPzw1xVgkFWODpxDa6Ai_Ecr_JCDao2pKTWKOokzAQrpnjn-_G__xpLDYJNLczDdwQR1rlXbxubRh7rol3v&csui=3) (Foreign Function & Memory API). ==Также были удалены устаревшие API и компоненты, такие как Security Manager и RMI Activation==. 

Основные нововведения и улучшения в Java 17:

- **[Запечатанные классы](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifNN4PtUDBn0GqAX-9QAcfWNJG8MOg%3A1754029950009&q=%D0%97%D0%B0%D0%BF%D0%B5%D1%87%D0%B0%D1%82%D0%B0%D0%BD%D0%BD%D1%8B%D0%B5+%D0%BA%D0%BB%D0%B0%D1%81%D1%81%D1%8B&sa=X&ved=2ahUKEwjT0r7g_uiOAxUaUlUIHTAiAYAQxccNegQIDxAB&mstk=AUtExfCAVBbqqLHUVeVp6WvO7B0dE7orwIvbkJGQ_UA7ZrODHgLFUZLUCjgtY-D_cVU7nOvVYXIAPs97fGdYECNbNvN7jZ3Wcf6kX-cCXxgyCpexS8htDlc84Eu0UEfmSvpI0FG03a7UiAvg3X4tuDD5Wum_Sr5KJnwO5ajuUjttc1m-TFpaF_h-w5vY8MNt9bz4bqlIRTPzw1xVgkFWODpxDa6Ai_Ecr_JCDao2pKTWKOokzAQrpnjn-_G__xpLDYJNLczDdwQR1rlXbxubRh7rol3v&csui=3) (Sealed Classes):**
    
    Позволяют ограничить иерархию наследования, определяя, какие классы могут расширять данный класс. Это обеспечивает лучшую инкапсуляцию и контроль над структурой классов. 
    
- **Улучшенный [API для генерации случайных чисел](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifNN4PtUDBn0GqAX-9QAcfWNJG8MOg%3A1754029950009&q=API+%D0%B4%D0%BB%D1%8F+%D0%B3%D0%B5%D0%BD%D0%B5%D1%80%D0%B0%D1%86%D0%B8%D0%B8+%D1%81%D0%BB%D1%83%D1%87%D0%B0%D0%B9%D0%BD%D1%8B%D1%85+%D1%87%D0%B8%D1%81%D0%B5%D0%BB&sa=X&ved=2ahUKEwjT0r7g_uiOAxUaUlUIHTAiAYAQxccNegQIFRAB&mstk=AUtExfCAVBbqqLHUVeVp6WvO7B0dE7orwIvbkJGQ_UA7ZrODHgLFUZLUCjgtY-D_cVU7nOvVYXIAPs97fGdYECNbNvN7jZ3Wcf6kX-cCXxgyCpexS8htDlc84Eu0UEfmSvpI0FG03a7UiAvg3X4tuDD5Wum_Sr5KJnwO5ajuUjttc1m-TFpaF_h-w5vY8MNt9bz4bqlIRTPzw1xVgkFWODpxDa6Ai_Ecr_JCDao2pKTWKOokzAQrpnjn-_G__xpLDYJNLczDdwQR1rlXbxubRh7rol3v&csui=3):**
    
    Добавлены новые интерфейсы и классы для работы со случайными числами, включая поддержку различных алгоритмов генерации. 
    
- Предварительный просмотр сопоставления с шаблоном для операторов `switch`:
    
    Позволяет использовать более гибкие шаблоны в метках `case` оператора `switch`, что упрощает обработку различных вариантов. 
    
- **[Foreign Function & Memory API](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifNN4PtUDBn0GqAX-9QAcfWNJG8MOg%3A1754029950009&q=Foreign+Function+%26+Memory+API&sa=X&ved=2ahUKEwjT0r7g_uiOAxUaUlUIHTAiAYAQxccNegQIFBAB&mstk=AUtExfCAVBbqqLHUVeVp6WvO7B0dE7orwIvbkJGQ_UA7ZrODHgLFUZLUCjgtY-D_cVU7nOvVYXIAPs97fGdYECNbNvN7jZ3Wcf6kX-cCXxgyCpexS8htDlc84Eu0UEfmSvpI0FG03a7UiAvg3X4tuDD5Wum_Sr5KJnwO5ajuUjttc1m-TFpaF_h-w5vY8MNt9bz4bqlIRTPzw1xVgkFWODpxDa6Ai_Ecr_JCDao2pKTWKOokzAQrpnjn-_G__xpLDYJNLczDdwQR1rlXbxubRh7rol3v&csui=3) (Инкубатор):**
    
    Предоставляет возможность взаимодействия с кодом и данными, находящимися за пределами виртуальной машины Java. Это позволяет приложениям Java взаимодействовать с нативными библиотеками и системными ресурсами. 
    
- **Улучшения в [API для работы с датой и временем](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifNN4PtUDBn0GqAX-9QAcfWNJG8MOg%3A1754029950009&q=API+%D0%B4%D0%BB%D1%8F+%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D1%8B+%D1%81+%D0%B4%D0%B0%D1%82%D0%BE%D0%B9+%D0%B8+%D0%B2%D1%80%D0%B5%D0%BC%D0%B5%D0%BD%D0%B5%D0%BC&sa=X&ved=2ahUKEwjT0r7g_uiOAxUaUlUIHTAiAYAQxccNegQIFhAB&mstk=AUtExfCAVBbqqLHUVeVp6WvO7B0dE7orwIvbkJGQ_UA7ZrODHgLFUZLUCjgtY-D_cVU7nOvVYXIAPs97fGdYECNbNvN7jZ3Wcf6kX-cCXxgyCpexS8htDlc84Eu0UEfmSvpI0FG03a7UiAvg3X4tuDD5Wum_Sr5KJnwO5ajuUjttc1m-TFpaF_h-w5vY8MNt9bz4bqlIRTPzw1xVgkFWODpxDa6Ai_Ecr_JCDao2pKTWKOokzAQrpnjn-_G__xpLDYJNLczDdwQR1rlXbxubRh7rol3v&csui=3):**
    
    Добавлен новый шаблон `B` для форматирования даты и времени, указывающий на суточный интервал. 
    
- **Улучшения в [API для работы с текстом](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifNN4PtUDBn0GqAX-9QAcfWNJG8MOg%3A1754029950009&q=API+%D0%B4%D0%BB%D1%8F+%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D1%8B+%D1%81+%D1%82%D0%B5%D0%BA%D1%81%D1%82%D0%BE%D0%BC&sa=X&ved=2ahUKEwjT0r7g_uiOAxUaUlUIHTAiAYAQxccNegQIFxAB&mstk=AUtExfCAVBbqqLHUVeVp6WvO7B0dE7orwIvbkJGQ_UA7ZrODHgLFUZLUCjgtY-D_cVU7nOvVYXIAPs97fGdYECNbNvN7jZ3Wcf6kX-cCXxgyCpexS8htDlc84Eu0UEfmSvpI0FG03a7UiAvg3X4tuDD5Wum_Sr5KJnwO5ajuUjttc1m-TFpaF_h-w5vY8MNt9bz4bqlIRTPzw1xVgkFWODpxDa6Ai_Ecr_JCDao2pKTWKOokzAQrpnjn-_G__xpLDYJNLczDdwQR1rlXbxubRh7rol3v&csui=3):**
    
    Добавлен метод `stripIndent()` для удаления общего выравнивания в текстовых блоках. 
    
- **Удаление устаревших API и компонентов:**
    
    Java 17 удаляет ряд устаревших API, включая Security Manager и RMI Activation. 
    
- **Новый [рендеринг для macOS](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifNN4PtUDBn0GqAX-9QAcfWNJG8MOg%3A1754029950009&q=%D1%80%D0%B5%D0%BD%D0%B4%D0%B5%D1%80%D0%B8%D0%BD%D0%B3+%D0%B4%D0%BB%D1%8F+macOS&sa=X&ved=2ahUKEwjT0r7g_uiOAxUaUlUIHTAiAYAQxccNegQIERAB&mstk=AUtExfCAVBbqqLHUVeVp6WvO7B0dE7orwIvbkJGQ_UA7ZrODHgLFUZLUCjgtY-D_cVU7nOvVYXIAPs97fGdYECNbNvN7jZ3Wcf6kX-cCXxgyCpexS8htDlc84Eu0UEfmSvpI0FG03a7UiAvg3X4tuDD5Wum_Sr5KJnwO5ajuUjttc1m-TFpaF_h-w5vY8MNt9bz4bqlIRTPzw1xVgkFWODpxDa6Ai_Ecr_JCDao2pKTWKOokzAQrpnjn-_G__xpLDYJNLczDdwQR1rlXbxubRh7rol3v&csui=3):**
    
    Добавлен новый рендеринг для macOS, использующий графический API Metal. 
    
- **Порт для macOS/AArch64:**
    
    Добавлен порт для платформы macOS на базе чипов Apple M1. 
    

Дополнительно:

- Java 17 предлагает более компактную, быструю и простую в поддержке платформу по сравнению с более ранними версиями, такими как Java 8. 
- Обновление до Java 17 может потребовать внесения изменений в код, особенно если используются устаревшие API, которые были удалены. 

В целом, Java 17 является важной версией, предлагающей множество улучшений и нововведений, направленных на повышение производительности, безопасности и удобства использования языка.

## Java 21

В Java 21 появилось много нововведений, включая ==стабилизацию виртуальных потоков, упорядоченные коллекции, улучшенное сопоставление с образцом и шаблоны строк, а также новый генеративный вариант сборщика мусора ZGC==. 

Основные нововведения в Java 21:

- **[Виртуальные потоки (Virtual Threads)](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifMSLS3Gl-4YAn0s0_-uGzZWCf8P4Q%3A1754029984658&q=%D0%92%D0%B8%D1%80%D1%82%D1%83%D0%B0%D0%BB%D1%8C%D0%BD%D1%8B%D0%B5+%D0%BF%D0%BE%D1%82%D0%BE%D0%BA%D0%B8+%28Virtual+Threads%29&sa=X&ved=2ahUKEwiGoN3w_uiOAxUiU1UIHadCI-0QxccNegQICxAB&mstk=AUtExfByq8FaK3W7KQ9IWojADpHbpXtT4VcxAcdAnL3-GHdLaIwzPvowJ0g5H5tBYCvkTdejzZjKtRQxYBrH8yr29z-_SsLnkFt9RVP33kNZ0dy8zXE8z3_S3Xytg_m0-Pl95d8OErR0F18jPoErFzCOCws-7G9kEwNKJScEYXorXOer7JYlhwryHQ-JkBIZAn4B2REvNcy9qwimIcbgor9Y26LOoq9I528-FiT32XcAGWfAu0vrL3Nsg5Uy4thvDFcsa0vFUd_APD-GXcupqwdUIei2&csui=3):**
    
    Легковесные потоки, значительно упрощающие написание и сопровождение высокопроизводительных многопоточных приложений. 
    
- **[Упорядоченные коллекции (Sequenced Collections)](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifMSLS3Gl-4YAn0s0_-uGzZWCf8P4Q%3A1754029984658&q=%D0%A3%D0%BF%D0%BE%D1%80%D1%8F%D0%B4%D0%BE%D1%87%D0%B5%D0%BD%D0%BD%D1%8B%D0%B5+%D0%BA%D0%BE%D0%BB%D0%BB%D0%B5%D0%BA%D1%86%D0%B8%D0%B8+%28Sequenced+Collections%29&sa=X&ved=2ahUKEwiGoN3w_uiOAxUiU1UIHadCI-0QxccNegQIERAB&mstk=AUtExfByq8FaK3W7KQ9IWojADpHbpXtT4VcxAcdAnL3-GHdLaIwzPvowJ0g5H5tBYCvkTdejzZjKtRQxYBrH8yr29z-_SsLnkFt9RVP33kNZ0dy8zXE8z3_S3Xytg_m0-Pl95d8OErR0F18jPoErFzCOCws-7G9kEwNKJScEYXorXOer7JYlhwryHQ-JkBIZAn4B2REvNcy9qwimIcbgor9Y26LOoq9I528-FiT32XcAGWfAu0vrL3Nsg5Uy4thvDFcsa0vFUd_APD-GXcupqwdUIei2&csui=3):**
    
    Новые интерфейсы SequencedCollection, SequencedSet и SequencedMap, позволяющие работать с элементами, имеющими предопределенный порядок. 
    
- **[Сопоставление с образцом (Pattern Matching) для switch и Record Patterns](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifMSLS3Gl-4YAn0s0_-uGzZWCf8P4Q%3A1754029984658&q=%D0%A1%D0%BE%D0%BF%D0%BE%D1%81%D1%82%D0%B0%D0%B2%D0%BB%D0%B5%D0%BD%D0%B8%D0%B5+%D1%81+%D0%BE%D0%B1%D1%80%D0%B0%D0%B7%D1%86%D0%BE%D0%BC+%28Pattern+Matching%29+%D0%B4%D0%BB%D1%8F+switch+%D0%B8+Record+Patterns&sa=X&ved=2ahUKEwiGoN3w_uiOAxUiU1UIHadCI-0QxccNegQIDhAB&mstk=AUtExfByq8FaK3W7KQ9IWojADpHbpXtT4VcxAcdAnL3-GHdLaIwzPvowJ0g5H5tBYCvkTdejzZjKtRQxYBrH8yr29z-_SsLnkFt9RVP33kNZ0dy8zXE8z3_S3Xytg_m0-Pl95d8OErR0F18jPoErFzCOCws-7G9kEwNKJScEYXorXOer7JYlhwryHQ-JkBIZAn4B2REvNcy9qwimIcbgor9Y26LOoq9I528-FiT32XcAGWfAu0vrL3Nsg5Uy4thvDFcsa0vFUd_APD-GXcupqwdUIei2&csui=3):**
    
    Расширение возможностей сопоставления с образцом, включая использование шаблонов в операторе switch и разбор значений Record. 
    
- **[Шаблоны строк (String Templates)](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifMSLS3Gl-4YAn0s0_-uGzZWCf8P4Q%3A1754029984658&q=%D0%A8%D0%B0%D0%B1%D0%BB%D0%BE%D0%BD%D1%8B+%D1%81%D1%82%D1%80%D0%BE%D0%BA+%28String+Templates%29&sa=X&ved=2ahUKEwiGoN3w_uiOAxUiU1UIHadCI-0QxccNegQIEBAB&mstk=AUtExfByq8FaK3W7KQ9IWojADpHbpXtT4VcxAcdAnL3-GHdLaIwzPvowJ0g5H5tBYCvkTdejzZjKtRQxYBrH8yr29z-_SsLnkFt9RVP33kNZ0dy8zXE8z3_S3Xytg_m0-Pl95d8OErR0F18jPoErFzCOCws-7G9kEwNKJScEYXorXOer7JYlhwryHQ-JkBIZAn4B2REvNcy9qwimIcbgor9Y26LOoq9I528-FiT32XcAGWfAu0vrL3Nsg5Uy4thvDFcsa0vFUd_APD-GXcupqwdUIei2&csui=3) (Preview):**
    
    Новая синтаксическая конструкция для встраивания выражений в строки. 
    
- **[Безымянные шаблоны и переменные (Unnamed Patterns and Variables)](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifMSLS3Gl-4YAn0s0_-uGzZWCf8P4Q%3A1754029984658&q=%D0%91%D0%B5%D0%B7%D1%8B%D0%BC%D1%8F%D0%BD%D0%BD%D1%8B%D0%B5+%D1%88%D0%B0%D0%B1%D0%BB%D0%BE%D0%BD%D1%8B+%D0%B8+%D0%BF%D0%B5%D1%80%D0%B5%D0%BC%D0%B5%D0%BD%D0%BD%D1%8B%D0%B5+%28Unnamed+Patterns+and+Variables%29&sa=X&ved=2ahUKEwiGoN3w_uiOAxUiU1UIHadCI-0QxccNegQIEhAB&mstk=AUtExfByq8FaK3W7KQ9IWojADpHbpXtT4VcxAcdAnL3-GHdLaIwzPvowJ0g5H5tBYCvkTdejzZjKtRQxYBrH8yr29z-_SsLnkFt9RVP33kNZ0dy8zXE8z3_S3Xytg_m0-Pl95d8OErR0F18jPoErFzCOCws-7G9kEwNKJScEYXorXOer7JYlhwryHQ-JkBIZAn4B2REvNcy9qwimIcbgor9Y26LOoq9I528-FiT32XcAGWfAu0vrL3Nsg5Uy4thvDFcsa0vFUd_APD-GXcupqwdUIei2&csui=3) (Preview):**
    
    Возможность использования безымянных переменных (например, `_`) в различных конструкциях языка. 
    
- **[Генеративный ZGC (Generational Z Garbage Collector)](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifMSLS3Gl-4YAn0s0_-uGzZWCf8P4Q%3A1754029984658&q=%D0%93%D0%B5%D0%BD%D0%B5%D1%80%D0%B0%D1%82%D0%B8%D0%B2%D0%BD%D1%8B%D0%B9+ZGC+%28Generational+Z+Garbage+Collector%29&sa=X&ved=2ahUKEwiGoN3w_uiOAxUiU1UIHadCI-0QxccNegQIExAB&mstk=AUtExfByq8FaK3W7KQ9IWojADpHbpXtT4VcxAcdAnL3-GHdLaIwzPvowJ0g5H5tBYCvkTdejzZjKtRQxYBrH8yr29z-_SsLnkFt9RVP33kNZ0dy8zXE8z3_S3Xytg_m0-Pl95d8OErR0F18jPoErFzCOCws-7G9kEwNKJScEYXorXOer7JYlhwryHQ-JkBIZAn4B2REvNcy9qwimIcbgor9Y26LOoq9I528-FiT32XcAGWfAu0vrL3Nsg5Uy4thvDFcsa0vFUd_APD-GXcupqwdUIei2&csui=3):**
    
    Новый вариант сборщика мусора ZGC, который разделяет обработку объектов на "старые" и "молодые", повышая эффективность очистки. 
    
- **[API механизма инкапсуляции ключей (Key Encapsulation Mechanism (KEM) API)](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifMSLS3Gl-4YAn0s0_-uGzZWCf8P4Q%3A1754029984658&q=API+%D0%BC%D0%B5%D1%85%D0%B0%D0%BD%D0%B8%D0%B7%D0%BC%D0%B0+%D0%B8%D0%BD%D0%BA%D0%B0%D0%BF%D1%81%D1%83%D0%BB%D1%8F%D1%86%D0%B8%D0%B8+%D0%BA%D0%BB%D1%8E%D1%87%D0%B5%D0%B9+%28Key+Encapsulation+Mechanism+%28KEM%29+API%29&sa=X&ved=2ahUKEwiGoN3w_uiOAxUiU1UIHadCI-0QxccNegQIDxAB&mstk=AUtExfByq8FaK3W7KQ9IWojADpHbpXtT4VcxAcdAnL3-GHdLaIwzPvowJ0g5H5tBYCvkTdejzZjKtRQxYBrH8yr29z-_SsLnkFt9RVP33kNZ0dy8zXE8z3_S3Xytg_m0-Pl95d8OErR0F18jPoErFzCOCws-7G9kEwNKJScEYXorXOer7JYlhwryHQ-JkBIZAn4B2REvNcy9qwimIcbgor9Y26LOoq9I528-FiT32XcAGWfAu0vrL3Nsg5Uy4thvDFcsa0vFUd_APD-GXcupqwdUIei2&csui=3):**
    
    API для использования криптографических алгоритмов инкапсуляции ключей, устойчивых к квантовым атакам. 
    
- **[Улучшения производительности](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifMSLS3Gl-4YAn0s0_-uGzZWCf8P4Q%3A1754029984658&q=%D0%A3%D0%BB%D1%83%D1%87%D1%88%D0%B5%D0%BD%D0%B8%D1%8F+%D0%BF%D1%80%D0%BE%D0%B8%D0%B7%D0%B2%D0%BE%D0%B4%D0%B8%D1%82%D0%B5%D0%BB%D1%8C%D0%BD%D0%BE%D1%81%D1%82%D0%B8&sa=X&ved=2ahUKEwiGoN3w_uiOAxUiU1UIHadCI-0QxccNegQIDRAB&mstk=AUtExfByq8FaK3W7KQ9IWojADpHbpXtT4VcxAcdAnL3-GHdLaIwzPvowJ0g5H5tBYCvkTdejzZjKtRQxYBrH8yr29z-_SsLnkFt9RVP33kNZ0dy8zXE8z3_S3Xytg_m0-Pl95d8OErR0F18jPoErFzCOCws-7G9kEwNKJScEYXorXOer7JYlhwryHQ-JkBIZAn4B2REvNcy9qwimIcbgor9Y26LOoq9I528-FiT32XcAGWfAu0vrL3Nsg5Uy4thvDFcsa0vFUd_APD-GXcupqwdUIei2&csui=3):**
    
    В Java 21 внесены различные оптимизации, повышающие общую производительность, особенно в многопоточных приложениях. 
    
- **[Поддержка платформы Windows 32-бит x86 снята с разработки](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifMSLS3Gl-4YAn0s0_-uGzZWCf8P4Q%3A1754029984658&q=%D0%9F%D0%BE%D0%B4%D0%B4%D0%B5%D1%80%D0%B6%D0%BA%D0%B0+%D0%BF%D0%BB%D0%B0%D1%82%D1%84%D0%BE%D1%80%D0%BC%D1%8B+Windows+32-%D0%B1%D0%B8%D1%82+x86+%D1%81%D0%BD%D1%8F%D1%82%D0%B0+%D1%81+%D1%80%D0%B0%D0%B7%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D0%BA%D0%B8&sa=X&ved=2ahUKEwiGoN3w_uiOAxUiU1UIHadCI-0QxccNegQIDBAB&mstk=AUtExfByq8FaK3W7KQ9IWojADpHbpXtT4VcxAcdAnL3-GHdLaIwzPvowJ0g5H5tBYCvkTdejzZjKtRQxYBrH8yr29z-_SsLnkFt9RVP33kNZ0dy8zXE8z3_S3Xytg_m0-Pl95d8OErR0F18jPoErFzCOCws-7G9kEwNKJScEYXorXOer7JYlhwryHQ-JkBIZAn4B2REvNcy9qwimIcbgor9Y26LOoq9I528-FiT32XcAGWfAu0vrL3Nsg5Uy4thvDFcsa0vFUd_APD-GXcupqwdUIei2&csui=3):**
    
    В новой версии Java прекращена поддержка 32-битной версии Windows. 
    

Эти и другие нововведения делают Java 21 значительным шагом вперед в развитии языка и платформы, предлагая разработчикам новые инструменты для создания более эффективных и современных приложений,.

## Java 23

В Java 23 появилось ==несколько новых функций и улучшений, включая поддержку примитивных типов в шаблонах, улучшенную обработку строк, декларативный импорт модулей, а также улучшенный сборщик мусора ZGC==. Кроме того, были представлены предварительные версии таких фич, как Stream Gatherers, Class-File API, Flexible Constructor Bodies, Structured Concurrency и Unnamed Classes and Main Methods. 

Подробнее о нововведениях:

- **Поддержка примитивных типов в шаблонах (pattern matching):**
    
    Теперь примитивные типы данных, такие как `int`, `double`, `boolean`, могут быть использованы в операторах `switch` и `instanceof` для более удобного сопоставления шаблонов. 
    
- **[Stream Gatherers](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifMsmy36jfqE0lkFMYuCJpf1udgoSA%3A1754030011837&q=Stream+Gatherers&sa=X&ved=2ahUKEwjihpT-_uiOAxXpFhAIHTJiCfYQxccNegQIGhAB&mstk=AUtExfCnjMCMXSGIieXlbzPCg09ECuZLGtoO5QjDgf-v69_dI6pdCE6t899QpoAu20wnsURruPW_72y6LV8FgXtSPHcHmxYYtJy8dC2-De_mbzEGaL6u7FUm3Ird-cc0FfDHOHEqSx9XozxEr1GAxe_NlgZz5M_rWr9VRmUaTJOZRfka6PEzTT2cws5uiWlsRvj0352J9W7hOAPMVoEZFAmoReU-ukQWCct7QpW3AZ9iVuIee-VwLkfSvM_NVN6FPNI1N77XIklpSLPvi4xyz1z-jh0f&csui=3):**
    
    Второе превью API, приближающее его к финализации. Это улучшает работу со Stream API, делая его более гибким и мощным. 
    
- **Декларативный импорт модулей:**
    
    Позволяет упростить импорт модулей в код, делая его более читаемым и понятным. 
    
- **[Generational ZGC](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifMsmy36jfqE0lkFMYuCJpf1udgoSA%3A1754030011837&q=Generational+ZGC&sa=X&ved=2ahUKEwjihpT-_uiOAxXpFhAIHTJiCfYQxccNegQIFRAB&mstk=AUtExfCnjMCMXSGIieXlbzPCg09ECuZLGtoO5QjDgf-v69_dI6pdCE6t899QpoAu20wnsURruPW_72y6LV8FgXtSPHcHmxYYtJy8dC2-De_mbzEGaL6u7FUm3Ird-cc0FfDHOHEqSx9XozxEr1GAxe_NlgZz5M_rWr9VRmUaTJOZRfka6PEzTT2cws5uiWlsRvj0352J9W7hOAPMVoEZFAmoReU-ukQWCct7QpW3AZ9iVuIee-VwLkfSvM_NVN6FPNI1N77XIklpSLPvi4xyz1z-jh0f&csui=3) по умолчанию:**
    
    Сборщик мусора ZGC теперь используется по умолчанию, что должно повысить производительность приложений, особенно при работе с большими объемами данных. 
    
- **[Class-File API (2-й preview)](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifMsmy36jfqE0lkFMYuCJpf1udgoSA%3A1754030011837&q=Class-File+API+%282-%D0%B9+preview%29&sa=X&ved=2ahUKEwjihpT-_uiOAxXpFhAIHTJiCfYQxccNegQIHhAB&mstk=AUtExfCnjMCMXSGIieXlbzPCg09ECuZLGtoO5QjDgf-v69_dI6pdCE6t899QpoAu20wnsURruPW_72y6LV8FgXtSPHcHmxYYtJy8dC2-De_mbzEGaL6u7FUm3Ird-cc0FfDHOHEqSx9XozxEr1GAxe_NlgZz5M_rWr9VRmUaTJOZRfka6PEzTT2cws5uiWlsRvj0352J9W7hOAPMVoEZFAmoReU-ukQWCct7QpW3AZ9iVuIee-VwLkfSvM_NVN6FPNI1N77XIklpSLPvi4xyz1z-jh0f&csui=3):**
    
    API для работы с файлами классов, позволяющий более гибко манипулировать кодом на уровне байт-кода. 
    
- **[Flexible Constructor Bodies (2-й preview)](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifMsmy36jfqE0lkFMYuCJpf1udgoSA%3A1754030011837&q=Flexible+Constructor+Bodies+%282-%D0%B9+preview%29&sa=X&ved=2ahUKEwjihpT-_uiOAxXpFhAIHTJiCfYQxccNegQILRAB&mstk=AUtExfCnjMCMXSGIieXlbzPCg09ECuZLGtoO5QjDgf-v69_dI6pdCE6t899QpoAu20wnsURruPW_72y6LV8FgXtSPHcHmxYYtJy8dC2-De_mbzEGaL6u7FUm3Ird-cc0FfDHOHEqSx9XozxEr1GAxe_NlgZz5M_rWr9VRmUaTJOZRfka6PEzTT2cws5uiWlsRvj0352J9W7hOAPMVoEZFAmoReU-ukQWCct7QpW3AZ9iVuIee-VwLkfSvM_NVN6FPNI1N77XIklpSLPvi4xyz1z-jh0f&csui=3):**
    
    Улучшает гибкость работы с конструкторами, позволяя более удобно инициализировать объекты. 
    
- **[Structured Concurrency (3-й preview)](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifMsmy36jfqE0lkFMYuCJpf1udgoSA%3A1754030011837&q=Structured+Concurrency+%283-%D0%B9+preview%29&sa=X&ved=2ahUKEwjihpT-_uiOAxXpFhAIHTJiCfYQxccNegQIExAB&mstk=AUtExfCnjMCMXSGIieXlbzPCg09ECuZLGtoO5QjDgf-v69_dI6pdCE6t899QpoAu20wnsURruPW_72y6LV8FgXtSPHcHmxYYtJy8dC2-De_mbzEGaL6u7FUm3Ird-cc0FfDHOHEqSx9XozxEr1GAxe_NlgZz5M_rWr9VRmUaTJOZRfka6PEzTT2cws5uiWlsRvj0352J9W7hOAPMVoEZFAmoReU-ukQWCct7QpW3AZ9iVuIee-VwLkfSvM_NVN6FPNI1N77XIklpSLPvi4xyz1z-jh0f&csui=3):**
    
    Упрощает написание многопоточного кода, делая его более надежным и предсказуемым. 
    
- **[Unnamed Classes and Main Methods (3-й preview)](https://www.google.com/search?sca_esv=7e391c2c0531b813&cs=0&sxsrf=AE3TifMsmy36jfqE0lkFMYuCJpf1udgoSA%3A1754030011837&q=Unnamed+Classes+and+Main+Methods+%283-%D0%B9+preview%29&sa=X&ved=2ahUKEwjihpT-_uiOAxXpFhAIHTJiCfYQxccNegQIFBAB&mstk=AUtExfCnjMCMXSGIieXlbzPCg09ECuZLGtoO5QjDgf-v69_dI6pdCE6t899QpoAu20wnsURruPW_72y6LV8FgXtSPHcHmxYYtJy8dC2-De_mbzEGaL6u7FUm3Ird-cc0FfDHOHEqSx9XozxEr1GAxe_NlgZz5M_rWr9VRmUaTJOZRfka6PEzTT2cws5uiWlsRvj0352J9W7hOAPMVoEZFAmoReU-ukQWCct7QpW3AZ9iVuIee-VwLkfSvM_NVN6FPNI1N77XIklpSLPvi4xyz1z-jh0f&csui=3):**
    
    Позволяет создавать классы и методы без явного указания имени, что упрощает написание небольших программ и сценариев. 
    
- **Улучшения в обработке строк:**
    
    Например, метод `String::hashCode` может быть оптимизирован компилятором для повышения производительности. 
    
- **Прекращение поддержки методов доступа к памяти в Sun:**
    
    Устаревшие методы `sun.misc.Unsafe` постепенно выводятся из использования. 
    

Java 23 не является LTS-версией, поэтому рекомендуется использовать её для изучения новых возможностей и тестирования приложений.