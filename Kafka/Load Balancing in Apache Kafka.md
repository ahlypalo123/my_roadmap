---
title: "How We Solve Load Balancing Challenges in Apache Kafka"
source: "https://medium.com/agoda-engineering/how-we-solve-load-balancing-challenges-in-apache-kafka-8cd88fdad02b"
author:
  - "[[Agoda Engineering]]"
published: 2024-07-30
created: 2025-05-03
description: "Apache Kafka has become an essential tool in Agoda for building resilient and scalable architectures. Its inherently distributed nature facilitates seamless data streaming and processing, which is…"
tags:
  - "clippings"
---
Get unlimited access to the best of Medium for less than $1/week.[Become a member](https://medium.com/plans?source=upgrade_membership---post_top_nav_upsell-----------------------------------------)

[

Become a member

](https://medium.com/plans?source=upgrade_membership---post_top_nav_upsell-----------------------------------------)## [Agoda Engineering & Design](https://medium.com/agoda-engineering?source=post_page---publication_nav-9bca683764b5-8cd88fdad02b---------------------------------------)

[![Agoda Engineering & Design](https://miro.medium.com/v2/resize:fill:76:76/1*RlfBLA66c56WySDq2YMdcA.jpeg)](https://medium.com/agoda-engineering?source=post_page---post_publication_sidebar-9bca683764b5-8cd88fdad02b---------------------------------------)

Learn about how products are developed at Agoda, what is being done under the hood, from engineering to design, to provide users a seamless experience at [agoda.com](http://agoda.com/).

*by* [*Yifan Huang*](https://www.linkedin.com/in/yifan-huang-37900916a/)

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*nRG9hk22sYGhBJx6K9ufBw.png)

Apache Kafka has become an essential tool in Agoda for building resilient and scalable architectures. Its inherently distributed nature facilitates seamless data streaming and processing, which is essential for handling the extensive data transactions within Agoda. Every day, the platform manages the transfer of hundreds of terabytes of data across various supply systems. This massive flow of data is extensively processed, analyzed, and stored efficiently, making Kafka an essential tool for maintaining the processing flow.

[Kafka](https://kafka.apache.org/) achieves parallelism by utilizing [partitions](https://kafka.apache.org/intro) to distribute messages across multiple queues, enabling various processors to consume messages in parallel. However, it’s common for each message to have varying processing loads or for consumers to have different processing rates. Achieving timely message processing requires balancing the workload properly because load imbalances can lead to bottlenecks, latency issues, and overall system instability, leading to extra maintenance efforts or additional resource allocation.

![](https://miro.medium.com/v2/resize:fit:1050/0*ixosAmhBDsyBBhoS)

Figure 1 Demonstration of Kafka partitions

In this blog post, we will introduce the concept of Load Balancing for Kafka-based applications. We will also explore the issue of imbalance and discuss strategies for effectively addressing these challenges.

## Background Story

At Agoda, we ensure that we source the best price from many external suppliers and swiftly update our database cache with the most current offers available. For instance, a single supplier can provide 1.5M price updates and offer details in just one minute. Any delays or failures in reflecting these updates can lead to incorrect pricing and customer booking failures.

The diagram below illustrates a typical Kafka application setup within one of our supply systems: distributor is a component that consumes price updates from 3rd party systems and distributes processing jobs to downstream processor service in multiple data centers while the processor transforms, processes data, and delivers the updates to downstream services and, eventually, databases.

![](https://miro.medium.com/v2/resize:fit:1050/0*UJAyD0Rjau177u9-)

Figure 2: Diagram of a Supply System at Agoda (The logic within distributor and processing services is simplified for demonstration purposes.)

In Kafka, the partitioner and assignor strategy affect the message distribution.

- Partitioner: Round-robin strategy, sticky partitioning, etc.
- Assigner: Range Assigner, Round Robin Assigner, etc.
![](https://miro.medium.com/v2/resize:fit:1050/0*p9G4RA8K60NCR7Fq)

Figure 3: Demonstration of Different Partitioning Strategies from Conduktor

These strategies are designed to work under two primary assumptions:

(1) consumers possess identical processing capabilities, and  
(2) messages are equal in workload.

Under these assumptions, all partitioning strategies aim to distribute messages as evenly as possible. However, in practice, these assumptions often do not hold due to the following reasons:

## Challenge #1: Heterogeneous hardware

Agoda manages its private cloud for service deployment, leveraging [Kubernetes](https://kubernetes.io/) to enhance scalability ([*See this article for how we transitioned to Private Cloud*](https://medium.com/agoda-engineering/private-cloud-and-you-736d8d99a51e)) and to reduce operating costs. With Agoda’s private cloud deployment, the server hardware generation assigned to an application may vary.

Different server hardware generations perform differently, leading to differences in processing rates. For example, the benchmark of processing using different Hardware generations shows significant differences in performance:

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*B6svY0ZjYVy-uJ7ZA18Jtg.png)

Table 1: measurement of capacity in our production

## Challenge #2: Uneven workload for each Kafka message

Different messages may require a different set of processing steps. For instance, processing a message might involve calling a third-party HTTP endpoint, and different response sizes or latencies can affect the processing rate. Additionally, for applications that involve database operations, the latency of their database queries may fluctuate depending on query parameters, resulting in varying processing rates.

Historically, we used the Kafka Round-Robin partitioner and assigner to distribute the same number of messages to each partition and Pod.

The illustration below shows 12 messages arriving in a time window. Here, the producer publishes two messages to each of the six partitions in this topic. Consequently, each worker consumes data from 2 partitions, which means that each worker needs to process four messages.

![](https://miro.medium.com/v2/resize:fit:1050/0*VwPda5gNsHRL2tJV)

Figure 4: Demonstration of previous supply system using Round-Robin partitioner and Round-Robin Assigner to distribute messages. Each worker is assigned an identical number of messages.

## Overprovisioning Problem

Over-provisioning involves allocating more resources than are necessary to handle the expected peak workload efficiently. In the setup described earlier, we encountered issues with over-provisioning due to an inefficient distribution of the load. It’s important to note that “inefficient! = uneven.” While round-robin ensures a perfectly even distribution of messages, it does not guarantee a perfect distribution in terms of performance in this case.

Consider an example where heterogeneous hardware is deployed in the processor service.

Suppose we have the processing rate of high and low throughputs as 20 msg/s and 10 msg/s, respectively (as simplified from data in Table 1). With two faster processors and one slower processor, we expect a total capacity of 20+20+10 = 50 messages/s. However, we couldn’t reach this capacity when keeping a Round-Robin distribution of messages. The illustration below shows what happens if traffic consistently hits 50 messages per second.

![](https://miro.medium.com/v2/resize:fit:1050/0*P-Qa3gyPgXtIeZMx)

Figure 5: If incoming traffic keeps at 50 messages/s, the slow processor cannot handle the load of ⅓ of overall messages, leading to built-up latency. To avoid high latency, additional resources are added to this system to maintain processing SLAs.

As we can see from this example, our processor service can only accept a maximum of 30 messages at a time to prevent lag and ensure timely delivery of updates.

In such a case, to actually process 50 messages/second, we have to scale up to 5 machines in total to guarantee a timely processing of all messages. We would overprovision an extra two machines to this system because of this inappropriate distribution logic (66.7% overprovisioning).

Overprovisioning can occur when the workload for each message is not identical. For example, some hotels may offer 100x more rates than others due to their complex pricing system and special discounts. Consequently, the workload for processing these larger responses can substantially increase for some consumers, causing delays in message handling for that consumer while other consumers remain idle. Such variations in the workload can lead to unnecessary message delays.

As both problems above are present in our supply system, an “unlucky” partition might be highly delayed, resulting in poor end-to-end (E2E) latency in the worst scenarios (99th percentile latency). To address these challenges, manual mitigation or scaling-up must be done to maintain the service level agreement (SLA) of the supply system, which in turn can increase maintenance efforts and hardware costs.

![](https://miro.medium.com/v2/resize:fit:701/0*rzvGCY35AMOOf0Ee)

Figure 6: Example of End-to-End (E2E) latency of 2 pods deployed in the same service. Slow processing pod negatively affects the overall performance.

## Static Balancing Solutions

Here are some solutions to balance load using a static configuration.

## Deployment on Identical Pods

You might consider controlling the types of hardware used in service deployments to mitigate issues. This approach is feasible if you are deploying services on virtual machines and possess ample resources with identical-performing hardware.

However, this strategy is usually not recommended in a private cloud environment due to decreased cost-effectiveness and flexibility, mainly because upgrading all existing hardware simultaneously can be challenging. If it is a good fit for your case, [Kubernetes affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) can be used to assign pods to certain types of nodes.

## Weighted Load Balancing

If the capacity is predictable and stays static most of the time, assigning varying weights to different consumers can help maximize the use of available resources. For instance, after giving higher weights to better-performing consumers, we can also route more traffic to those consumers.

## Lag-Aware: Act responsively

While we can estimate the capacities and workloads of messages to devise a static rule to determine a weighted load balancing strategy (refer to the section on Static Balancing Solutions), this approach may not always be feasible in a real production environment due to several factors:

- Messages are not uniform in workload, making it challenging to estimate machine capacity.
- Dependencies (such as network and 3rd party connections) are unstable, sometimes leading to changing capacities in the actual processing.
- The system frequently adds new features, adding extra maintenance efforts to keep the weights updated.

To address these issues, we implemented a lag-aware mechanism in the system to dynamically monitor current lags in each partition and respond accordingly based on current traffic conditions.

Let’s review the setup of the supply system, where we introduce two options for incorporating lag awareness:

1. **Lag-aware Producer**: These producers utilize dynamic partitioning logic that considers the lag information of the target topic.
2. **Lag-aware Consumer**: These consumers are designed to monitor current lags and can unsubscribe themselves to trigger a rebalancing of the load if necessary. Typically, a custom rebalance strategy can be adopted to adjust partition assignments.
![](https://miro.medium.com/v2/resize:fit:1050/0*2k3TPPJR7YzErg4D)

Figure 7: This diagram illustrates the high-level concepts of lag-aware producers and consumers. Both strategies aim to provide a responsive reaction to lags by incorporating lag awareness into different components.

## Lag-aware Producer

**Use case:** Lag-aware producers offer a straightforward implementation but are unsuitable for every use case. Here are instances where lag-aware producers should not be used:

- **Pure consumer application**: your application does not control message production.
- **Multiple Consumer Groups:** When produced messages are consumed by more than one consumer group, lag-aware producers may generate an unnecessarily skewed load for other consumer groups since lags are information specific to one consumer group only.

The supply system has an internal producer that publishes task messages to its processor. In such cases, producing messages based on the Kafka lag of a single consumer group has no side effects on other systems.

As shown in this diagram, Lag-aware producers use a customized algorithm to determine the traffic to each partition based on its lag. To reduce the number of calls to Kafka brokers, the system maintains an internal cache of lags rather than calling Kafka brokers before publishing each message.

Using lag data, a customized algorithm is designed to publish less traffic to partitions experiencing high lag and more to those with low lag to balance the workload on each partition. When lags are balanced and stable, this approach should ensure an even distribution of messages.

![](https://miro.medium.com/v2/resize:fit:893/0*Mg1lxKzMTy7LRAXT)

Figure 8: In this example, Partitions 4 and 6 have a much higher lag than other partitions. Less traffic should be sent to those partitions from an internal producer. Some algorithms to achieve this balance are shown in the section below.

## Algorithms

We have experimented with two algorithms in the supply system:

1. **Same-Queue Length Algorithm**

This algorithm considers each partition lag as the queue size of processing. After fetching the lag information, it publishes a proper number of messages in order to fill up the short queues.

This method is more suitable for skewed lag distribution due to heterogeneous hardwares, where high-performing pods (machines) are able to process faster in most of the cases.

![](https://miro.medium.com/v2/resize:fit:1050/0*zp-S1Y_GbIzbjCX4)

Figure 9: Demonstration of the same-queue-length algorithm. Initially, there are different lengths across different queues. This algorithm tries to produce a different number of messages to achieve the same queue length in all queues. Here, queue length is the same concept as Kafka lag, representing the number of messages that have not yet been processed.

**2\. Outlier Detection Algorithm**

This algorithm utilizes a statistical method to determine the upper outliers of all partitions and temporarily stops the publishing process for those slow outliers. For our specific needs, an IQR (interquartile range) & STD (standard deviation) outlier detection algorithm has been proposed. The flowchart of the algorithm is shown below.

![](https://miro.medium.com/v2/resize:fit:1050/0*pjkj5kF6aFcWwBwU)

- **Slow partitions:** (Closed) Message production to these partitions is stopped due to their existing lag.
- **OK partitions:** (Observation/Half-Open) To improve the performance of underperforming machines, an observation period is added when the system tries to promote slow partitions to good partitions. This observation stage can be optimized to a “Half-Open” state by producing only a small percentage of messages and observing. Half-Open is beneficial when the lag-fetching interval is relatively long, as it prevents situations where consumers are delayed waiting for incoming messages while updated lag data is yet to be queried.
- **Good partitions**: (Open) Publish as usual and distribute evenly to all good partitions.

## Results of Lag-aware producers

Prior to integrating the lag-aware feature, the system exhibited highly variable end-to-end latency spikes, particularly during specific hours of the day when traffic peaks occurred. This high latency, especially at the 99th percentile, was largely due to certain slow pods that received heavy tasks. After the deployment of the lag-aware feature, we were able to reduce the resources allocated to this system by 50% while maintaining SLA in a stable state without manual mitigation.

![](https://miro.medium.com/v2/resize:fit:1050/0*Zo8AuF_oHv7riQVY)

Previously, when examining Kafka lag by partition metrics, we noted that only a few partitions (5 out of 40) exhibited significant lag, with the remainder showing minimal to no lag. After the deployment of the new feature, we observed no significant high lag in the partitions.

![](https://miro.medium.com/v2/resize:fit:1050/0*tcpDFg7uR4R50G0B)

## Lag-aware Consumer

Lag-aware consumers are used when lag-aware producers are not applicable, such as when multiple consumer groups subscribe to the same topic. In these cases, the producer process impacts more than one service, rendering lag-aware producers ineffective.

Since some applications require load balancing but are not suitable for lag-aware producers, we began experimenting with lag-aware consumers to apply this dynamic load-balancing strategy more broadly. This is still an ongoing experiment.

Here is a brief introduction to the concepts involved. In a downstream service like the Processor service, an instance experiencing high lag can proactively unsubscribe from the topic to trigger a rebalance. During rebalance, a customized Assigner can be used to balance partitions across all consumer instances.

Historically, it is very expensive to trigger a rebalance because eager rebalance stops all processing in a consumer group. However, the introduction of [incremental cooperative rebalance protocol](https://www.confluent.io/blog/incremental-cooperative-rebalancing-in-kafka/) in Kafka 2.4 has minimized the performance impact, allowing for more frequent rebalances to better distribute the load on each partition.

To enhance flexibility in reassignment, the number of partitions should be greater than the number of workers. This ratio should vary based on the application, with the assumption that a worker can handle at least the load from one partition to avoid starvation.

![](https://miro.medium.com/v2/resize:fit:1050/0*68P7QtdFGeIzwZSs)

In this example, worker 3 is running on slow hardware, causing higher lags in partitions 5 and 6. Consequently, worker 3 may proactively unsubscribe from the topic to trigger a rebalance and redistribute the partitions more effectively. A custom Assignor should be implemented to reassign the partitions based on machine metrics and lag information in this example.

## Cluster-Level Load Balancing

Technically, if a Kafka cluster generates an unevenly distributed load, load balancing can be implemented at the Kafka cluster level based on the nature of the messages. For example, Agoda has a dedicated cluster [Hadin for Hadoop data delivery](https://medium.com/agoda-engineering/ingesting-petabytes-of-data-per-week-into-hadoop-from-kafka-457718cc308c), which uses data byte rate to determine partition assignments. For more details, see [*How our data scientists’ petabytes of data is ingested into Hadoop from Kafka*](https://medium.com/agoda-engineering/ingesting-petabytes-of-data-per-week-into-hadoop-from-kafka-457718cc308c)*.*

## Conclusion

In this blog, we explored methods to address load imbalance in Kafka applications. Specifically, we focused on two dynamic approaches: lag-aware producers and consumers. A lag-aware producer requires a dedicated producer for a specific topic to optimize traffic for one consumer group. In contrast, lag-aware consumers are useful when there is no dedicated producer, as they balance the load by triggering custom rebalances within a consumer group to redistribute the load.

In summary, implementing load balancing in Kafka can significantly enhance service performance by efficiently distributing the workload across available resources. However, choosing the right implementation and algorithm tailored to your application’s specific requirements for optimal results is crucial. Since these use cases are common and some algorithms can be reused in other applications, we plan to develop a general library to facilitate easy onboarding for other applications.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*qKAAoep6HRc8EBXkqWLK5g.jpeg)

[![Agoda Engineering & Design](https://miro.medium.com/v2/resize:fill:96:96/1*RlfBLA66c56WySDq2YMdcA.jpeg)](https://medium.com/agoda-engineering?source=post_page---post_publication_info--8cd88fdad02b---------------------------------------)

[![Agoda Engineering & Design](https://miro.medium.com/v2/resize:fill:128:128/1*RlfBLA66c56WySDq2YMdcA.jpeg)](https://medium.com/agoda-engineering?source=post_page---post_publication_info--8cd88fdad02b---------------------------------------)

[Last published Apr 24, 2025](https://medium.com/agoda-engineering/tracking-a-java-memory-leak-how-daemon-threads-in-pyroscope-nearly-crashed-our-service-3cf42379b9c6?source=post_page---post_publication_info--8cd88fdad02b---------------------------------------)

Learn about how products are developed at Agoda, what is being done under the hood, from engineering to design, to provide users a seamless experience at [agoda.com](http://agoda.com/).

Learn more about how we build products at Agoda and what is being done under the hood to provide users with a seamless experience at [agoda.com](http://agoda.com/).

## Responses (13)

Ahlypalo

What are your thoughts?  

```c
Great article.There is a CNCF tool for queue depth based autoscaling, known as KEDA. It is very helpful in such kind of scenarios
```

26

```c
Insightful article.
```

5

```c
Great article but basically, what you want to do is to implement a queuing system and distribute the oldest message to a consumer becoming available. Kafka isn't really a good fit for this use case, is it? There are alternatives like RabbitMQ or…
```

4

## More from Agoda Engineering and Agoda Engineering & Design

## Recommended from Medium

[

See more recommendations

](https://medium.com/?source=post_page---read_next_recirc--8cd88fdad02b---------------------------------------)