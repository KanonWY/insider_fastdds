## 事件回调的一些细节

## 1、事件回调的继承关系
![](./source/Listener_callback_chain.svg)

从上面的关系图可知:

```c++
class DimainParticipantListener : public SubscriberListener, public PublisherListener, public TopicListener
{};

class PublisherListener: public DataWriterListener
{};

class SubscriberListener: public DataReaderListener
{};
```
从上图中可以看出，域发现者回调类可以重写重写所有的事件回调.

## 2、域发现者事件回调
默认情况下如果注册了域参与者注册回调会默认注册`StatuMask::all()`，所有的回调都将会触发。
域参与者注册回调有以下特有的虚函数:
```c++


    // 域参与者注册回调
    // 触发回调:
    // (1) 在同一域中发现新的 DomainParticipant
    // (2) 先前已知的 DomainParticipant 已被删除
    // (3) 或者某些 DomainParticipant 已更改其 QoS。
    virtual void on_participant_discovery(
            DomainParticipant* participant,
            fastrtps::rtps::ParticipantDiscoveryInfo&& info,
            bool& should_be_ignored)
    {
        static_cast<void>(participant);
        static_cast<void>(info);
        // 该参数的为输出参数，对应的域参与者是否应该被忽略。
        should_be_ignored = false;
    }

    // 数据读取器触发回调
    // 触发条件:
    // (1) 在同一域中发现了新的 DataReader，
    // (2) 先前已知的 DataReader 已被删除，
    // (3) 或者某些 DataReader 已更改其 QoS
    virtual void on_data_reader_discovery(
            DomainParticipant* participant,
            fastrtps::rtps::ReaderDiscoveryInfo&& info,
            bool& should_be_ignored)
    {
        static_cast<void>(participant);
        static_cast<void>(info);
        static_cast<void>(should_be_ignored);
    }

    // 数据写入器触发回调
    // 触发回调:
    // (1) 在同一域中发现了新的 DataWriter
    // (2) 先前已知的 DataWriter 已被删除 
    // (3) 某些 DataWriter 已更改其 QoS。
    virtual void on_data_writer_discovery(
            DomainParticipant* participant,
            fastrtps::rtps::WriterDiscoveryInfo&& info,
            bool& should_be_ignored)
    {
        static_cast<void>(participant);
        static_cast<void>(info);
        static_cast<void>(should_be_ignored);
    }

```

## 3、订阅器事件回调
继承了数据读取器的监听事件回调, 因此可以重写数据读取器的所有监听事件回调
```c++
class SubscriberListener : public DataReaderListener
{
    // 属于该订阅器的数据可用回调
    // 触发回调:
    // (1) 新数据在属于该订阅者的任何 DataReader 上可用
    // 注意事项:
    // 此回调的调用没有排队，这意味着如果一次收到多个新数据更改，则只能为所有这些更改发出一个回调调用，而不是每次更改一个回调调用
    // 如果应用程序在此回调中检索接收到的数据，则它必须持续读取数据，直到没有新的更改为止。
    FASTDDS_EXPORTED_API virtual void on_data_on_readers(
            Subscriber* sub)
    {
        (void)sub;
    }

};
```

## 4、数据读取器事件回调
数据读取器事件监听回调会监听数据接受处理的相关消息.
- 数据可用触发回调
DataReader 上有可供应用程序使用的新数据, 会触发该回调，此回调的调用没有排队，这意味着如果一次收到多个新数据更改，则只能为所有这些更改发出一个回调调用，而不是每次更改一个。如果应用程序在此回调中检索接收到的数据，则它必须继续读取数据，直到没有新的更改为止。
```c++
    FASTDDS_EXPORTED_API virtual void on_data_available(
            DataReader* reader)
    {
        (void)reader;
    }
```
- 有话题想匹配的写入器
DataReader 找到了与 Topic 匹配且具有公共分区和兼容 QoS 的 DataWriter，或者已不再与之前认为匹配的 DataWriter 匹配。当匹配的 DataWriter 更改其 DataWriterQos 时也会触发它。
```c++
    FASTDDS_EXPORTED_API virtual void on_subscription_matched(
            DataReader* reader,
            const fastdds::dds::SubscriptionMatchedStatus& info)
    {
        (void)reader;
        (void)info;
    }
```

