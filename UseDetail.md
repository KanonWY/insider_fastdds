## 使用时的注意事项

## 1、回调的覆盖

- 更 detail 的回调类的注册会影响泛化类比的回调注册的触发。（注意，这些回调指的是从基类继承来的，如果是自己的回调则不会有这种影响）

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


Using StatusMask::none() when creating the Entity only disables the DDS standard callbacks.
这意味着 StatusMask 仅仅禁用 DDS 标准回调。  
Any callback specific to Fast DDS is always enabled:  
这意味者 FastDDS 回调不受到 StatusMask 影响.


## 2、具体的回调
### 2.1 DataReaderListener
- `on_data_available()`  
DataReader 上的应用程序有可用的新数据。此回调的调用没有排队，这意味着如果一次收到多个新数据更改，则只能为所有这些更改发出一个回调调用，而不是每次更改一个回调调用,如果应用程序在此回调中检索接收到的数据，则它必须继续读取数据，直到没有新的更改为止。
- `on_subscription_matched`  
DataReader 找到了与 Topic 匹配且具有公共分区和兼容 QoS 的 DataWriter，或者已不再与之前认为匹配的 DataWriter 匹配.当匹配的 DataWriter 更改其 DataWriterQos 时也会触发它。
- `on_requested_deadline_missed`  
DataReader 在其 DataReaderQos 上配置的截止期限内未收到数据。对于 DataReader 丢失数据的每个截止时间段和数据实例，都会调用它。
- `on_requested_incompatible_qos`  
DataReader 找到了一个与 Topic 匹配且具有公共分区的 DataWriter，但其 QoS 与 DataReader 上定义的 QoS 不兼容。
- `on_liveliness_changed`  
匹配的DataWriter的活跃状态已改变。不活动的 DataWriter 已变为活动状态，或者反之亦然。
- `on_sample_rejected`  
收到的数据样本被拒绝。(TODO: 什么情况下数据样本将被拒绝)
- `on_sample_lost`  
数据样本丢失并且永远不会被接收。(TODO: 什么情况下数据会被永久丢失)

### 2.2 SubscriberListener
```c++
class SubscriberListener : public DataReaderListener
{};
```
显然会继承`DataReaderListener` 的所有虚函数。  特有回调函数:
- `on_data_on_reader`  
新数据在属于该订阅者的任何 DataReader 上可用会触发该函数。
此回调的调用没有排队，这意味着如果一次收到多个新数据更改，则只能为所有这些更改发出一个回调调用，而不是每次更改一个。如果应用程序在此回调中检索接收到的数据，则它必须继续读取数据，直到没有新的更改为止。
```c++
// 继承 SubscriberListener 即可
class CustomSubscriberListener : public SubscriberListener
{

public:

    CustomSubscriberListener()
        : SubscriberListener()
    {
    }

    virtual ~CustomSubscriberListener()
    {
    }

    // 任何该订阅器上的 reader 数据到达
    void on_data_on_readers(
            Subscriber* sub) override
    {
        static_cast<void>(sub);
        std::cout << "New data available" << std::endl;
    }
};
```
### 2.3 DataWriterListener （数据写入器触发回调)
- `on_publication_matched`  
DataWriter找到与Topic匹配且具有公共分区和兼容QoS的DataReader，或者已不再与之前认为匹配的DataReader匹配。
- `on_offered_deadline_missed`  
DataWriter 未能在其 DataWriterQos 上配置的截止期限内提供数据。对于 DataWriter 未能提供数据的每个截止时间段和数据实例，都会调用它。
- `on_liveliness_lost`  
ataWriter 不遵循其 DataWriterQos 上的活跃度配置，因此 DataReader 实体将认为 DataWriter 不再处于活动状态。
- `on_unacknowledged_sample_removed`  
数据写入器删除了尚未被每个匹配的数据读取器确认的样本。（什么情况下，写入器会删除样本？，缓存空间不足？）  

#### on_unacknowledged_sample_removed
在网络受限或发布吞吐量要求过高的情况下，可能会发生这种情况。此回调可用于检测这些情况，以便发布应用程序可以应用一些解决方案来缓解此问题，例如降低发布率。

认为样本已被删除但未被确认的标准取决于 ReliabilityQosPolicy：
- `BEST_EFFORT_RELIABILITY_QOS`:  
尽力交付，如果样本尚未通过传输发送，DataWriters 将通知样本已被删除但未确认。  
- `RELIABLE_RELIABILITY_QOS`  

Fast DDS 中非标准回调 on_unacknowledged_sample_removed() 的作用及其在不同可靠性 QoS 策略下的行为。这是为了帮助用户检测未被确认的样本移除的情况，并采取适当措施来解决这些问题。

关键点解析
回调功能：

on_unacknowledged_sample_removed() 是一个非标准的回调函数，当一个样本在没有被所有匹配的 DataReader 发送或接收时，该回调会通知用户。
这种情况可能发生在网络受限或发布速率过高的情况下。通过这个回调，发布应用程序可以检测到这些情况，并采取措施来缓解问题，例如降低发布速率。
未被确认的标准：

样本被认为未被确认的标准取决于 ReliabilityQosPolicy 的设置：
BEST_EFFORT_RELIABILITY_QOS：

在 BEST_EFFORT_RELIABILITY_QOS 下，如果样本没有通过传输发送出去，DataWriter 将通知样本未被确认。
在这种模式下，不保证所有数据都能成功传输，主要关注尽最大努力传输数据。
RELIABLE_RELIABILITY_QOS：

在 RELIABLE_RELIABILITY_QOS 下，如果不是所有匹配的 DataReader 都确认接收了样本（即没有发送相应的确认消息），DataWriter 将认为样本未被确认。
一个被通知为未确认的样本可能已经被一个或多个 DataReader 接收，但不是所有匹配的 DataReader。或者在样本移除的那一刻，Fast DDS 并没有收到所有 DataReader 的确认消息。
这种情况下，可能会出现竞争条件：在样本被移除时，部分 DataReader 的确认消息可能还未到达。这可能是由于传输过程中消息丢失，或者消息仍在传输途中。
尽管存在误报（即某些样本实际上已经被接收，但确认消息还未到达），但从用户的角度来看，了解样本未被所有匹配的 DataReader 确认接收是更有意义的。
总结
on_unacknowledged_sample_removed() 回调函数允许用户检测未确认样本的情况。
在 BEST_EFFORT 模式下，如果样本未发送出去，系统会通知用户。
在 RELIABLE 模式下，如果样本未被所有匹配的 DataReader 确认接收，系统会通知用户，尽管可能存在误报。
这种机制帮助用户识别和解决潜在的网络或性能问题，例如通过降低发布速率来减少未确认样本的情况。

**`BEST_EFFORT` 模式**：只要样本从 `DataWriter` 成功发送，无论是否被 `DataReader` 接收或确认，`on_unacknowledged_sample_removed()` 回调都不会被触发。

**`RELIABLE` 模式**：样本需要被所有匹配的 `DataReader` 确认接收，否则即使样本已经发送，回调也会被触发。

