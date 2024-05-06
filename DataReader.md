## DataReader

DataReader 恰好附加到一个充当其工厂的订阅者。此外，每个 DataReader 自创建以来就绑定到一个主题。该主题必须在创建 DataReader 之前存在，并且必须绑定到 DataReader 想要发布的数据类型。

在订阅者中为特定主题创建新的 DataReader 的效果是使用该主题描述的名称和数据类型发起新的订阅。

创建 DataReader 后，当从远程发布接收到数据值的更改时，应用程序将收到通知。然后可以使用 DataReader 的 DataReader::read_next_sample() 或 DataReader::take_next_sample() 成员函数检索这些更改。

```c++
两个收取数据的区别
read_next_sample();
take_next_sample();

```

#### DataReaderListener

DataReaderListener 是一个抽象类，定义了响应 DataReader 上的状态更改而触发的回调。默认情况下，所有这些回调都是空的并且不执行任何操作。用户应该实现此类的专门化，以覆盖应用程序所需的回调。未被覆盖的回调将保持其空实现。

DataReaderListener 定义了以下回调：

- on_data_available()

DataReader 上的应用程序有可用的新数据。此回调的调用没有排队，这意味着如果一次接收到多个新数据更改，则只能为所有这些更改发出一个回调调用，而不是每次更改一个。如果应用程序在此回调中检索接收到的数据，则它必须继续读取数据，直到没有新的更改为止。

- on_subscription_matched()

DataReader 找到了与 Topic 匹配且具有公共分区和兼容 QoS 的 DataWriter，或者已不再与之前认为匹配的 DataWriter 匹配。当匹配的 DataWriter 更改其 DataWriterQos 时也会触发它。

- on_requested_deadline_missed()

DataReader 在其 DataReaderQos 上配置的截止期限内未收到数据。对于 DataReader 丢失数据的每个截止时间段和数据实例，都会调用它。

- on_requested_incompatible_qos()

DataReader 找到了一个与 Topic 匹配且具有公共分区的 DataWriter，但其 QoS 与 DataReader 上定义的 QoS 不兼容。

- on_liveliness_changed()

匹配的DataWriter的活跃状态已改变。不活动的 DataWriter 已变为活动状态，或者反之亦然。

- on_sample_rejected()

收到的数据样本被拒绝

- on_sample_lost()

数据样本丢失并且永远不会被接收

#### SampleInfo

当从 DataReader 检索样本时，除了样本数据之外，还会返回 SampleInfo 实例。该对象包含补充返回的数据值并帮助解释的附加信息。例如，如果 valid_data 值为 false，则 DataReader 不会通知应用程序数据实例中的新值，而是通知其状态发生变化，并且必须丢弃返回的数据值。

其中包括了时间戳、数据是否有效，sample状态等等。

以下部分描述 SampleInfo 的数据成员以及每个数据成员与返回的示例数据相关的含义：

```c++
struct SampleInfo
{
    //! indicates whether or not the corresponding data sample has already been read
    SampleStateKind sample_state;

    //! indicates whether the DataReader has already seen samples for the most-current generation of the related instance.
    ViewStateKind view_state;

    //! indicates whether the instance is currently in existence or, if it has been disposed, the reason why it was disposed.
    InstanceStateKind instance_state;

    //! number of times the instance had become alive after it was disposed
    int32_t disposed_generation_count;

    //! number of times the instance had become alive after it was disposed because no writers
    int32_t no_writers_generation_count;

    //! number of samples related to the same instance that follow in the collection
    int32_t sample_rank;

    //! the generation difference between the time the sample was received, and the time the most recent sample in the collection was received.
    int32_t generation_rank;

    //! the generation difference between the time the sample was received, and the time the most recent sample was received.
    //! The most recent sample used for the calculation may or may not be in the returned collection
    int32_t absolute_generation_rank;

    //! time provided by the DataWriter when the sample was written
    fastrtps::rtps::Time_t source_timestamp;

    //! time provided by the DataReader when the sample was added to its history
    fastrtps::rtps::Time_t reception_timestamp;

    //! identifies locally the corresponding instance
    InstanceHandle_t instance_handle;

    //! identifies locally the DataWriter that modified the instance
    //!
    //! Is the same InstanceHandle_t that is returned by the operation get_matched_publications on the DataReader
    InstanceHandle_t publication_handle;

    //! whether the DataSample contains data or is only used to communicate of a change in the instance
    bool valid_data;

    //!Sample Identity (Extension for RPC)
    fastrtps::rtps::SampleIdentity sample_identity;

    //!Related Sample Identity (Extension for RPC)
    fastrtps::rtps::SampleIdentity related_sample_identity;

};

```

