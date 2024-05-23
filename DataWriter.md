## DataWriter

#### 结构

```c++
class DataWriter : public DomainEntity
{
  ...
};
```

此外，每个 DataWriter 自创建以来就绑定到一个主题 (Topic)。该主题必须在创建 DataWriter 之前存在，并且必须绑定到 DataWriter 想要发布的数据类型。

在发布者中为特定主题创建新的 DataWriter 的效果是使用该主题描述的名称和数据类型启动新发布。

创建 DataWriter 后，应用程序可以使用 DataWriter 上的 write() 成员函数**通知数据值的更改**。这些更改将传输到与此出版物匹配的所有订阅。

#### DataWriterQos

DataWriterQos 控制 DataWriter 的行为。

[DataWriterQos](https://fast-dds.docs.eprosima.com/en/latest/fastdds/dds_layer/publisher/dataWriter/dataWriter.html)

#### DataWriterListener

DataWriterListener 是一个抽象类，定义了响应 DataWriter 上的状态更改而触发的回调。默认情况下，所有这些回调都是空的并且不执行任何操作。用户应该实现此类的专门化，以覆盖应用程序所需的回调。未被覆盖的回调将保持其空实现。

[`DataWriterListener`](https://fast-dds.docs.eprosima.com/en/latest/fastdds/api_reference/dds_pim/publisher/datawriterlistener.html#_CPPv4N8eprosima7fastdds3dds18DataWriterListenerE) 定义了以下回调：

- on_publication_matched()

DataWriter找到与Topic匹配且具有公共分区和兼容QoS的DataReader，或者已不再与之前认为匹配的DataReader匹配。

- on_offered_deadline_missend()

DataWriter 未能在其 DataWriterQos 上配置的截止期限内提供数据。对于 DataWriter 未能提供数据的每个截止时间段和数据实例，都会调用它。

- on_offered_incompatible_qos()

DataWriter 已找到与 Topic 匹配且具有公共分区的 DataReader，但所请求的 QoS 与 DataWriter 上定义的 QoS 不兼容。

- on_liveliness_lost()

DataWriter 不尊重其 DataWriterQos 上的活跃度配置，因此 DataReader 实体将认为 DataWriter 不再处于活动状态。

- on_unacknowledged_sample_removed()

数据写入器删除了尚未被每个匹配的数据读取器确认的样本。

#### 发布数据

##### 写操作阻塞

如果 DataWriterQos 上的可靠性类型设置为 RELIABLE，则 write() 操作可能会阻塞。具体来说，如果已达到配置的资源限制中指定的限制，则 write() 操作将阻塞等待空间变得可用。在这种情况下，可靠性 max_blocking_time 配置写操作可能阻塞等待的最长时间。如果在 DataWriter 能够存储修改而不超出限制之前经过了 max_blocking_time，则写入操作将失败并返回 TIMEOUT。

##### 借用数据缓冲

当用户使用新的样本值调用 write() 时，数据将从给定样本复制到 DataWriter 的内存中。对于大型数据类型，此副本可能会消耗大量时间和内存资源。相反，DataWriter 可以将内存中的样本借给用户，并且用户可以用所需的值填充该样本。当使用此类借出的样本调用 write() 时，DataWriter 不会复制其内容，因为它已经拥有缓冲区。

一旦使用借出的样本调用 write() ，借出的样本将被视为已归还，并且对样本内容进行任何更改都是不安全的。

```c++
    // Borrow a data instance
    void* data = nullptr;
    if (ReturnCode_t::RETCODE_OK == data_writer->loan_sample(data))
    {
        bool error = false;

        // Fill the data values
        // (...)

        if (error)
        {
            // Return the loan without publishing
            data_writer->discard_loan(data);
            return;
        }

        // Publish the new value
        if (data_writer->write(data, eprosima::fastrtps::rtps::InstanceHandle_t()) != ReturnCode_t::RETCODE_OK)
        {
            // Error
            return;
        }
    }

    // The data instance can be reused to publish new values,
    // but delete it at the end to avoid leaks
    custom_type_support->deleteData(data);
```

