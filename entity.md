## 类型实体

![实体继承关系](https://fast-dds.docs.eprosima.com/en/latest/_images/entity_diagram.svg)

Entity 是所有 DDS 实体的抽象基类，表示支持 QoS 策略、侦听器和状态的对象。

```c++
class Entity
{
  StatusMask status_mask_;							// 状态掩码
  StatusCondition status_condition_;		// 状态条件
  InstanceHandle_t instance_handle_;    // 实体句柄
  
  bool enable_;
};

class DomainEntity: public Entity
{
  
};
```

#### status

每个实体都与一组状态对象相关联，这些状态对象的值表示该实体的通信状态。这些状态值的更改会触发调用适当的侦听器回调以异步通知应用程序。

```c++
struct BaseStatus
{
  	int32_t total_count = 0;
  	int32_t total_count_change = 0;
};

#define FASTDDS_STATUS_COUNT size_t(16)

// 状态掩码是 16 位的bitset
class StatusMask: public std::bitset<FASTDDS_STATUS_COUNT>
{
  
};
```

状态的更改会触发相应的侦听事件。对于名为 XXXStatus 的给定状态对象，实体侦听器接口定义了一个回调函数 on_XXX()，当状态更改时将调用该函数。

#### StatusCondition

每个实体都拥有一个 StatusCondition，每当其启用状态发生变化时都会收到通知。 StatusCondition 提供实体和等待集之间的链接。

条件和等待集为应用程序提供了一种替代机制，通过 StatusCondition 使其了解状态对象的更改。

[官方文档中的status说明](https://fast-dds.docs.eprosima.com/en/latest/fastdds/dds_layer/core/status/status.html)

#### enable entity

所有实体都可以创建为启用或不启用。默认情况下，工厂配置为创建启用的实体，但可以使用启用的工厂上的 EntityFactoryQosPolicy 进行更改。

#### Qos

服务质量 (QoS) 用于指定服务的行为，允许用户定义每个实体的行为方式。为了增加系统的灵活性，QoS被分解为多个可以独立配置的QoS策略。但是配置多个 Qos 策略也许会冲突。

