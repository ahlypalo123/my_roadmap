---
title: "Что такое merge join, hash join и nested loop с примером на PostgreSQL."
source: "https://dzen.ru/a/ZumWFZ9ztwDXdcQc"
author:
  - "[[gelovolro]]"
published:
created: 2025-08-04
description: "Статья автора «gelovolro» в Дзене ✍: В этой статье давайте рассмотрим три ключевых типа физических соединений, которые PostgreSQL использует при выполнении логических внешних и внутренних соединений,"
tags:
  - "clippings"
---

### Введение.

В этой статье давайте рассмотрим **три ключевых типа физических соединений**, которые PostgreSQL использует при выполнении логических внешних и внутренних соединений, использующих оператор JOIN:

- merge join,
- hash join,
- и nested loop.

Почему я начал с упоминания **про деление на физические и логические соединения**? Это разные уровни. Когда вы пишите SQL-запросы с применением LEFT OUTER JOIN или RIGHT JOIN, это логический уровень соединений. Как правило, использующийся для декартовых произведений, в которых остаются строки, удовлетворяющие условию самого соединения. То, как исполняются “под капотом” подобные запросы в РСУБД - это уже физический уровень, который зависит от ряда факторов, как:

- *размеры таблиц, участвующих при операции соединения.*
- *размер памяти, выделенный в “work\_mem” под операции сортировок и в “hash\_mem\_multiplier” под операций с хешированием.*
- *наличие индексов.*
- *применяемые операции сравнения или равенства при соединении.*

В зависимости от данных условий, планировщик PostgreSQL может выбрать тот или иной метод (алгоритм), который реализует обработку логических соединений.

### Описание каждого типа соединения.

  
**Merge join** \- это метод соединения слиянием, используемый когда обе таблицы достаточно большие и уже есть отсортированные данные по полям (JOIN-ключам участвующим в соединении), например при помощи индексов. Если таблицы не отсортированы заранее, то PostgreSQL выполнит сортировку перед выполнением соединения, что может увеличить затраты. **Ключевое отличие от hash join в том**, что merge join требует обязательной сортировки данных по JOIN-ключам, когда hash join не требует сортировки вообще. Он просто построит хеш-таблицу и будет использовать ее для поиска совпадений. Так же, про merge join стоит добавить, что он особенно полезен для диапазонных соединений (*например, >=, <=*), поскольку после сортировки он может эффективно обрабатывать такие условия.

**Hash join** \- это метод соединения при помощи хэширования, который используется в тех случаях, когда данные, по которым требуется выполнить соединение, не отсортированы. Алгоритм работает так, что сначала выбирается одна из таблиц, обычно меньшая по памяти, и для каждой ее строки создается запись в хеш-таблице. Затем, другая таблица (*большая*) сканируется, и каждая строка проверяется на сравнение с хеш-таблицей. Если найдено совпадение по нужным значениям, то строки объединяются. Так же, более явно подчеркну деталь, что PostgreSQL выбирает меньшую таблицу **для создания хеш-таблицы, чтобы минимизировать использование памяти**. Данная хеш-таблица затем используется для быстрого поиска соответствий во второй, более крупной таблицей. Финализируя можно сказать, что hash join чаще всего используется для больших таблиц, когда отсутствуют индексы или когда данные не отсортированы, а условие соединения предполагает точное совпадение (*\=*). Диапазонные соединения, как правило, менее эффективны с hash join.

**Nested Loop** \- это метод соединения, использующий вложенные циклы, который для каждого значения одной таблицы, ищет соответствующее значение из другой таблицы. Если детальнее, то берется первое значение из первой таблицы и сравнивается последовательно со всеми значениями из второй таблицы, если находится соответствие, то запись включается в финальный набор данных. Когда значение из первой таблицы сравнилось со всеми значениями из второй таблицы, далее берется второе значение из первой маленькой таблицы и снова происходит сравнение со всеми значениями из второй таблицы. Собственно почему и используется название “nested loop” (*вложеннеого цикла*), т.к. весь процесс происходит до тех пор, пока каждое значение из первой таблицы не будет сравнено с каждым значением из второй таблицы.

