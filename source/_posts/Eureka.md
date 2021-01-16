---
title: Eureka
date: 2021-01-16 15:37:40
tags: Spring cloud
---

<img src="https://tva1.sinaimg.cn/large/008eGmZEgy1gmpl0xz090j31au0m6q4i.jpg" alt="image-20210116154626451" style="zoom:50%;" />

<center>图1 Eureka 体系结构图</center>

博客重点参考[该文](https://blog.csdn.net/jinjiniao1/article/details/100771613)，此外该文对Eureka的体系结构有更深层次地理解，也可以查看[该文](https://www.infoq.cn/article/jlDJQ*3wtN2PcqTDyokh)。

#### 1、Eureka核心概念

图1是Eureka的体系结构图，宏观上主要分为两个部分Server以及Client部分，其中Client又可分为Consumer以及Provider两部分下面将依照这个体系结构图介绍与之相关的一些概念。

+ **Eureka Server: 注册中心服务端**

  该部分主要提供三个功能

  - 服务注册：服务提供者启动时，会通过 Eureka Client 向 Eureka Server 注册信息，Eureka Server 会存储该服务的信息，Eureka Server 内部有二层缓存机制来维护整个注册表。

  + 提供注册表：服务消费者在调用服务时，如果 Eureka Client 没有缓存注册表的话，会从 Eureka Server 获取最新的注册表。
  + 同步状态： Eureka Client 通过注册、心跳机制和 Eureka Server 同步当前客户端的状态。

+ **Eureka Client：注册中心客户端**

  Eureka Client 会拉取、更新和缓存 Eureka Server 中的信息。因此当所有的 Eureka Server 节点都宕掉，服务消费者依然可以使用缓存中的信息找到服务提供者，但是当服务有更改的时候会出现信息不一致。

+ **Register：服务注册**

  服务的提供者，将自身注册到注册中心，服务提供者也是一个 Eureka Client。当 Eureka Client 向 Eureka Server 注册时，它提供自身的元数据，比如 IP 地址、端口，运行状况指示符 URL，主页等。

+ **Renew：服务续约**

  Eureka Client 会每隔 30 秒发送一次心跳来续约。 通过续约来告知 Eureka Server 该 Eureka Client 运行正常，没有出现问题。 默认情况下，如果 Eureka Server 在 90 秒内没有收到 Eureka Client 的续约，Server 端会将实例从其注册表中删除，此时间可配置，一般情况不建议更改。

  ```java
  # 服务续约任务的调用间隔时间，默认为30秒
  eureka.instance.lease-renewal-interval-in-seconds=30
  # 服务失效的时间，默认为90秒。
  eureka.instance.lease-expiration-duration-in-seconds=90
  ```

  *Eviction 服务剔除*

  当 Eureka Client 和 Eureka Server 不再有心跳时，Eureka Server 会将该服务实例从服务注册列表中删除，即服务剔除。

+ **Cancel：服务下线**

  Eureka Client 在程序关闭时向 Eureka Server 发送取消请求。 发送请求后，该客户端实例信息将从 Eureka Server 的实例注册表中删除。该下线请求不会自动完成，它需要调用以下内容：

  ```java
  DiscoveryManager.getInstance().shutdownComponent()；
  ```

+ **Get Registry：获取注册表信息**

  Eureka Client 从服务器获取注册表信息，并将其缓存在本地。客户端会使用该信息查找其他服务，从而进行远程调用。该注册列表信息定期（每30秒钟）更新一次。每次返回注册列表信息可能与 Eureka Client 的缓存信息不同，Eureka Client 自动处理。

  如果由于某种原因导致注册列表信息不能及时匹配，Eureka Client 则会重新获取整个注册表信息。 Eureka Server 缓存注册列表信息，整个注册表以及每个应用程序的信息进行了压缩，压缩内容和没有压缩的内容完全相同。Eureka Client 和 Eureka Server 可以使用 JSON/XML 格式进行通讯。在默认情况下 Eureka Client 使用压缩 JSON 格式来获取注册列表的信息。这里涉及两个需要重点配置的参数：

  ```java
  # 启用服务消费者从注册中心拉取服务列表的功能
  eureka.client.fetch-registry=true
  # 设置服务消费者从注册中心拉取服务列表的间隔
  eureka.client.registry-fetch-interval-seconds=30
  ```

+ **Remote Call：远程调用**

  当 Eureka Client 从注册中心获取到服务提供者信息后，就可以通过 Http 请求调用对应的服务；服务提供者有多个时，Eureka Client 客户端会通过 Ribbon 自动进行负载均衡。

  #### 2、自我保护机制

  默认情况下，如果 Eureka Server 在一定的 90s 内没有接收到某个微服务实例的心跳，会注销该实例。但是在微服务架构下服务之间通常都是跨进程调用，网络通信往往会面临着各种问题，比如微服务状态正常，网络分区故障，导致此实例被注销。固定时间内大量实例被注销，可能会严重威胁整个微服务架构的可用性。为了解决这个问题，Eureka 开发了自我保护机制，那么什么是自我保护机制呢？Eureka Server 在运行期间会去统计心跳失败比例在 15 分钟之内是否低于 85%，如果低于 85%，Eureka Server 即会进入自我保护机制。

  Eureka Server 进入自我保护机制，会出现以下几种情况：

  - Eureka 不再从注册列表中移除因为长时间没收到心跳而应该过期的服务
  - Eureka 仍然能够接受新服务的注册和查询请求，但是不会被同步到其它节点上(即保证当前节点依然可用)
  - 当网络稳定时，当前实例新的注册信息会被同步到其它节点中

  Eureka 自我保护机制是为了防止误杀服务而提供的一个机制。当个别客户端出现心跳失联时，则认为是客户端的问题，剔除掉客户端；当 Eureka 捕获到大量的心跳失败时，则认为可能是网络问题，进入自我保护机制；当客户端心跳恢复时，Eureka 会自动退出自我保护机制。

  如果在保护期内刚好这个服务提供者非正常下线了，此时服务消费者就会拿到一个无效的服务实例，即会调用失败。对于这个问题需要服务消费者端要有一些容错机制，如重试，断路器等。

  通过在 Eureka Server 配置如下参数，开启或者关闭保护机制，生产环境建议打开：

  ```java
  eureka.server.enable-self-preservation=true
  ```

  3.Eureka集群

  <img src="https://tva1.sinaimg.cn/large/008eGmZEgy1gmpsadl5oqj315q0ke464.jpg" alt="image-20210116195740723" style="zoom:50%;" />

<center>图2 Eureka集群</center>

从图中可以看出Server之前通过Replicate进行同步，Server之间是对等的，没有主从关系。因此，几个节点挂掉不会影响正常节点的工作，剩余的节点依然可以提供注册和查询服务。而 Eureka Client 在向某个 Eureka 注册时，如果发现连接失败，则会自动切换至其它节点。只要有一台 Eureka Server 还在，就能保证注册服务可用(保证可用性)，只不过查到的信息可能不是最新的(不保证强一致性)。Eureka使用Region以及Zone两个概念对集群进行分区:

- region：可以理解为地理上的不同区域，比如亚洲地区，中国区或者深圳等等。没有具体大小的限制。根据项目具体的情况，可以自行合理划分 region。

- zone：可以简单理解为 region 内的具体机房，比如说 region 划分为深圳，然后深圳有两个机房，就可以在此 region 之下划分出 zone1、zone2 两个 zone。

  Zone 内的Client优先和Zone内的Server进行同步，当Zone的Server挂掉之后才会从其他的Zone中获取信息。

#### 3.Eureka的工作流程

1. Eureka Server 启动成功，等待服务端注册。在启动过程中如果配置了集群，集群之间定时通过 Replicate 同步注册表，每个 Eureka Server 都存在独立完整的服务注册表信息；
2. Eureka Client 启动时根据配置的 Eureka Server 地址去注册中心注册服务；
3. Eureka Client 会每 30s 向 Eureka Server 发送一次心跳请求，证明客户端服务正常；
4. 当 Eureka Server 90s 内没有收到 Eureka Client 的心跳，注册中心则认为该节点失效，会注销该实例；
5. 单位时间内 Eureka Server 统计到有大量的 Eureka Client 没有上送心跳，则认为可能为网络异常，进入自我保护机制，不再剔除没有上送心跳的客户端；
6. 当 Eureka Client 心跳请求恢复正常之后，Eureka Server 自动退出自我保护模式；
7. Eureka Client 定时全量或者增量从注册中心获取服务注册表，并且将获取到的信息缓存到本地；
8. 服务调用时，Eureka Client 会先从本地缓存找寻调取的服务。如果获取不到，先从注册中心刷新注册表，再同步到本地缓存；
9. Eureka Client 获取到目标服务器信息，发起服务调用；
10. Eureka Client 程序关闭时向 Eureka Server 发送取消请求，Eureka Server 将实例从注册表中删除。

#### 4.Eureka数据存储

既然是服务注册中心，必然要存储服务的信息，我们知道 ZK 是将服务信息保存在树形节点上。而下面是 Eureka 的数据存储结构：

<img src="https://tva1.sinaimg.cn/large/008eGmZEgy1gmpt68ebvjj312y0u07ia.jpg" alt="image-20210116202814847" style="zoom:50%;" />

<center>图3 Eureka数据存储结构</center>

Eureka 的数据存储分了两层：**数据存储层**和**缓存层**。

Eureka Client 在拉取服务信息时，先从缓存层获取（相当于 Redis），如果获取不到，先把数据存储层的数据加载到缓存中（相当于 Mysql），再从缓存中获取。值得注意的是，数据存储层的数据结构是服务信息，而缓存中保存的是经过处理加工过的、可以直接传输到 Eureka Client 的数据结构。

Eureka 这样的数据结构设计是把内部的数据存储结构与对外的数据结构隔离开了，就像是我们平时在进行接口设计一样，对外输出的数据结构和数据库中的数据结构往往都是不一样的。

##### **4.1 数据存储层**

这里为什么说是存储层而不是持久层？因为 rigistry 本质上是一个双层的 ConcurrentHashMap，存储在内存中的。

- 第一层的key是`spring.application.name`，value是第二层ConcurrentHashMap；
- 第二层ConcurrentHashMap的key是服务的InstanceId，value是Lease对象，Lease对象包含了服务详情和服务治理相关的属性；

下面是对应的源码：

```java
private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry= new ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>();
```

##### 4.2 二级缓存层

Eureka 实现了二级缓存来保存即将要对外传输的服务信息，数据结构完全相同。

- 一级缓存：`ConcurrentHashMap<Key,Value> readOnlyCacheMap`，本质上是HashMap，无过期时间，保存服务信息的对外输出数据结构。
- 二级缓存：`LoadingCache<Key,Value> readWriteCacheMap`，本质上是guava的缓存，包含失效机制，保存服务信息的对外输出数据结构。

下面是对应的源码：

```java
private final ConcurrentMap<Key, ResponseCacheImpl.Value> readOnlyCacheMap = new ConcurrentHashMap();
private final LoadingCache<Key, ResponseCacheImpl.Value> readWriteCacheMap;
```

既然是缓存，那必然要有更新机制，来保证数据的一致性。下面是缓存的更新机制：

<img src="https://tva1.sinaimg.cn/large/008eGmZEgy1gmpttadzvoj31dc0mytft.jpg" alt="image-20210116205011663" style="zoom:50%;" />

<center>图4 缓存更新和删除机制</center>

更新机制包含删除和加载两个部分，上图黑色箭头表示删除缓存的动作，绿色表示加载或触发加载的动作。

**删除二级缓存：**

1. Eureka Client发送register、renew和cancel请求并更新registry注册表之后，删除二级缓存；
2. Eureka Server自身的Evict Task剔除服务后，删除二级缓存；
3. 二级缓存本身设置了guava的失效机制，隔一段时间后自己自动失效；

**加载二级缓存：**

1. Eureka Client发送getRegistry请求后，如果二级缓存中没有，就触发guava的load，即从registry中获取原始服务信息后进行处理加工，再加载到二级缓存中。
2. Eureka Server更新一级缓存的时候，如果二级缓存没有数据，也会触发guava的load。

**更新一级缓存：**

1. Eureka Server内置了一个TimerTask，定时将二级缓存中的数据同步到一级缓存（这个动作包括了删除和加载）。

#### 5.Zookeeper与Eureka对比

<img src="https://tva1.sinaimg.cn/large/008eGmZEgy1gmpsw8h03cj30x70u0n75.jpg" alt="image-20210116201824157" style="zoom: 25%;" />

<center>图5 CAP</center>

关于注册中心的解决方案，dubbo 支持了 Zookeeper、Redis、Multicast 和 Simple，官方推荐 Zookeeper。Spring Cloud 支持了 Zookeeper、Consul 和 Eureka，官方推荐 Eureka。两者之所以推荐不同的实现方式，原因在于组件的特点以及适用场景不同。简单来说：

- ZK的设计原则是CP，即强一致性和分区容错性。他保证数据的强一致性，但舍弃了可用性，**如果出现网络问题可能会影响ZK的选举，导致ZK注册中心的不可用**。

- Eureka的设计原则是AP，即可用性和分区容错性。他保证了注册中心的可用性，但舍弃了数据一致性，**各节点上的数据有可能是不一致的（会最终一致）**。

  |          |                   Zookeeper                    | Eureka                                                       |
  | :------: | :--------------------------------------------: | ------------------------------------------------------------ |
  | 设计原则 |                       CP                       | AP                                                           |
  |   优点   |                   数据强一致                   | 服务高可用                                                   |
  |   缺点   | 网络分区会影响Leader选举，超过阈值后集群不可用 | 服务节点间的数据可能不一致； Client-Server间的数据可能不一致； |
  | 适用场景 |        单机房集群，对数据一致性要求较高        | 云机房集群，跨越多机房部署；对注册中心服务可用性要求较高     |

#### 6.入门案例

https://github.com/zhichengshi/spring-demo/tree/master/eureka