- deadline 期限内未收到数据
DataReader 在其 DataReaderQos 上配置的截止期限内未收到数据。对于 DataReader 丢失数据的每个截止时间段和数据实例，都会调用它。
```c++
    FASTDDS_EXPORTED_API virtual void on_requested_deadline_missed(
            DataReader* reader,
            const RequestedDeadlineMissedStatus& status)
    {
        (void)reader;
        (void)status;
    }
```
- 找到了一个与当前读取器匹配的写入器，但是 Qos 不一致
```c++
  FASTDDS_EXPORTED_API virtual void on_requested_incompatible_qos(
            DataReader* reader,
            const RequestedIncompatibleQosStatus& status)
    {
        (void)reader;
        (void)status;
    }
```
- 匹配的写入的活跃状态改变
匹配的DataWriter的活跃状态已改变。不活动的 DataWriter 已变为活动状态，或者反之亦然。
```c++
    FASTDDS_EXPORTED_API virtual void on_liveliness_changed(
            DataReader* reader,
            const LivelinessChangedStatus& status)
    {
        (void)reader;
        (void)status;
    }
```
- 接收的数据样例被拒绝
回调函数签名
```c++
    FASTDDS_EXPORTED_API virtual void on_sample_rejected(
            DataReader* reader,
            const SampleRejectedStatus& status)
    {
        (void)reader;
        (void)status;
    }
```

在Fast DDS中，样本可能会因资源限制原因而被拒绝。然而，样本被拒绝的事实并不意味着它们丢失了，即被拒绝的样本将来可能会被接受。  
样例拒绝状态:
```c++
// 数据拒收状态
struct SampleRejectedStatus
{
  
    // 当前DataReader的Topic下拒绝样本的累计总数
    uint32_t total_count = 0;
    // 自上次调用 on_sample_rejected() 或读取状态以来 Total_count 的变化。它只能是正数或零。
    uint32_t total_count_change = 0; 
    // 数据拒收的原因
    SampleRejectedStatusKind last_reason = NOT_REJECTED;
    InstanceHandle_t last_instance_handle;
};

// 数据拒收原因枚举
enum SampleRejectedStatusKind
{
    NOT_REJECTED,
    REJECTED_BY_INSTANCES_LIMIT,   // DDS 层中
    REJECTED_BY_SAMPLES_LIMIT,     // RTPS 层中
    REJECTED_BY_SAMPLES_PER_INSTANCE_LIMIT //DDS 层，历史记录上限
};
```
系统必须管理并维护一定的资源，以确保所有样本（特别是那些较低序列号的样本）能被接收和处理。即使当前有空闲资源，这些资源也可能已经预留或计划用于未来的样本。
当达到 max_samples 限制时，系统中的最大样本数量已经满了，因此没有多余的资源来接收新的样本。为了保证数据的一致性和顺序性，系统会拒绝接收新的样本。  
当一个新样本对应于一个新实例时，中间件需要保留资源来容纳它。但是，如果已存储在历史记录中的实例数量已经达到了最大允许限制（max_instances），则中间件无法为新实例分配资源，导致样本被拒绝。  
当DataReader被配置为使用 KEEP_ALL_HISTORY_QOS 时，它会保留接收到的所有历史样本。这意味着实例的历史记录可能会占用大量资源。
当某个特定实例的样本数量已达到 max_samples_per_instance 的限制时，即使还有其他实例，也无法再接收该实例的新样本。这是因为已经达到了为该实例保留的最大样本数，因此无法再存储更多的样本。  

```c++
KEEP_ALL_HISTORY_QOS  // 保留所有历史
KEEP_LAST_HISTORY_QOS // 显示保留样本，使用 history.depth 来配置
TODO: 进一步学习历史样本的保存和获取及其原理

```

- 数据样本丢失
数据样本丢失并且永远不会被接收
回调函数原型
```c++

    FASTDDS_EXPORTED_API virtual void on_sample_lost(
            DataReader* reader,
            const SampleLostStatus& status)
    {
        (void)reader;
        (void)status;
    }
```
根据可靠性（），有两种不同的标准将样本视为丢失：
(1) 当使用 `BEST_EFFORT_RELIABILITY_QOS` 时，每当收到具有更大序列号的样本时，尚未接收到的样本将被视为丢失。  
(2) 使用 `RELIABLE_RELIABILITY_QOS` 时，每当 DataWriter 通过 `RTPS HEARTBEAT` 子消息通知样本不再可用时，尚未收到的样本将被视为丢失。

