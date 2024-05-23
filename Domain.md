#### Domain

#### 1、域介绍

域代表一个单独的通信平面。它在共享公共通信基础设施的实体之间创建了逻辑分离。从概念上讲，它可以看作是一个虚拟网络，链接同一域上运行的所有应用程序，并将它们与不同域上运行的应用程序隔离。这样，多个独立的分布式应用程序可以共存于同一个物理网络中，互不干扰，甚至互不知晓。

每一个域有其唯一是识别，`uint32 dimainID` 。共享此domainId的应用程序属于同一域并且能够进行通信。

对于要添加到域的应用程序，它必须创建具有适当的domainId 的DomainParticipant 实例。 DomainParticipant 的实例是通过 DomainParticipantFactory 单例创建的。（以上翻译自官方文档）

#### 2、DomainParticipant（域参与者/加入者）

每个 DomainParticipant 从创建时起就链接到单个域，并包含与该域相关的所有实体。它还充当发布者、订阅者和主题的工厂。可以使用 DomainParticipantQos 上指定的 QoS 值来修改 DomainParticipant 的行为。 QoS 值可以在创建 DomainParticipant 时设置，或者稍后使用 DomainParticipant::set_qos() 成员函数进行修改。

可以将 DomainParticipant 视为一个应用中参与 DDS 通信的纽带。

作为一个实体，DomainParticipant 接受一个 DomainParticipantListener，它将收到有关 DomainParticipant 实例的状态更改的通知。

#### 3、DomainParticipantListener（域参与者监听器）

DomainParticipantListener 是一个抽象类，定义了响应 DomainParticipant 上的状态更改而触发的回调。

```c++
class DomainParticipantListener
  :public PublisherListener,
	 public SubscriberListener,
	 public TopicListener,
{
  // 
  virtual void on_participant_discovery(DomainParticipant* particiant,
                                       fastrtps::rtps::ParticipantDiscoveryInfo&& info
                                       );
};
```

因此，它有能力对报告给其任何附属实体的各种事件做出反应。由于事件总是通知给可以处理该事件的最具体的实体侦听器，因此仅当没有其他实体能够处理该事件时，才会调用 DomainParticipantListener 从其他侦听器继承的回调，或者因为它没有附加侦听器，或者因为实体上的 StatusMask 禁用回调。（TODO： StatusMask 如何禁用回调）

`on_participant_discovery`:在同一域中发现新的 DomainParticipant，先前已知的 DomainParticipant 已被删除，或者某些 DomainParticipant 已更改其 QoS会触发该函数，

// TODO: 验证所有的 listener 回调行为

#### 4、DomainParticipantFactory(域加入者工厂)

该类的唯一目的是允许创建和销毁 DomainParticipant 对象。 DomainParticipantFactory本身没有工厂，它是一个单例对象，可以通过DomainParticipantFactory类上的get_instance()静态成员函数来访问。

DomainParticipantFactory 不接受任何侦听器，因为它不是实体。

同样也可以使用 Qos 设置工厂的属性

```c++
DomainParticipantFactoryQos qos;
qos.entity_factory().autoenable_created_entities = true;
if (DomainParticipantFactory::get_instance()->set_qos(qos) != RETCODE_OK)
{
    // Error
    return;
}

// 创建 DomainParticipant
DomainParticipant* enabled_participant =
        DomainParticipantFactory::get_instance()->create_participant(0, PARTICIPANT_QOS_DEFAULT);
if (nullptr == enabled_participant)
{
    // Error
    return;
}
```

通过 xml 配置文件创建

```bash

DomainParticipantFactory::get_instance()->load_XML_profiles_file("profiles.xml");

DomainParticipant* participant_with_profile =
        DomainParticipantFactory::get_instance()->create_participant_with_profile(0, "participant_profile");
if (nullptr == participant_with_profile)
{
    // Error
    return;
}
```

#### 5、删除域参与者

仅当属于参与者（发布者、订阅者或主题）的所有实体已被删除时，才能删除 DomainParticipant。否则，该函数将发出错误，并且 DomainParticipant 将不会被删除。这可以通过使用 DomainParticipant 的 delete_contained_entities() 成员函数来执行，也可以使用 DomainParticipant 中相应的 delete_ 方法手动删除每个实体。

```bash
// 创建 DomainParticipant
DomainParticipant* participant =
        DomainParticipantFactory::get_instance()->create_participant(0, PARTICIPANT_QOS_DEFAULT);
if (nullptr == participant)
{
    // Error
    return;
}

// 借助 DomainParticipant 进行通信
......

// 删除 entities created by the DomainParticipant
if (participant->delete_contained_entities() != RETCODE_OK)
{
    // DomainParticipant failed to delete the entities it created.
    return;
}

// 删除  DomainParticipant
if (DomainParticipantFactory::get_instance()->delete_participant(participant) != RETCODE_OK)
{
    // Error
    return;
}
```

#### 6、分区

分区在一个域内的物理隔离内引入了逻辑实体隔离级别概念。它在域和主题之外采用另一种方式来区分发布者和订阅者。

为了使发布者与订阅者通信，它们必须至少属于一个公共分区。

暂时认为不配置的都在默认空白分区即可。TODO: 分区的原理