- sample_state

指示之前是否已读取相应的数据样本。它可以采用以下值之一：READ:第一次被读取，NOT_READ:数据样本之前已被读取或获取。

- view_state

指示这是否是 DataReader 检索的该数据实例的第一个样本。它可以采用以下值之一：

NEW：这是第一次检索到该实例的样本

NOT_NEW: 之前已检索到该实例的其他样本。

- instance_state

指示实例当前是否存在或已被处置。在后一种情况下，它还提供有关处置原因的信息。它可以采用以下值之一：

#### 访问接收到的数据

应用程序可以通过读取或获取来访问和使用 DataReader 上接收到的数据值。

读取是通过以下任何成员函数完成的：

```c++
//读取 DataReader 上可用的下一个先前未访问的数据值，并将其存储在提供的数据缓冲区中。
DataReader::read_next_sample()
```
```c++
//提供获取符合特定条件的样本集合的机制。
DataReader::read();
DataReader::read_instance();
DataReader::read_next_instance();
```

获取是通过以下任何成员函数完成的：
```c++
//读取 DataReader 上可用的下一个先前未访问的数据值，并将其存储在提供的数据缓冲区中。
DataReader::take_next_sample();
```
```c++
DataReader::take();
DataReader::take_instance();
DataReader::take_next_instance();
```
take 获取数据时，返回的样本也会从 DataReader 中删除，因此无法再访问它们。
当DataReader中没有符合条件的数据时，所有操作都会返回NO_DATA，输出参数保持不变。
除了数据值之外，数据访问操作还为 SampleInfo 实例提供有助于解释返回的数据值的附加信息，例如原始 DataWriter 或发布时间戳。请参阅 SampleInfo 部分以获取其内容的详细描述。

#### 借出归还数据

```c++
   // Sequences are automatically initialized to be empty (maximum == 0)
    FooSeq data_seq;
    SampleInfoSeq info_seq;

    // with empty sequences, a take() or read() will return loaned
    // sequence elements
    ReturnCode_t ret_code = data_reader->take(data_seq, info_seq,
                    LENGTH_UNLIMITED, ANY_SAMPLE_STATE,
                    ANY_VIEW_STATE, ANY_INSTANCE_STATE);

    // process the returned data

    // must return the loaned sequences when done processing
    data_reader->return_loan(data_seq, info_seq);
```

#### 处理返回的数据

调用 DataReader::read() 或 DataReader::take() 操作后，访问返回序列上的数据非常容易。序列 API 提供了 length() 操作，返回集合中元素的数量。应用程序代码只需要检查该值并使用 [] 运算符来访问相应的元素。仅当 SampleInfo 序列上的相应元素指示存在有效数据时，才应访问 DDS 数据序列上的元素。使用数据共享时，**检查示例是否有效很重要**。

```c++
    // Sequences are automatically initialized to be empty (maximum == 0)
    FooSeq data_seq;
    SampleInfoSeq info_seq;

    // with empty sequences, a take() or read() will return loaned
    // sequence elements
    ReturnCode_t ret_code = data_reader->take(data_seq, info_seq,
                    LENGTH_UNLIMITED, ANY_SAMPLE_STATE,
                    ANY_VIEW_STATE, ANY_INSTANCE_STATE);

    // process the returned data
    if (ret_code == ReturnCode_t::RETCODE_OK)
    {
        // Both info_seq.length() and data_seq.length() will have the number of samples returned
        for (FooSeq::size_type n = 0; n < info_seq.length(); ++n)
        {
            // Only samples with valid data should be accessed
            if (info_seq[n].valid_data && data_reader->is_sample_valid(&data_seq[n], &info_seq[n]))
            {
                // Do something with data_seq[n]
            }
        }

        // must return the loaned sequences when done processing
        data_reader->return_loan(data_seq, info_seq);
    }
```



#### 在回调中访问数据

当 DataReader 从任何匹配的 DataWriter 接收到新数据值时，它会通过两个监听器回调通知应用程序：

