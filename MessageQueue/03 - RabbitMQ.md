### 简介

### 核心概念
RabbitMQ 整体上是一个生产者与消费者模型，主要负责接收、存储和转发消息，RabbitMQ 整体上是一个生产者与消费者模型，主要负责接收、存储和转发消息
RabbitMQ 的整体模型架构如下：![[Pasted image 20240401155437.png]]
#### Producer、Consumer
Producer：生产消息
Consumer：消费者

消息一般由 2 部分组成：消息头和消息体。
消息体是不透明的，而消息头则是由一系列的可选属性组成，这些属性包括 routing-key（路由键）、priority（优先级）、delivery-mode（持久性存储）。Producer 把消息交由 RabbitMQ 后，RabbitMQ 会根据消息头把消息发送给感兴趣的 Consumer
#### Exchange
在 RabbitMQ 中，消息并不是直接被投递到 Queue 中，中间还必须经过 Exchange 这一层，Exchange 会把消息分配到对应的 Queue 中

Exchange 用来接收生产者发送的消息并将这些消息路由给服务器中的 Queue，如果路由不到，或许会返回给 Producer，或许会被直接丢弃。

Exchange 有 4 种类型，对应不同的路由策略：
1、direct
direct 类型的 Exchange 路由规则也很简单，它会把消息路由到那些 Bindingkey 与 RoutingKey 完全匹配的 Queue 中，常用在处理有优先级的任务，根据任务的优先级把消息发送到对应的队列，这样可以指派更多的资源去处理高优先级的队列
2、fanout
fanout 类型的 Exchange 路由规则非常简单，它会把所有发送到该 Exchange 的消息路由到所有与它绑定的 Queue 中，不需要做任何判断操作，所以 fanout 类型是所有的交换机类型里面速度最快的。fanout 类型常用来广播消息
3、topic
与 direct 类型的交换器相似，也是将消息路由到 BindingKey 和 RoutingKey 相匹配的队列中，但这里的匹配规则有些不同，它约定：
- RoutingKey 为一个点号“．”分隔的字符串（被点号“．”分隔开的每一段独立的字符串称为一个单词），如 “com.rabbitmq.client”、“java.util.concurrent”、“com.hidden.client”
- BindingKey 和 RoutingKey 一样也是点号“．”分隔的字符串
- BindingKey 中可以存在两种特殊字符串“*”和“#”，用于做模糊匹配，其中“*”用于匹配一个单词，“#”用于匹配多个单词(可以是零个)
![[Pasted image 20240401162921.png]]
4、headers
headers 类型的交换器性能会很差，而且也不实用，基本上不会看到它的存在

Exchange 示意图如下：![[Pasted image 20240401160154.png]]
Producer 将消息给交换器的时候，一般会指定一个 RoutingKey，用来指定这个消息的路由规则，而这个 RoutingKey 需要与 Exchange 类型和 BindingKey 联合使用才能最终生效

RabbitMQ 中通过 Binding 将 Exchange 与 Queue 关联起来，在绑定的时候一般会指定一个 BindingKey，这样 RabbitMQ 就知道如何正确将消息路由到队列了

一个 Binding 就是基于 RoutingKey 将 Exchange 和 Queue 连接起来的路由规则，所以可以将交换器理解成一个由绑定构成的路由表。Exchange 和 Queue 的绑定可以是多对多的关系

Binding 示意图：![[Pasted image 20240401160901.png]]

Producer 将消息发送给 Exchange 时，需要一个 RoutingKey，当 BindingKey 和 RoutingKey 相匹配时，消息会被路由到对应的 Queue 中。在绑定多个 Queue 到同一个 Exchange 时，这些绑定允许使用相同的 BindingKey。BindingKey 并不是在所有的情况下都生效，它依赖于交换器类型，比如 fanout 类型的交换器就被无视，而是将消息路由到所有绑定到该交换器的队列中
#### Queue
Queue 用来保存消息直到发送给消费者。它是消息的容器，也是消息的终点。一个消息可投入一个或多个队列。消息一直在对立里面，等待消费者连接到这个队列取走

RabbitMQ 中消息只能存储在 队列 中，这一点和 Kafka 这种消息中间件相反。Kafka 将消息存储在 topic 这个逻辑层面，而相应的队列逻辑只是 topic 实际存储文件中的位移表示。

多个消费者可以订阅同一个队列，这是队列中的消息会被平均分摊（Round-Robin，轮询）给多个消费者进行处理，而不是每个消费者都能收到所有的消息并处理，这样避免消息被重复消费

