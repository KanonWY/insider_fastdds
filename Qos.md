## 1、Qos 概述
服务质量 (QoS) 用于指定服务的行为，允许用户定义每个实体的行为方式。为了增加系统的灵活性，QoS被分解为多个可以独立配置的QoS策略。然而，在某些情况下，多项政策可能会发生冲突。这些冲突通过 QoS 设置器函数返回的 ReturnCodes 通知给用户。  
每个 Qos 策略都有一个在 QosPolicyId_t 枚举器中定义的唯一 ID。此 ID 在某些状态实例中用于标识状态所引用的特定 Qos 策略。  
有些 QoS 策略是不可变的，这意味着只能在实体创建时或调用启用操作之前指定。  
每个 DDS 实体都有一组特定的 QoS 策略，这些策略可以是标准 QoS 策略、XTypes 扩展和 eProsima 扩展的组合。  

## 2、标准 Qos

### DealineQosPolicy
这个QoS（Quality of Service）策略在新样本的频率低于某个阈值时会发出警报。它对于需要定期更新数据的情况非常有用。
- 关联实体  
`Topic`、`DataReader`、`DataWriter`
- 发布端  
在发布端，deadline定义了应用程序预期提供新样本的最大周期。也就是说，发布者应该在这个周期内提供新的数据样本。
- 接收端  
在订阅端，deadline定义了预期接收新样本的最大周期。订阅者期望在这个周期内接收到新的数据样本。
- 具键的主题  
对于具有键的主题，这个QoS策略是按键应用的。假设需要定期发布一些车辆的位置，在这种情况下，可以将车辆的ID设置为数据类型的键，并将deadline QoS设置为期望的发布周期。
- 应用场景  
例如，如果一个系统需要每秒更新一次车辆的位置，那么可以将deadline QoS设置为1秒。如果发布者没有在1秒内发布新数据，或者订阅者在1秒内没有接收到新数据，就会触发警报。
- 触发回调  
on_offered_deadline_missed()：在发布者没有按时发送数据时触发。  
on_requested_deadline_missed()：在订阅者没有按时接收到数据时触发。
- 使用 Qos 限制  
为了保持 DataReaders 和 DataWriter 中 DeadlineQosPolicy 之间的兼容性，提供的截止时间（在 DataWriter 上配置）必须小于或等于请求的截止时间（在 DataReader 上配置），否则，实体将被视为不兼容。
- 使用 demo
```c++
TODO: Write a example!
```

### DurabilityQosPolicy
持久化策略，即使网络上没有 DataReader，DataWriter 也可以在整个主题中发送消息。此外，在写入一些数据后加入主题的 DataReader 可能有兴趣访问该信息。
DurabilityQosPolicy 定义了系统在数据读取器加入主题之前已存在的那些样本（samples）的行为。系统的行为取决于 DurabilityQosPolicyKind 的值。
#### DurabilityQosPolicyKind
- VOLATILE_DURABILITY_QOS  
过去的样本被忽略，新加入的数据读取器只接收自加入时刻后生成的样本。  
对过去数据不感兴趣，只关注最新数据的场景，如实时传感器数据流。  

- TRANSIENT_LOCAL_DURABILITY_QOS  
当新的数据读取器加入时，它的历史数据会填充为过去的样本  
这些样本只在本地存储，即在发布者所在的节点上保留，而不会在网络中的其他节点上共享。  
样本的存储会受到发布者的历史深度（History QoS）和资源限制的影响。  
- TRANSIENT_DURABILITY_QOS  
当新的数据读取器加入时，它的历史数据会填充为过去的样本，这些样本被存储在持久化存储上。  
样本被存储在持久化存储上（如磁盘），允许在系统重启后仍然可以访问。  
需要历史数据且需要在系统重启后仍然可用的场景，如需要可靠记录的业务应用。  

- PERSISTENT_DURABILITY_QOS（未实现）