- [`on_data_available()`](https://fast-dds.docs.eprosima.com/en/latest/fastdds/api_reference/dds_pim/subscriber/datareaderlistener.html#_CPPv4N8eprosima7fastdds3dds18DataReaderListener17on_data_availableEP10DataReader).
- [`on_data_on_readers()`](https://fast-dds.docs.eprosima.com/en/latest/fastdds/api_reference/dds_pim/subscriber/subscriberlistener.html#_CPPv4N8eprosima7fastdds3dds18SubscriberListener18on_data_on_readersEP10Subscriber).

```c++
class CustomizedDataReaderListener : public DataReaderListener
{

public:

    CustomizedDataReaderListener()
        : DataReaderListener()
    {
    }

    virtual ~CustomizedDataReaderListener()
    {
    }

    void on_data_available(
            DataReader* reader) override
    {
        // Create a data and SampleInfo instance
        Foo data;
        SampleInfo info;

        // Keep taking data until there is nothing to take
        while (reader->take_next_sample(&data, &info) == ReturnCode_t::RETCODE_OK)
        {
            if (info.valid_data)
            {
                // Do something with the data
                std::cout << "Received new data value for topic "
                          << reader->get_topicdescription()->get_name()
                          << std::endl;
            }
            else
            {
                std::cout << "Remote writer for topic "
                          << reader->get_topicdescription()->get_name()
                          << " is dead" << std::endl;
            }
        }
    }

};
```



如果一次接收到多个新数据更改，则可能仅触发一次回调，而不是每次更改一次。应用程序必须继续读取或获取，直到没有新的更改可用。

#### 使用等待线程访问数据

```c++
// Create a DataReader
DataReader* data_reader =
        subscriber->create_datareader(topic, DATAREADER_QOS_DEFAULT);
if (nullptr == data_reader)
{
    // Error
    return;
}

// Prepare a wait-set to wait for data on the DataReader
WaitSet wait_set;
StatusCondition& condition = data_reader->get_statuscondition();
condition.set_enabled_statuses(StatusMask::data_available());
wait_set.attach_condition(condition);

// Create a data and SampleInfo instance
Foo data;
SampleInfo info;

//Define a timeout of 5 seconds
eprosima::fastrtps::Duration_t timeout (5, 0);

// Loop reading data as it arrives
// This will make the current thread to be dedicated exclusively to
// waiting and reading data until the remote DataWriter dies
while (true)
{
    ConditionSeq active_conditions;
    if (ReturnCode_t::RETCODE_OK == wait_set.wait(active_conditions, timeout))
    {
        while (ReturnCode_t::RETCODE_OK == data_reader->take_next_sample(&data, &info))
        {
            if (info.valid_data)
            {
                // Do something with the data
                std::cout << "Received new data value for topic "
                          << topic->get_name()
                          << std::endl;
            }
            else
            {
                // If the remote writer is not alive, we exit the reading loop
                std::cout << "Remote writer for topic "
                          << topic->get_name()
                          << " is dead" << std::endl;
                break;
            }
        }
    }
    else
    {
        std::cout << "No data this time" << std::endl;
    }
}
```

#### 数据读取非阻塞调用

使用 DataReader::wait_for_unread_message() 成员函数也可以实现相同的目的，该函数会阻塞，直到有新的数据样本可用或给定的超时到期。如果超时后没有新数据可用，则返回 false 值。该函数返回 true 值意味着 DataReader 上有新数据可供应用程序检索。

```c++
// Create a DataReader
DataReader* data_reader =
        subscriber->create_datareader(topic, DATAREADER_QOS_DEFAULT);
if (nullptr == data_reader)
{
    // Error
    return;
}

// Create a data and SampleInfo instance
Foo data;
SampleInfo info;

//Define a timeout of 5 seconds
eprosima::fastrtps::Duration_t timeout (5, 0);

// Loop reading data as it arrives
// This will make the current thread to be dedicated exclusively to
// waiting and reading data until the remote DataWriter dies
while (true)
{
    if (data_reader->wait_for_unread_message(timeout))
    {
        if (ReturnCode_t::RETCODE_OK == data_reader->take_next_sample(&data, &info))
        {
            if (info.valid_data)
            {
                // Do something with the data
                std::cout << "Received new data value for topic "
                          << topic->get_name()
                          << std::endl;
            }
            else
            {
                // If the remote writer is not alive, we exit the reading loop
                std::cout << "Remote writer for topic "
                          << topic->get_name()
                          << " is dead" << std::endl;
                break;
            }
        }
    }
    else
    {
        std::cout << "No data this time" << std::endl;
    }
}
```