**Читая иные статьи в интернете, я видел информацию, что “якобы” nested loop всегда строго используется при таблицах, когда одна - маленькая, а другая - большая.** Это совсем необязательно, и я сейчас приведу SQL-код для примера с выводом плана запроса через “EXPLAIN ANALYZE”:

![к сожалению, Дзен перестал поддерживать функционал вставки кода, на момент написания статьи, поэтому приходится вставлять картинку, а снизу ссылку на код в GitHub Gist](https://static.dzeninfra.ru/s3/zen-lib/1.003.1/dzen-desktop/lz5XeGt8fsa/cQ9a72h66/1fc23cEEJZd/VxmQKv6YnVc8tL5UBmFuqVqb0foojESLe3x1VATwda-w-VoBm_b0Zfv_2FY1FeqkjdDq_plV6ZtkHNFzBoBHTVLa1ix-CAlS8yWvBtExueqZjhE-_QvAW_xooAUCtnR5vqudAYhhYP55v9pR_t9iYxDKoGRFSbeFtphE4sRVQNhVTkB7fVyZkm90v8hIt-3mmJiY9iwby8iqdPpXHPe0r3r9VEhAYiT5bq3MS_iRKuPbx-BlD00xj7ZXSu2DVp3S0EMIuLJZWN8xOvwKTWWh6R1bE75m3QYH7HV2k1X0sebxPNzLxK7sO2vgDECw3q_jkI5qaQuNNkh414FkDlGRWhoCyjK_3pQZdbs6BEMprKNdUBx27d3FDe13NRub-7Km9_zTggZi5Huup4BOMtSj7VkMKmgHzHvG_BiP5oCZUNKSBoO7vxfXE704sQ9FrG7hG1paNC3VRQ6q9v7fXfu17v20lEwDpmD7Ia0DTbPeKG_cR-ElTU33j3-YB2dKHtEf103P9DdYVJYxdXcLBG1po5HT0X7i08DN43v_3Na1_aT09lhDRmaoMGRuBsyyVaEr2Ammo4PLO09_Vsbhxx5akZEEzre121NYdfC_BAtr6qpVWB445V3EgqV9eVebNX2n_PFYzIIsbbPuKkpFuhqnJ5kJoeqDSbaLvl8C7gVd3hjfS0E_dJMdmDm6M8_MJSQsmRRR-uPUBc0peX6VUnx1qDx9EYBLYGcx5-nGi7tYYKdZiCgphYWxDfYVBOHH054Ql87GuH2bUxG0OjzBDCamKVga07Au1QWLpr4xWBuyv2GyfNtDTC7tcSyigsT9E6Yg3g2t7kNOMcT80EVvjVRf0xhHQj8_kZaQdr08wsfso6vUVhGwKtvMRG3-tN0UdrRvs7FYD0SqJfCtqM9FMNFg59ML4mBFi7WAeFjCoUORFxIfBomy8xnW2nX6MsXCIWSqEJpUM61egEzh-TkVF_K1oT_1GIpN7yh4aWPPRDlaYSucC-tjgotxy7keSu6HkZYQFUELs75R2xO4vjZJC-Tt7tNaFHAgUc2NqHT8n19y-W87eB-Cwu7h-Wiiiwky02DplAtqYQOFN8gx1cfjhV1YFBaLCXB_WJORNfI2xQ5urGOakN145tOBhS04d1qXMLcgdfHTys6pKnVtJw0IfZIjJRHMomSHSXZH9tBHLQpemphTDclx_ZjZ2b-z8ERO7WYgE5SWdW7cgE8r_XbblPLyorx1WQmDoS2yYWVOCbSZomZdR25rTgR8yvYQBmpNmdbQUsNJ-vCUEFb8dPeOy2GvIBGZkjWumcDMbjV1Fxa18Sa_uRrCjettM6bixwe0nOrh2Q_vrsmLd0aznQitT1RVFdrFDro0k1TQ8P0yDwej7GSQEJjwJJYNQKm5eVQZv7bvurQcw86rY7-v5caHOpzp6pCLJaxKw_GO95COpUST2tIbiY34ut2bF_hxsQKB5G2vmRub8-IdTcpqsXJVXr09r7K5m0vBpyTyqKdIDHNcJuBQxmKpAkmwSnwZh6FFlNCS0M3CPTVfXJ8wMn7HDaxnYxISEfxk2gZH6T5xGpd4ue7_fFBFBq4hOCngh8D5VaQuFM2qqk9E_AtzFcBuDlidXVbHjnFznlmeOX34jwepY-STE910YRINjK44dNFcN3Zh_nVXiIruo3RlIA0IdRapZpwJqaOKhneE8dzC5kVeUxiRAY_xsJVc0_yx90oM6mpm21vQNmOdRcwhO_oTm7SyrHa6Xc1MIap_5SjPzrNV42dWBaWhzUF-RLlXBS8DmB9XGoYA_LQYWBZ4MT4Kxe1nahMcHnIq2wvE6nC_mtP9ea11vV_Fwmik_KWmyEY1kikn2Uai6o9FdUC0WA4rC5iX0ZiCyjm0mxWR_fP7BslsreYS05F8YF5MBC6ztxnVMLEnOjyZDUCjrbRmaADB-xtvLRIFqmVKw_lFPxlF7IBWmJnZBoOytJbcln64vg)

к сожалению, Дзен перестал поддерживать функционал вставки кода, на момент написания статьи, поэтому приходится вставлять картинку, а снизу ссылку на код в GitHub Gist

пример nested loop с одинаковым размером таблиц

Если выполнить данный запрос, то при выводе плана вы увидите именно nested loop, хотя размер у 2-ух таблиц - одинаковый:

![вывод плана с nested loop](https://static.dzeninfra.ru/s3/zen-lib/1.003.1/dzen-desktop/lz5XeGt8fsa/cQ9a72h66/1fc23cEEJZd/VxmQKv6YnVc8tL5UBmFuqVqb0foojESLe3x1VATwda-w-VoBm_b0Zfv_2FY1FeqkjdDq_plV6ZtkHNFzBoBHTVLa1ix-CAlS8yWvBtExub8OjhHrfB9V2jy8ocTXILV5amudAYhhYP55v9pR_t9iYxDKoGRFSbeFtphE4sRVQNhVTkB7fVyZkm90v8hIt-3mmJiY9iwby8iqdPpXHPe0r3r9VEhAYiT5bq3MS_iRKuPbx-BlD00xj7ZXSu2DVp3S0EMIuLJZWN8xOvwKTWWh6R1bE75m3QYH7HV2k1X0sebxPNzLxK7sO2vgDECw3q_jkI5qaQuNNkh414FkDlGRWhoCyjK_3pQZdbs6BEMprKNdUBx27d3FDe13NRub-7Km9_zTggZi5Huup4BOMtSj7VkMKmgHzHvG_BiP5oCZUNKSBoO7vxfXE704sQ9FrG7hG1paNC3VRQ6q9v7fXfu17v20lEwDpmD7Ia0DTbPeKG_cR-ElTU33j3-YB2dKHtEf103P9DdYVJYxdXcLBG1po5HT0X7i08DN43v_3Na1_aT09lhDRmaoMGRuBsyyVaEr2Ammo4PLO09_Vsbhxx5akZEEzre121NYdfC_BAtr6qpVWB445V3EgqV9eVebNX2n_PFYzIIsbbPuKkpFuhqnJ5kJoeqDSbaLvl8C7gVd3hjfS0E_dJMdmDm6M8_MJSQsmRRR-uPUBc0peX6VUnx1qDx9EYBLYGcx5-nGi7tYYKdZiCgphYWxDfYVBOHH054Ql87GuH2bUxG0OjzBDCamKVga07Au1QWLpr4xWBuyv2GyfNtDTC7tcSyigsT9E6Yg3g2t7kNOMcT80EVvjVRf0xhHQj8_kZaQdr08wsfso6vUVhGwKtvMRG3-tN0UdrRvs7FYD0SqJfCtqM9FMNFg59ML4mBFi7WAeFjCoUORFxIfBomy8xnW2nX6MsXCIWSqEJpUM61egEzh-TkVF_K1oT_1GIpN7yh4aWPPRDlaYSucC-tjgotxy7keSu6HkZYQFUELs75R2xO4vjZJC-Tt7tNaFHAgUc2NqHT8n19y-W87eB-Cwu7h-Wiiiwky02DplAtqYQOFN8gx1cfjhV1YFBaLCXB_WJORNfI2xQ5urGOakN145tOBhS04d1qXMLcgdfHTys6pKnVtJw0IfZIjJRHMomSHSXZH9tBHLQpemphTDclx_ZjZ2b-z8ERO7WYgE5SWdW7cgE8r_XbblPLyorx1WQmDoS2yYWVOCbSZomZdR25rTgR8yvYQBmpNmdbQUsNJ-vCUEFb8dPeOy2GvIBGZkjWumcDMbjV1Fxa18Sa_uRrCjettM6bixwe0nOrh2Q_vrsmLd0aznQitT1RVFdrFDro0k1TQ8P0yDwej7GSQEJjwJJYNQKm5eVQZv7bvurQcw86rY7-v5caHOpzp6pCLJaxKw_GO95COpUST2tIbiY34ut2bF_hxsQKB5G2vmRub8-IdTcpqsXJVXr09r7K5m0vBpyTyqKdIDHNcJuBQxmKpAkmwSnwZh6FFlNCS0M3CPTVfXJ8wMn7HDaxnYxISEfxk2gZH6T5xGpd4ue7_fFBFBq4hOCngh8D5VaQuFM2qqk9E_AtzFcBuDlidXVbHjnFznlmeOX34jwepY-STE910YRINjK44dNFcN3Zh_nVXiIruo3RlIA0IdRapZpwJqaOKhneE8dzC5kVeUxiRAY_xsJVc0_yx90oM6mpm21vQNmOdRcwhO_oTm7SyrHa6Xc1MIap_5SjPzrNV42dWBaWhzUF-RLlXBS8DmB9XGoYA_LQYWBZ4MT4Kxe1nahMcHnIq2wvE6nC_mtP9ea11vV_Fwmik_KWmyEY1kikn2Uai6o9FdUC0WA4rC5iX0ZiCyjm0mxWR_fP7BslsreYS05F8YF5MBC6ztxnVMLEnOjyZDUCjrbRmaADB-xtvLRIFqmVKw_lFPxlF7IBWmJnZBoOytJbcln64vg)

вывод плана с nested loop

Скорее стоит отметить иную деталь, что nested loop обычно более эффективен, когда одна таблица меньше другой. **Скорее такая формулировка более корректна.** Это связано с тем, что внешний цикл проходит по меньшей таблице, а внутренний - по большей, что делает его менее ресурсоемким.

### Остальные SQL-примеры под merge & hash joins

Для демонстрации merge/hash joins, нам с вами потребуется подготовить пару таблиц со вставкой данных по случайным значениям. Выполните следующий SQL-код:

![к сожалению, Дзен перестал поддерживать функционал вставки кода, на момент написания статьи, поэтому приходится вставлять картинку, а снизу ссылку на код в GitHub Gist](https://avatars.dzeninfra.ru/get-zen_doc/271828/pub_66e996159f73b700d775c41c_66e99b04c6e7f97b1a6f75b1/scale_1200)

к сожалению, Дзен перестал поддерживать функционал вставки кода, на момент написания статьи, поэтому приходится вставлять картинку, а снизу ссылку на код в GitHub Gist

тестовые таблицы для демонстрации merge/hash joins

После создания таблиц и вставки данных в них, проверьте что данные появились:

> SELECT \* FROM public.test\_join\_tbl1 LIMIT 10 OFFSET 0;  
> SELECT \* FROM public.test\_join\_tbl2 LIMIT 10 OFFSET 0;

Проверьте вывод:

![-4](https://avatars.dzeninfra.ru/get-zen_doc/271828/pub_66e996159f73b700d775c41c_66e99c66229d6e2162133e30/scale_1200)

Теперь давайте исполним запрос, который выведет нам hash join:

![к сожалению, Дзен перестал поддерживать функционал вставки кода, на момент написания статьи, поэтому приходится вставлять картинку, а снизу ссылку на код в GitHub Gist](https://avatars.dzeninfra.ru/get-zen_doc/271828/pub_66e996159f73b700d775c41c_66e99e7448b8c60032e4d0b8/scale_1200)

к сожалению, Дзен перестал поддерживать функционал вставки кода, на момент написания статьи, поэтому приходится вставлять картинку, а снизу ссылку на код в GitHub Gist

запрос, который выведет hash join

**Важное!** Стоит упомянуть, что если вы тестируете этот SQL-код на PostgreSQL с достаточно увеличенным значением work\_mem (*к примеру 128-256 Мб*), или если сервис РСУБД "простаивает", т.е. его никто активно не использует в данный момент, то планировщик может выбрать и merge join, потому что ему хватит свободной памяти для того, чтобы предварительно отсортировать данные для merge join операции. В таком случае, понизьте значение work\_mem до минимального:

> SET work\_mem = '64kB';

и можете увеличить кол-во генерируемых данных через generates\_series(). Потом повторите SQL-запрос с JOIN. Почему так? **Здесь нельзя однозначно дать точный ответ**, т.к. это зависит от вычислительных харакетристик вашей среды, где будет проходить тестирование, так и от настроек самого PostgreSQL сервиса (*тот же упомянутый work\_mem*), т.к. планировщик будет определить наилучший алгоритм для исполнения запроса - в текущий момент времени, учитывая количество свободных ресурсов и количество данных используемых в запросе. Если же все ресурсы - свободны и нет параллельной нагрузки, то может выбраться merge join алгоритм. При реальном "боевом" использовании, РСУБД сервис нагружается параллельными запросами из-за чего становится более явным преимущество использования индексов для примера с merge/hash join.

**Вернемся к выводу плана:**

![вывод плана с hash join](https://avatars.dzeninfra.ru/get-zen_doc/271828/pub_66e996159f73b700d775c41c_66e99fec4ae1232c7ebed73d/scale_1200)

вывод плана с hash join

Даже если мы добавим сортировку в SQL-код:

> ORDER BY tbl1.id, tbl2.id;

  
То hash join останется в выводе:

![-7](https://avatars.dzeninfra.ru/get-zen_doc/271828/pub_66e996159f73b700d775c41c_66e9a11c4ae1232c7ec0c1dc/scale_1200)

### Что нужно для того, чтобы получился merge join?

  
Для этого нужно добавить индексы:

> CREATE INDEX idx\_t1\_id ON test\_join\_tbl1(id);  
> CREATE INDEX idx\_t2\_id ON test\_join\_tbl2(id);

  
И повторить запрос с ORDER BY частью:

![к сожалению, Дзен перестал поддерживать функционал вставки кода, на момент написания статьи, поэтому приходится вставлять картинку, а снизу ссылку на код в GitHub Gist](https://avatars.dzeninfra.ru/get-zen_doc/271828/pub_66e996159f73b700d775c41c_66e9a2652c6e9208c598b198/scale_1200)

к сожалению, Дзен перестал поддерживать функционал вставки кода, на момент написания статьи, поэтому приходится вставлять картинку, а снизу ссылку на код в GitHub Gist

создаем индексы и выводим merge join

В итоге получится merge join при выводе:

![-9](https://avatars.dzeninfra.ru/get-zen_doc/271828/pub_66e996159f73b700d775c41c_66e9a3b770b40e1ebab25534/scale_1200)

Вот, мы с вами и добились вывода всех 3-ех типов физических соединений: merge join, hash join и nested loop. Стоит упомнить еще, что вы можете отключить типы соединений используя следующие команды:

> SET enable\_hashjoin = off;  
> SET enable\_mergejoin = off;  
> SET enable\_nestloop = off;

  
Но на самом деле, вы не сможете полностью отключить использование, к примеру, nested loop для соединений:) В некоторых случаях, nested loop может быть единственным доступным методом соединения, особенно когда условие соединения не предполагает точное совпадение. Например использование операторов сравнения <, <=, >, >=

### Некоторые детали про hash join

  
**Сначала про сам алгоритм...**

Стоит упомянуть, что hash join, который реализован в PostgreSQL является не совсем классическим, когда используются build & probe фазы (компановка и проба). PostgreSQL использует гибридный hash join, о чем пишут сами разработчики PostgreSQL: [https://github.com/postgres/postgres/blob/master/src/backend/executor/nodeHashjoin.c](https://dzen.ru/away?to=https%3A%2F%2Fgithub.com%2Fpostgres%2Fpostgres%2Fblob%2Fmaster%2Fsrc%2Fbackend%2Fexecutor%2FnodeHashjoin.c)

> This is based on the "hybrid hash join" algorithm described shortly in the [https://en.wikipedia.org/wiki/Hash\_join#Hybrid\_hash\_join](https://dzen.ru/away?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FHash_join%23Hybrid_hash_join) and in detail in the referenced paper: "An Adaptive Hash Join Algorithm for Multiuser Environments" Hansjörg Zeller; Jim Gray (1990). Proceedings of the 16th VLDB conference. Brisbane: 186–197.

  
Если вкратце, то это комбинация классического алгоритма hash join и grace hash join.

**Влияние work\_mem**

При выводе плана с hash join можно увидеть такую строчку в планировщике:

> “Buckets: 1048576 Batches: 1 Memory Usage: 58583kB”

  
**Buckets** \- это количество ячеек в хеш-таблице, которые PostgreSQL использует для распределения данных из одной из таблиц.

**Batches** \- это количество партий в которых PostgreSQL разделяет данные, если хеш-таблица слишком велика для размещения в памяти.

Часть из вывода планировщика указывает на то, что все данные хеш-таблицы поместились в ОЗУ целиком и не было необходимости разбивать данные на несколько партий. Так же напомню, что хэш-таблица строится на базе одной таблицы при операции соединения, которая более меньшая по объемам памяти. Поэтому данный вывод показывает статистику именно относительно меньшей таблицы.

Но что будет, если мы установим более меньшее значение work\_mem?

> SET work\_mem = '64kB';  
> SHOW work\_mem;

  
И повторим запрос без ORDER BY части, чтобы увидеть вывод hash join:

![к сожалению, Дзен перестал поддерживать функционал вставки кода, на момент написания статьи, поэтому приходится вставлять картинку, а снизу ссылку на код в GitHub Gist](https://avatars.dzeninfra.ru/get-zen_doc/271828/pub_66e996159f73b700d775c41c_66e9a611b6871a7cc2b613e8/scale_1200)

к сожалению, Дзен перестал поддерживать функционал вставки кода, на момент написания статьи, поэтому приходится вставлять картинку, а снизу ссылку на код в GitHub Gist

запрос, который выведет hash join

Вывод будет следующим:

![-11](https://avatars.dzeninfra.ru/get-zen_doc/271828/pub_66e996159f73b700d775c41c_66e9a6d5e910864e65f7af19/scale_1200)

Сразу видно, что количество buckets и batches стали иным. Так что, видно линейное влияние work\_mem параметра. Здесь, так же, стоит упомянуть и про другой параметр, hash\_mem\_multiplier. Этот параметр служит для определения максимального объема памяти, который может быть выделен для операций хэширования.

Итоговый объем определяется как произведение значения work\_mem и коэффициента hash\_mem\_multiplier. По умолчанию hash\_mem\_multiplier равен 2.0, что означает, что для операций хэширования доступно в два раза больше памяти, чем базовое значение work\_mem.

Лучше увеличить значение hash\_mem\_multiplier, если при выполнении запросов часто наблюдается сбрасывание данных на диск, а простое увеличение work\_mem приводит к ошибкам нехватки памяти.

**Постскриптум:**

- при написании статьи использовалась следующая версия PostgreSQL v16.3
- к сожалению, Дзен перестал поддерживать функционал вставки исходного программного кода с syntax highlighting, на момент написания данной статьи. Про схожие проблемы рассказывают и другие Дзен-авторы: https://dzen.ru/a/XtUGaNtXXh9qY-gb поэтому вместо исходного кода, Вы видите комбинацию из картинки и ссылки на SQL-код в GitHub Gist. Прошу прощения за неудобства!

![-12](https://avatars.dzeninfra.ru/get-zen_doc/271828/pub_66e996159f73b700d775c41c_66e999bd560a7772d0257256/scale_1200)

Реклама • 16+

Как стать Go разработчиком? Полный roadmap

Забирай бесплатный урок «Как получить оффер на Go в 300к+ в 2025 году»

lukyanov.tech

Перейти на сайт

Реклама

Вакансия: разработчик BIOS/BMC в YADRO!

Создавайте системное ПО YADRO с нуля!

careers.yadro.com

Перейти на сайт

Реклама • 16+

Бесплатный онлайн мастер-класс для HR

«Психология работы с ИИ». Обучение с сертификатом и бонусами. Успейте записаться!

hrconf.workhere.ru

Перейти на сайт

Реклама • 16+

Из задрота в Go-разработчики: 20+ материалов для свитча!

Отдаю готовую подборку к собеседованиям на Golang. Скорее забирай!

olezhek28.courses

Перейти на сайт

Реклама • 16+

Стань разработчиком AI-сотрудников

Онлайн-вебинар по программированию. Будь в курсе всех новинок из мира AI. Заходи!

neural-university.ru

Перейти на сайт

Реклама

Сколько стоит квартира в Ростове? Узнать

Поможем подобрать квартиру по вашим параметрам с выгодой до 1 026 000 ₽!

tochno-stolitcino.ru

Перейти на сайт

Реклама • 16+

Стань Senior Backend — Продвинутый курс Golang за 3999р

200 уроков — Полноценные API. Работа с Go allocator, Stack, Heap, GC и горутинами!

purpleschool.ru

Перейти на сайт

Реклама • 16+

Бесплатный курс по Java

Создайте крупный проект на Java. Бесплатный курс.

srs.myrusakov.ru

Перейти на сайт

Реклама

Курсы программирования для взрослых в Таганроге!

Обучение востребованным специальностям. Диплом, трудоустройство. Быстро и эффективно!

taganrog.top-academy.ru

Перейти на сайт

Взгляните на эти темы[Гаджеты и электроника](https://dzen.ru/topic/gadzhety-i-it)

[

Умные часы

](https://dzen.ru/topic/umnye-chasy)[

Найти тему

](https://dzen.ru/explore)[

Игровые консоли

](https://dzen.ru/topic/igrovye-konsoli)[

Камеры и фототехника

](https://dzen.ru/topic/kamery-i-fototekhnika)[

Колонки и аудиосистемы

](https://dzen.ru/topic/kolonki-i-audiosistemy)[

Графические планшеты

](https://dzen.ru/topic/graficheskie-planshety)[

VR-очки

](https://dzen.ru/topic/vr-ochki)[

Смартфоны

](https://dzen.ru/topic/smartfony)[

Планшеты

](https://dzen.ru/topic/planshety)[Ноутбуки](https://dzen.ru/topic/noutbuki)

[

Фитнес-трекеры

](https://dzen.ru/topic/fitnes-trekery)[

Наушники

](https://dzen.ru/topic/garnitury-i-naushniki)[

Электронные книги

](https://dzen.ru/topic/ehlektronnye-knigi)[

Xiaomi

](https://dzen.ru/topic/xiaomi)[

Телевизоры

](https://dzen.ru/topic/televizory)[

Умные колонки Apple

](https://dzen.ru/topic/umnye-kolonki-apple)[

Технологии будущего

](https://dzen.ru/topic/tekhnologii-budushchego)[

Ноутбуки Dell

](https://dzen.ru/topic/noutbuki-dell)[

Производственные технологии

](https://dzen.ru/topic/proizvodstvennye-texnologii)[

Технологии в СМИ

](https://dzen.ru/topic/tekhnologii-v-smi)

Рекомендуем почитать

4 минуты

[

Какие бывают JOIN?

](https://dzen.ru/a/ZtfSgiJxP3ymu5TP?from=feed&utm_referrer=https%3A%2F%2Fzen.yandex.com&integration=publishers_platform_yandex&place=article_related&secdata=CNz5zLaiMiBrUI4JagUBeGtsLIgB58S7lrLtvap7kAEA&clid=560&rid=2500908459.1751.1754290496554.16519&referrer_clid=228& "Какие бывают JOIN?")

На мой взгляд САМЫЙ часто задаваемый вопрос на собеседованиях тестировщику (его спросят 99,99%). Поэтому лучше в нем разобраться. В SQL существует несколько типов объединений (JOIN), которые позволяют объединять строки из двух или более таблиц на основе определенного условия. Основные типы JOIN включают: 1. INNER JOIN (Внутреннее объединение): В этом случае будут возвращены только те сотрудники, у которых есть соответствующий отдел. 2. LEFT JOIN (Левое внешнее объединение): В этом случае будут...

5 минут

[

Сравнение REST и SOAP

](https://dzen.ru/a/Zso-71dsiw1DLeki?from=feed&utm_referrer=https%3A%2F%2Fzen.yandex.com&integration=publishers_platform_yandex&place=article_related&secdata=COTOmZadMiBrUI4JagUBeGtsLIgB58S7lrLtvap7kAEA&clid=560&rid=2500908459.1751.1754290496554.16519&referrer_clid=228& "Сравнение REST и SOAP")

REST (Representational State Transfer) и SOAP (Simple Object Access Protocol) — это два широко используемых подхода для взаимодействия между клиентом и сервером в веб-сервисах. Каждый из них имеет свои особенности, преимущества и недостатки. REST в первую очередь ориентирован на использование стандартных протоколов Интернета, и основным из них является HTTP. Однако REST может работать и поверх других протоколов. Вот основные из них: SOAP — это более универсальный протокол обмена сообщениями, который может работать на основе различных транспортных протоколов...

4 минуты

[

Реляционные базы данных и язык SQL. Хранимые функции типа SQL в PostgreSQL

](https://dzen.ru/a/aI3QXDhLqEBvVJrO?from=feed&utm_referrer=https%3A%2F%2Fzen.yandex.com&integration=publishers_platform_yandex&place=article_related&secdata=COuHoNCGMyBsUI4JagQBAmwsiAHnxLuWsu29qnuQAQA%3D&clid=560&rid=2500908459.1751.1754290496554.16519&referrer_clid=228& "Реляционные базы данных и язык SQL. Хранимые функции типа SQL в PostgreSQL")

Доброго здоровья читателям моего канала programmer's notes. Поддерживаем мой канал. Сервер баз данных PostgreSQL позволяет создавать специальные объекты, называемые хранимые функции. Использование таких функций даёт возможность переносить часть кода на сторону сервера. В стандартной установке сервера можно создавать функции двух типов: SQL-функции, состоящие из обычных sql-запросов и и функции на алгоритмическом языке программирования plpgsql. Также существуют дополнительные библиотеки, которые,...

Реклама • 16+

Курс «Мидл Java-разработчик»

Современный Java: Virtual Threads, Records, Sealed Types, Pattern Matching

31 500 ₽

practicum.yandex.ru

Перейти на сайт

### Статьи и видео без рекламы

С подпиской Дзен Про

![](https://s3.dzeninfra.ru/zen/icons/dzen-pro/floating-stella.png)