### HistoryQosPolicy
当实例的值在成功传送到现有 DataReader 实体之前更改一次或多次时，此 QoS 策略控制系统的行为。它决定了发布者（Publisher）和订阅者（Subscriber）如何存储和传递样本数据。
它包含两个属性:  
(1) kind: 控制服务是否只交付最新值或者固定值  
- KEEP_LAST: 只保留最后 N 个
- KEEP_ALL: 保留所有样本
(2)  depth: 需要保留的最大样本数，仅在 KEEP_LAST 时生效, 必须小于或者等于 `max_sample_per_instance`.

#### HistoryQosPolicyKind
- KEEP_LAST_HISTORY_QOS  
- KEEP_ALL_HISTORY_QOS


### LivelinessQosPolicy
用于控制服务确保网络上的特定实体仍然活跃的机制的 QoS（服务质量）策略。通过不同的设置，可以区分数据更新频繁的应用程序和数据偶尔变化的应用程序。此外，它还允许应用程序自定义应该由活跃性机制检测的故障类型。
#### 配置选项
- kind   
确认活跃性机制由服务器控制还是发布端控制
- lease_duration  
指定DataWriter最后一次触发活动性以来等待的时间。在此时间段内，没有收到活动性断言，则认为 DataWriter 不再活跃
- announcement_period
指定 DataWriter 之间发送连续活跃性消息的时间间隔，仅在 `自动断言`和`参与者手动断言`时候有效，且小于 `lease_duration`。

#### LivelinessQosPolicyKind
- AUTOMATIC_LIVEINESS_QOS  
服务自动负责在所需的速率下更新活跃性租约（lease）。只要本地进程和连接到远程参与者的链路存在，远程参与者内的实体将被认为是活跃的  
适用于只需检测远程应用是否仍在运行的应用  


- MANUAL_BY_PARTICIPANT_LIVELINESS_QOS  
要求发布端的应用在租约时间（lease_duration）到期之前定期声明活跃性。发布任何新的数据值会隐式声明 DataWriter 的活跃性，但也可以通过调用 assert_liveliness 函数显式地声明。  
如果发布端的一个实体声明了活跃性，服务会推断同一个 DomainParticipant 中的所有其他实体也都是活跃的  


- MANUAL_BY_TOPIC_LIVELINESS_QOS
要求发布端的应用在租约时间到期之前定期声明活跃性。发布任何新的数据值会隐式声明 DataWriter 的活跃性，但也可以通过调用 assert_liveliness 函数显式地声明。  
这是更严格的模式，每一个DataWriter 内的实例被声明活跃，以确保 DataWriter 被认为是活跃的  


|特性                     | MANUAL_BY_PARTICIPANT_LIVELINESS_QOS |MANUAL_BY_TOPIC_LIVELINESS_QOS|
|----|----|----|
|粒度                     | DomainParticipan 内所有实体               | 单个 DataWriter|
|活跃性断言影响范围       | 影响同一个 DomainParticipant 内的所有实体 | 仅影响单个 DataWriter|
|活跃性断言方式           | 任意一个 DataWriter 断言活跃性即可        | 每个 DataWriter 必须独立断言其活跃性|
|发布新数据隐式断言活跃性 | 是                                        | 是|
|适用场景                 |需要多个 DataWriter 共享活跃性断言的场景   | 需要独立管理每个 DataWriter 活跃性的场景|

为了保持DataReader 和DataWriter 中LiveinessQosPolicy 之间的兼容性，DataWriter 类型必须高于或等于DataReader 类型。不同种类之间的顺序是：
```c++
|AUTOMATIC_LIVELINESS_QOS-api| < |MANUAL_BY_PARTICIPANT_LIVELINESS_QOS-api| < |MANUAL_BY_TOPIC_LIVELINESS_QOS-api|
```

### ReliabilityQosPolicy
该 QoS 策略指示服务提供和请求的可靠性级别。请参阅可靠性Qos策略。

