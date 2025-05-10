---
title: "Глубокое погружение в Java Memory Model"
source: "https://habr.com/ru/articles/685518/"
author:
  - "[[Хабр]]"
published: 2022-08-30
created: 2025-04-27
description: "Я провел в изучении JMM много часов и теперь делюсь с вами знаниями в простой и понятной форме. В этой статье мы подробно разберем Java Memory Model (JMM) и применим полученные знания на практике. Да,..."
tags:
  - "clippings"
---
![](https://habrastorage.org/r/w1560/webt/eb/nm/iu/ebnmiu3gfh1rn74kqsnvqrv2rnm.png)

  

Я провел в изучении JMM много часов и теперь делюсь с вами знаниями в простой и понятной форме.

  

В этой статье мы подробно разберем Java Memory Model (JMM) и применим полученные знания на практике. Да, в интернете накопилось достаточно много информации про JMM/happens-before, и, кажется, что очередную статью про такую заезженную тему можно пропускать мимо. Однако я постараюсь дать вам намного большее и глубокое понимание JMM, чем большинство информации в интернете. После прочтения этой статьи вы будете уверенно рассуждать о таких вещах как memory ordering, data race и happens-before. JMM — сложная тема и не стоит верить мне на слово, поэтому большинство моих утверждений подтверждается цитатами из спеки, дизассемблером и jcstress тестами.

[[1. Введение. контекст]]
[[2. Введение. JMM]]
[[3. Введение. Memory Ordering]]
[[4. Введение. Sequential Consistency]]
[[5. Введение. Sequential Consistency-Data Race Free]]
[[6. Введение. Sequential Consistency. Why]]
[[7. JMM. Synchronization Order]]
[[8. JMM. Happens-before. теория]]
[[9. JMM. Happens-before. практика]]
[[10. Safe Publication, RA semantics]]
[[11. Happens-before is about actions]]
[[12. Cache Coherence]]
[[13. Eventual Visibility]]
[[14. Memory Barriers]]
[[15. Как JMM обеспечивает консистентный memory order. подводим итоги]]
[[16. JMM. Atomicity]]
[[17. JMM. final fields]]
[[18. Benign data races]]
## Заключение

  

Надеюсь, данная статья дала вам некоторое понимание JMM, а полученные знания помогут вам писать безопасные и корректные многопоточные программы.

  

Хотя я и привёл здесь много низкоуровневой информации, но на самом деле запоминать такие детали совершенно не обязательно — я лишь хотел дать вам более глубокое понимание того, что происходит под капотом JMM. Просто пользуйтесь предоставленными примитивами синхронизации, а JMM сделает все за вас, ведь она создана как раз с той целью, чтобы скрыть, абстрагировать нижние уровни и предоставить гарантии, избавляющие вас от проблем memory reordering.

  

Пользуйтесь JMM, и да пребудет с вами thread safety.

  

P.S: и запомните: **data races are evil**.

  

---

  

## Ресурсы

  

Обратите внимание на репозиторий в поддержку данной статьи — [https://github.com/blinky-z/JmmArticleHabr](https://github.com/blinky-z/JmmArticleHabr). Там вы сможете найти еще больше, не включенных в статью jcstress тестов и дизассемблированных программ, а также инструкции и результаты запуска тестов и дизассемблера на x86/arm64.

  

Основы:

  
- [Memory model](https://en.wikipedia.org/wiki/Memory_model_\(programming\)), Wikipedia
- [Memory ordering](https://en.wikipedia.org/wiki/Memory_ordering), Wikipedia
- [JLS §17.4. Memory Model](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html#jls-17.4), спецификация
- [The Java Memory Model](http://www.cs.umd.edu/~pugh/java/memoryModel/index.html) — "домашняя страница" JMM, содержащая кучу полезной информации
- [The JSR-133 Cookbook](https://gee.cs.oswego.edu/dl/jmm/cookbook.html) — неофициальный гайдлайн по имплементации JMM для разработчиков JDK, Doug Lea
- [Memory Barriers — a Hardware View for Software Hackers](https://raw.githubusercontent.com/tpn/pdfs/master/Memory%20Barriers%20-%20a%20Hardware%20View%20for%20Software%20Hackers%20\(July%2023%2C%202010\).pdf) — крайне подробный обзор работы CPU, кэша и барьеров памяти
- [How does memory reordering help processors and compilers?](https://stackoverflow.com/questions/37725497/how-does-memory-reordering-help-processors-and-compilers/37739933#37739933), StackOverflow
  

Memory Model:

  
- [Memory Consistency Models: A Tutorial](https://www.cs.utexas.edu/~bornholt/post/memory-models.html), blog post
- [Understanding the Java Memory Model](https://abailly.github.io/posts/jmm.html), blog post
- [Sequential consistency](https://en.wikipedia.org/wiki/Sequential_consistency), Wikipedia
- [The Synchronizes-With Relation](https://preshing.com/20130823/the-synchronizes-with-relation/), blog post
- [The Happens-Before Relation](https://preshing.com/20130702/the-happens-before-relation/), blog post
- [Acquire and Release Semantics](https://preshing.com/20120913/acquire-and-release-semantics/), blog post
- [An Introduction to Lock-Free Programming](https://preshing.com/20120612/an-introduction-to-lock-free-programming/#sequential-consistency), blog post
- [Memory Reordering Caught in the Act](https://preshing.com/20120515/memory-reordering-caught-in-the-act/), blog post
- [Sequential Consistency & Total Store Order](https://www.cis.upenn.edu/~devietti/classes/cis601-spring2016/sc_tso.pdf), slides
  

Блог Алексея Шипилева — это целая кладезь знаний про JMM и не только. Крайне советую прочитать следующие его статьи:

  
- [Java Memory Model Pragmatics](https://shipilev.net/blog/2014/jmm-pragmatics/), blog post
- [Safe Publication and Safe Initialization in Java](https://shipilev.net/blog/2014/safe-public-construction/), blog post
- [Close Encounters of The Java Memory Model Kind](https://shipilev.net/blog/2016/close-encounters-of-jmm-kind/), blog post
- [All Fields Are Final](https://shipilev.net/blog/2014/all-fields-are-final/), blog post
  

Compiler Memory Ordering:

  
- [Instruction scheduling](https://en.wikipedia.org/wiki/Instruction_scheduling), Wikipedia
- [Memory Ordering at Compile Time](https://preshing.com/20120625/memory-ordering-at-compile-time/), blog post
  

CPU Memory ordering/Memory Barrier:

  
- [Memory barrier](https://en.wikipedia.org/wiki/Memory_barrier), Wikipedia
- [Memory Barriers Are Like Source Control Operations](https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/), blog post
- [Weak vs. Strong Memory Models](https://preshing.com/20120930/weak-vs-strong-memory-models/), blog post
- [Memory Barriers/Fences](https://mechanical-sympathy.blogspot.com/2011/07/memory-barriersfences.html?m=1), blog post
- [Linux kernel memory barriers](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/memory-barriers.txt), Linux kernel docs
- [Сводная таблица возможных переупорядочиваний среди различных микроархитектур](https://en.wikipedia.org/wiki/Memory_ordering#Runtime_memory_ordering)
- Intel (x86):  
	- [Intel Memory Ordering](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-system-programming-manual-325384.pdf#G13.14501), Intel docs
	- [Does an x86 CPU reorder instructions?](https://stackoverflow.com/questions/50307693/does-an-x86-cpu-reorder-instructions), StackOverflow
	- [Making sense of Memory Barriers](https://stackoverflow.com/questions/37798053/making-sense-of-memory-barriers), StackOverflow
	- [what is a store buffer?](https://stackoverflow.com/questions/11105827/what-is-a-store-buffer), StackOverflow
	- [Size of store buffers on Intel hardware? What exactly is a store buffer?](https://stackoverflow.com/questions/54876208/size-of-store-buffers-on-intel-hardware-what-exactly-is-a-store-buffer), StackOverflow
- ARM:  
	- [ARM Memory Ordering](https://developer.arm.com/documentation/den0024/a/Memory-Ordering?lang=en), ARM docs
  

CPU Cache:

  
- [CPU cache](https://en.wikipedia.org/wiki/CPU_cache), Wikipedia
- [Cache hierarchy](https://en.wikipedia.org/wiki/Cache_hierarchy), Wikipedia
- [Cache coherence](https://en.wikipedia.org/wiki/Cache_coherence), Wikipedia
- [MESI protocol](https://en.wikipedia.org/wiki/MESI_protocol), Wikipedia
- [CPU Cache Flushing Fallacy](https://mechanical-sympathy.blogspot.com/2013/02/cpu-cache-flushing-fallacy.html), blog post
- [Cache coherency primer](https://fgiesen.wordpress.com/2014/07/07/cache-coherency/), blog post
- [Memory barriers force cache coherency?](https://stackoverflow.com/questions/30958375/memory-barriers-force-cache-coherency), StackOverflow
  

Volatile:

  
- [Volatile](https://jpbempel.github.io/2012/10/09/volatile.html), blog post
- [Volatile and memory barriers](https://jpbempel.github.io/2015/05/26/volatile-and-memory-barriers.html), blog post
- [Java theory and practice: Managing volatility](http://web.archive.org/web/20210221170926/https://www.ibm.com/developerworks/java/library/j-jtp06197/), blog post by Brian Goetz
  

Книги:

  
- [A Primer on Memory Consistency and Cache Coherence](https://www.morganclaypool.com/doi/10.2200/S00962ED2V01Y201910CAC049)
- [Java Concurrency in Practice](https://www.amazon.com/Java-Concurrency-Practice-Brian-Goetz/dp/0321349601)

+109

Будущее здесь: лучшее

Годнота из блогов компаний

Открыт приём в Школу анализа данных

Электроны зажигают!

Сезону Open source нужны твои истории

Дух горной прохлады

Космос ждёт

[Java разработчик](https://career.habr.com/vacancies/java_developer)

204 вакансии