RabbitMQ 不支持队列层面的广播消费，如果有广播消费需求会很麻烦，不建议这样使用
#### Broker
对于 RabbitMQ 来说，一个 RabbitMQ Broker 可以简单地看作一个 RabbitMQ 服务节点，或者 RabbitMQ 服务实例，大多数情况下也可以将一个 RabbitMQ Broker 看作一台 RabbitMQ 服务器

Producer 将消息存入 RabbitMQ Broker，以及消费者从 Broker 中消费数据的整个流程![[Pasted image 20240401162520.png]]
### RabbitMQ 工作模式
RabbitMQ 有 6 种工作模式：
- Work Queues
- Publish/subscribe
- Routing
- Topics
- Header 模式
- RPC
#### Work queues
多个消费端消费同一个队列中的消息，队列采用轮询的方式将消息是平均发送给消费者![[Pasted image 20240401164112.png]]
特点：
1、一条消息只会被一个消费端接收
2、队列采用轮训的方式将消息平均发送给消费者
3、消费者在处理完某条消息后，才会收到下一条消息

#### Publish/subscribe
这种模式又称为发布订阅模式，相对于Work queues模式，该模式多了一个交换机，生产端先把消息发送到交换机，再由交换机把消息发送到绑定的队列中，每个绑定的队列都能收到由生产端发送的消息![[Pasted image 20240401164555.png]]
特点：
1、每个消费者监听自己的队列
2、生产者将消息发给broker，由交换机将消息转发到绑定此交换机的每个队列，每个绑定交换机的队列都将接收消息
3、此时的 exchange 类型为 fanout

#### Routing
Routing 模式又称路由模式，该种模式除了要绑定交换机外，发消息要指定routing key，队列通过通道绑定交换机的时候，需要指定 binding key，通过 routing key 和 binding key 的匹配发分发消息
![[Pasted image 20240401165147.png]]
特点：
1、每个消费者监听自己的队列，并设置 binding key
2、生产者将消息发给交换机，并指定消息的 routing key
3、exchange 类型为 direct

#### Topics
Topics 模式和Routing 路由模式最大的区别就是，Topics 模式发送消息和消费消息的时候是通过通配符去进行匹配的![[Pasted image 20240401165343.png]]
特点：
1、每个消费者监听自己的队列，并配置带通配符的 binding key
2、生产者将消息发给交换机，并指定消息的 routing key
3、exchange 类型为 topic

#### RPC
 RPC 即客户端远程调用服务端的方法 ，使用MQ可以实现RPC的异步调用，基于 Direct 交换机实现![[Pasted image 20240401165821.png]]
 特点：
 1、客户端即是生产者也是消费者，向RPC请求队列发送RPC调用消息，同时监听RPC响应队列
 2、服务端监听RPC请求队列的消息，收到消息后执行服务端的方法，得到方法返回的结果
 3、服务端将RPC方法 的结果发送到RPC响应队列
 4、客户端（RPC调用方）监听RPC响应队列，接收到RPC调用结果

### RabbitMQ 集群
集群有两种模型：
- 普通集群模式
- 镜像集群模式
##### 普通集群模式
意思就是在多台机器上启动多个 RabbitMQ 实例，每个机器启动一个。你创建的 queue，只会放在一个 RabbitMQ 实例上，但是每个实例都同步 queue 的元数据（元数据可以认为是 queue 的一些配置信息，通过元数据，可以找到 queue 所在实例）

你消费的时候，实际上如果连接到了另外一个实例，那么那个实例会从 queue 所在实例上拉取数据过来。这方案主要是提高吞吐量的，就是说让集群中多个节点来服务某个 queue 的读写操作
##### 镜像集群模式
这种模式，才是所谓的 RabbitMQ 的高可用模式。跟普通集群模式不一样的是，在镜像集群模式下，你创建的 queue，无论元数据还是 queue 里的消息都会存在于多个实例上，就是说，每个 RabbitMQ 节点都有这个 queue 的一个完整镜像，包含 queue 的全部数据的意思。然后每次你写消息到 queue 的时候，都会自动把消息同步到多个实例的 queue 上

RabbitMQ 有很好的管理控制台，就是在后台新增一个策略，这个策略是镜像集群模式的策略，指定的时候是可以要求数据同步到所有节点的，也可以要求同步到指定数量的节点，再次创建 queue 的时候，应用这个策略，就会自动将数据同步到其他的节点上去了

优点：这样的好处在于，你任何一个机器宕机了，没事儿，其它机器（节点）还包含了这个 queue 的完整数据，别的 consumer 都可以到其它节点上去消费数据

缺点：消息需要同步到所有机器上，导致网络带宽压力和消耗很重，RabbitMQ 一个 queue 的数据都是放在一个节点里的，镜像集群下，也是每个节点都放这个 queue 的完整数据