```c++
struct BaseStatus {
    // 总积累
    int32_t total_count = 0;
    // 上次调用以来的积累
    int32_t total_count_change = 0;
};

```

## 5、发布器的事件回调
发布器的事件回调类继承了写入器的数据回调类，但是本事没有任何特有的回调.

## 6、数据写入器

- 匹配的数据读取器回调事件
回调函数接口
```c++
    // DataWriter找到与Topic匹配且具有公共分区和兼容QoS的DataReader，或者已不再与之前认为匹配的DataReader匹配。
    virtual void on_publication_matched(
            DataWriter* writer,
            const PublicationMatchedStatus& info)
    {
        static_cast<void>(writer);
        static_cast<void>(info);
    }
```

- deadline 未发送数据
DataWriter 未能在其 DataWriterQos 上配置的截止期限内提供数据。对于 DataWriter 未能提供数据的每个截止时间段和数据实例，都会调用它。
```c++
    virtual void on_offered_deadline_missed(
            DataWriter* writer,
            const OfferedDeadlineMissedStatus& status)
    {
        static_cast<void>(writer);
        static_cast<void>(status);
    }
```

- qos 不兼容
```c++
    virtual void on_offered_incompatible_qos(
            DataWriter* writer,
            const OfferedIncompatibleQosStatus& status)
    {
        static_cast<void>(writer);
        static_cast<void>(status);
    }
```

- 活跃度不匹配
DataWriter 不尊重其 DataWriterQos 上的活跃度配置，因此 DataReader 实体将认为 DataWriter 不再处于活动状态。
```c++
    virtual void on_liveliness_lost(
            DataWriter* writer,
            const LivelinessLostStatus& status)
    {
        static_cast<void>(writer);
        static_cast<void>(status);
    }
```

- U-ACK 数据删除事件
数据写入器删除了尚未被每个匹配的数据读取器确认的样本。
```c++
    virtual void on_unacknowledged_sample_removed(
            DataWriter* writer,
            const InstanceHandle_t& instance)
    {
        static_cast<void>(writer);
        static_cast<void>(instance);
    }
```
`on_unacknowledged_sample_removed()` 非标准回调会在样本已被删除而未由匹配的 DataReader 发送/接收时通知用户。  
在网络受限或发布吞吐量要求过高的情况下，可能会发生这种情况。此回调可用于检测这些情况，以便发布应用程序可以应用一些解决方案来缓解此问题，例如降低发布率。

什么情况下可以认为样本已被删除但是未被确认?  
它取决于可靠性的QOS `ReliabilityQosPolicy`!  
(1) `BEST_EFFORT_RELIABILITY_QOS`, 尽力交付.如果样本尚未通过传输发送，DataWriters 将通知样本已在未确认的情况下被移除。  
(2) `RELIABLE_RELIABILITY_QOS`，可靠交付.

#### RELIABLE_RELIABILITY_QOS
在这种配置下，DataWriter会等待每个匹配的DataReader的ACK（确认）消息来确认样本是否成功接收。
如果没有收到所有DataReader的ACK消息，样本会被标记为“未确认移除”。
这种情况下，可能存在竞态条件，即ACK消息可能在样本移除时尚未到达或在传输中丢失，导致误报。


#### 消息确认流程
- 样本发送后，等待ACK消息：
DataWriter发送样本后，开始等待匹配的DataReader发送ACK消息。
- 超时管理：
如果在QoS策略定义的时间内没有收到ACK消息，DataWriter可以采取措施，如重发样本或将样本标记为未确认。
例如，如果使用了DisablePositiveACKsQosPolicy，则在规定时间内没有收到NACK消息会被视为样本已成功接收。
- 处理竞态条件：
在这些策略下，仍可能出现竞态条件，例如ACK或NACK消息在网络传输中丢失或延迟到达。
系统设计者需要在应用中考虑这种情况，并通过适当的重试机制、超时处理和日志记录来管理这些竞态条件。

#### NACK 机制
在DDS（数据分发服务）系统中，NACK（Negative Acknowledgment，负确认）是一种机制，用于通知发送者（DataWriter）接收者（DataReader）没有成功接收到样本。这种机制在可靠数据传输中非常重要，尤其是在使用DisablePositiveACKsQosPolicy策略时.


