## Topic

#### 1、IDL 与 话题

```idl
struct HelloWorld
{
		unsigned long index;
		string message;
};
```

在直接使用 fast-dds 的静态话题发送的时候，需要先定义 IDL，然后使用 fast-dds-gen 生成 cpp 代码。默认情况下会生成多个文件

```bash
kanon.h				IDL 中的类型定义
kanon.cxx			IDL 中的类型定义
kanonCdrAux.ipp		序列化和反序列化代码
kanonCdrAux.hpp		序列化和反序列化代码
kanonPubSubTypes.h		FastDDS 使用接口
kanonPubSubTypes.cxx	FastDDS 使用接口
```

其中 kanonPubSubTypes.h 是用来给用户使用的，其中生成的类都是继承了 DataTopicType 的类，可以使用这些类来创建 TypeSupport，然后注册到域参与者中。

#### 2、话题描述 (TopicDescription)

TopicDescription 是一个抽象类，用作描述数据流的所有类的基础。应用程序不会直接创建 TopicDescription 的实例，而是创建它的派生类。（Topic/ContentFilterTopic）

话题描述

```c++
class TopicDescription
{
		std::string name_;
		std::string	type_name_;	//类型名
};
```

#### 3、TopicDataType & TypeSupport

TopicDataType描述了发布和订阅之间交换的数据类型，即对应于Topic的数据。用户必须为应用程序将使用的每种特定类型创建一个专门的类。

// **TODO: 注册过程**

TypeSupport 对象封装 TopicDataType 的实例，提供注册类型以及与发布和订阅交互所需的函数。

```c++
class TopicDataType
{
  
};

class TypeSupport: public std::shared_ptr<fastdds::dds::TopicDataType>
{
   explict TypeSupport(fastdds::dds::TopicDataType* ptr)
   	: std::shared_ptr<fastdds::dds::TopicDataType>(ptr)
   {}
};
```

普通数据话题创建

```c++
DomainParticipant* participant =
        DomainParticipantFactory::get_instance()->create_participant(0, PARTICIPANT_QOS_DEFAULT);
if (nullptr == participant)
{
    // Error
    return;
}

// 使用话题数据类型创建 Typesupport
TypeSupport custom_type_support(new CustomDataType());
// 如果为 nullptr 使用数据类型本身的值作为名字
custom_type_support.register_type(participant, nullptr);

// 再次注册没用，也不会报错
custom_type_support.register_type(participant, custom_type_support.get_type_name());

// 使用注册了的 datatype 创建话题
Topic* topic =
        participant->create_topic("topic_name", custom_type_support.get_type_name(), TOPIC_QOS_DEFAULT);
if (nullptr == topic)
{
    // Error
    return;
}

custom_type_support.register_type(participant, "data_type_name");
```

动态数据类型

可以按照 OMG Extensible and Dynamic Topic Types for DDS 接口动态定义数据类型，而不是直接编写专门的 TopicDataType 类。数据类型也可以在动态加载的 XML 文件上描述。

```c++
// 创建域参与者
DomainParticipant* participant =
        DomainParticipantFactory::get_instance()->create_participant(0, PARTICIPANT_QOS_DEFAULT);
if (nullptr == participant)
{
    // Error
    return;
}

// 从 xml 中加载类型描述
DomainParticipantFactory::get_instance()->load_XML_profiles_file("text.xml");

// 检索指定名字的数据实例
DynamicType::_ref_type dyn_type;
DomainParticipantFactory::get_instance()->get_dynamic_type_builder_from_xml_by_name("DynamicType", dyn_type);

// 注册动态类型
TypeSupport dyn_type_support(new DynamicPubSubType(dyn_type));
dyn_type_support.register_type(participant, nullptr);

// 创建话题
Topic* topic =
        participant->create_topic("topic_name", dyn_type_support.get_type_name(), TOPIC_QOS_DEFAULT);
if (nullptr == topic)
{
    // Error
    return;
}
```

#### 4、话题（TOPIC）

一个话题归属在域的发现者之下，可以使用`DomainParticipant`的成员函数`create_topic`来创建一个话题。

话题类是一个实体，还继承了话题描述，和 FastDDS 大多数内部实现一样使用了 pIMpl 的技法，保证 API 稳定。

```c++
class Topic: public DomainEntity: public TopicDescription
{
  protected:
  	TopicProxy* *impl_;
};
```

- 基于 API 创建话题

```c++
Topic* create_topic(
	const std::string& topic_name,		//话题名
	const std::string& type_name,			//将传输的已注册数据类型的名称
	const TopicQos& qos,							//描述 Topic 行为的 TopicQos
	TopicListener* listen = nullptr,	//话题监听器
	const StatusMask& mask = StatusMask::all()	//激活或停用触发 TopicListener 上的各个回调的 StatusMask。默认情况下所有事件均已启用
);
```

- 基于配置文件创建话题

```c++
Topic* create_topic_with_profile(
	const std::string& topic_name,		//话题名
	const std::string& type_name,			//将传输的已注册数据类型的名称
	const std::string& profile_name,	//配置文件名
	TopicListener* listen = nullptr,	//话题监听器
	const StatusMask& mask = StatusMask::all()	//激活或停用触发 TopicListener 上的各个回调的 StatusMask。默认情况下所有事件均已启用
);
```

- 删除话题

```bash
ReturnCode_t delete_topic(const Topic* topic);
```

#### 5、内容过滤话题（ContentFilteredTopic）

ContentFilteredTopic 是更广泛的 TopicDescription 概念的特化。 ContentFilteredTopic 是具有过滤属性的主题。它使得订阅主题成为可能，同时指定对主题数据子集的兴趣。ContentFilteredTopic 提供主题（称为相关主题）和一些用户定义的过滤属性之间的关系：

- 过滤表达式，对相关主题的内容建立逻辑表达式。它类似于 SQL 语句中的 WHERE 子句
- 表达式参数列表，为过滤器表达式中存在的参数提供值。过滤器表达式中的每个参数都必须有一个参数字符串。

ContentFilteredTopic 不是实体，因此它既没有 QoS，也没有侦听器。使用 ContentFilteredTopic 创建的 DataReader 将使用相关主题中的 QoS。可以为同一个 ContentFilteredTopic 创建多个 DataReader，并且更改 ContentFilteredTopic 的过滤器属性将影响使用它的所有 DataReader。



https://mie-sub.pz.pe/subscribe/TBOVIC3I1O8GIUIL?clash=ssr&trojan



