## Publisher

发布者代表属于它的一个或多个 DataWriter 对象进行操作。它充当一个容器，允许在发布者的 PublisherQos 给出的通用配置下对不同的 DataWriter 对象进行分组。



```c++
class Publisher : public DomainEntity
{
  protected:
  	PublisherImpl* impl_;	// 具体实现
};
```



```c++
    //! Map of Pointers to the associated Data Writers. Topic name is the key.
    std::map<std::string, std::vector<DataWriterImpl*>> writers_;
```

属于同一发布者的 DataWriter 对象除了发布者的 PublisherQos 之外，彼此之间没有任何其他关系，并且在其他情况下独立运行。具体来说，发布者可以托管不同主题和数据类型的 DataWriter 对象。

#### PublisherQos

PublisherQos 控制了 Publisher 的行为。

可以设置和修改 Qos

[PublisherQos](https://fast-dds.docs.eprosima.com/en/latest/fastdds/dds_layer/publisher/publisher/publisher.html)

#### PublisherListerner

PublisherListener 继承自 DataWriterListener。因此，它能够对报告给 DataWriter 的所有事件做出反应，并且它自己没有任何新的注册回调。

由于事件始终会通知到可以处理该事件的**最具体的实体侦听器**，***因此仅当触发的 DataWriter 没有附加侦听器，或者回调被 DataWriter 上的 StatusMask 禁用时***，才会调用 PublisherListener 从 DataWriterListener 继承的回调。

```c++
class PublisherListener: public DataWriterListener
{
};
```

#### 创建 Publisher

使用 Participant 来创建 Publisher

```c++
DomainParticipant* participant =
        DomainParticipantFactory::get_instance()->create_participant(0, PARTICIPANT_QOS_DEFAULT);
if (nullptr == participant)
{
    // Error
    return;
}

// 创建默认 QOS 的 publisher
Publisher* publisher_with_default_qos =
        participant->create_publisher(PUBLISHER_QOS_DEFAULT);
if (nullptr == publisher_with_default_qos)
{
    // Error
    return;
}


/////////////////////////////////////////////
// 手动修改 QOS 来创建
PublisherQos custom_qos;

// Modify QoS attributes
// (...)

Publisher* publisher_with_custom_qos =
        participant->create_publisher(custom_qos);
if (nullptr == publisher_with_custom_qos)
{
    // Error
    return;
}


////////////////////////////////////////////
/// 注册监听回调
CustomPublisherListener custom_listener;
Publisher* publisher_with_default_qos_and_custom_listener =
        participant->create_publisher(PUBLISHER_QOS_DEFAULT, &custom_listener);
if (nullptr == publisher_with_default_qos_and_custom_listener)
{
    // Error
    return;
}
```

基于配置文件的发布者创建

```c++
// 加载配置文件
DomainParticipantFactory::get_instance()->load_XML_profiles_file("profiles.xml");


DomainParticipant* participant =
        DomainParticipantFactory::get_instance()->create_participant(0, PARTICIPANT_QOS_DEFAULT);
if (nullptr == participant)
{
    // Error
    return;
}

// 使用配置文件创建
Publisher* publisher_with_profile =
        participant->create_publisher_with_profile("publisher_profile");
if (nullptr == publisher_with_profile)
{
    // Error
    return;
}

////////////////////////////////////////////////
///// 注册监听回调
CustomPublisherListener custom_listener;
Publisher* publisher_with_profile_and_custom_listener =
        participant->create_publisher_with_profile("publisher_profile", &custom_listener);
if (nullptr == publisher_with_profile_and_custom_listener)
{
    // Error
    return;
}
```