|数据成员| Type | Default Value |
|----|----|----|
|kind              | ReliabilityQosPolicyKind | BEST_EFFORT_RELIABILITY_QOS for DataReaders RELIABLE_RELIABILITY_QOS for DataWriters |
|max_blocking_time | Duration_t               | 100 ms |

#### ReliabilityQosPolicyKind
- BEST_EFFORT_RELIABILITY_QOS  
在这种模式下，数据传输不要求确认接收，因此缺失的数据样本不会被重新发送。这种模式假设新的数据样本会频繁产生，因此不需要重新发送任何样本。  
数据样本在发送后不会等待接收确认  
数据样本按照发生的顺序存储在 DataReader 的历史中，即使 DataReader 丢失了一些数据样本，较旧的值也不会覆盖较新的值  
适用于不要求高可靠性的场景，比如实时视频流、传感器数据等，丢失一些数据对整体效果影响不大  


- RELIABLE_RELIABILITY_QOS  
在这种模式下，服务会尝试传递所有 DataWriter 历史中的样本，并期待 DataReader 的接收确认  
数据样本在发送后会等待接收确认，如果有数据丢失，服务会重新发送这些数据样本  
DataWriter 发送的数据样本在前面的样本未被接收前，不能被 DataReader 获取。服务会重新传输丢失的数据样本，以重建 DataWriter 历史的正确快照  
这种模式可能会阻塞写操作，因此会设置 max_blocking_time，一旦时间到达未发送的数据，写操作将返回错误  
适用于要求高可靠性的数据传输场景，如金融交易数据、命令和控制系统等，确保每个数据样本都被成功传递  

#### 兼容性

为了保持 DataReader 和 DataWriter 之间的兼容性，DataWriter 的可靠性等级必须高于或等于 DataReader 的可靠性等级。不同等级的顺序如下：
```c++
BEST_EFFORT_RELIABILITY_QOS < RELIABLE_RELIABILITY_QOS

```

### ResourceLimitsQosPolicy
此 QoS 策略控制服务可以使用的资源，以满足应用程序和其他 QoS 策略施加的要求。

| 数据成员 | 类型 | 默认值 |
| ----   | ---- | ---- |
| max_samples               | int32_t | 5000 |
| max_instance              | int32_t | 10   |
| max_samples_per_instance  | int32_t | 400  |
| allocated_samples         | int32_t | 100  |
|extra_samples              | int32_t | 1    |

- max_samples:  
控制 DataWriter 或 DataReader 可以管理的所有实例的最大样本数量。  
这代表中间件可以为某个 DataReader 或 DataWriter 存储的最大样本数。如果值小于或等于 0，则表示没有限制（无限资源）。  
用于限制一个 DataWriter 或 DataReader 可以处理的总数据量，避免过多数据占用系统资源。  

- max_instance:  
控制 DataWriter 或 DataReader 可以管理的实例的最大数量  

实例可以被认为是唯一键的不同集合。如果值小于或等于 0，则表示没有限制（无限资源）  
用于限制 DataWriter 或 DataReader 可以处理的不同数据集的数量。  

- max_samples_per_instance  
控制 DataWriter 或 DataReader 在每个实例内可以管理的最大样本数量。  
用于限制每个实例内可以处理的数据量  

- allocated_samples  
声明初始化时将分配的样本数量。  
这个值决定了在初始化时为 DataWriter 或 DataReader 分配的样本数量。预分配样本可以提高系统性能，减少在运行时分配样本的开销。  

- extra_samples  
声明将在池中分配的额外样本数量。   
最大样本数量（max_samples）加上额外样本数量（extra_samples）决定了池中的总样本数量。即使历史记录已满，这些额外的样本也可以作为样本的储备  
用于提供额外的缓冲，以应对突发的样本需求，确保即使在最大样本数已满的情况下，仍然有额外样本可用。  


