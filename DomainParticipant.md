## DomainParticipant

域参与者代表了一个在 local DomainID 中运行的。它也是采用 pImp 实现的，并且只能使用
友元类创建，自己无法直接创建。
```c++
class DomainParticipant : public Entity {

protected:

    // 构造函数, 无法直接构建，只能在友元类中使用构建
    DomainParticipant(
            const StatusMask& mask = StatusMask::all())
    : Entity(mask)
    , impl_(nullptr)
    {
    }

    // 实际实现
    DomainParticipantImpl* impl_;

    friend class DomainParticipantFactory;
    friend class DomainParticipantImpl;
    friend class ::dds::domain::DomainParticipant;
public:
    // 核心函数
    // (1) 设置 Qos，获取 Qos
    // (2) 获取监听函数，设置监听函数
    // (3) 创建发布者，根据配置文件创建发布者
    // (4) 创建监听者，根据配置文件创建监听者
    // (5) 注册类型
    ////// 以上函数仅仅是接口实际是调用 impl_ 中的函数。
};

```

#### DomainParticipantImpl
域参与者实现是域参与者的内部实现，它必须接收域参与者指针构建

```c++
class DomainParticipantImpl
{
    // 设置友元类，使用友元可以直接访问
    friend class DomainParticipantFactory;
    friend class DomainParticipant;
    friend class ReaderFilterCollection;

protected:
    // 构造函数是 protected，因此只能被友元函数调用
    DomainParticipantImpl(
            DomainParticipant* dp,
            DomainId_t did,
            const DomainParticipantQos& qos,
            DomainParticipantListener* listen = nullptr)
    {
protected:

    DomainId_t domain_id_;
    int32_t participant_id_ = -1;
    fastrtps::rtps::GUID_t guid_;
    std::atomic<uint32_t> next_instance_id_;
    DomainParticipantQos qos_;

    // 执行 rtps 层的域参与者对象指针
    fastrtps::rtps::RTPSParticipant* rtps_participant_;
    // 指向实现的域参与者
    DomainParticipant* participant_;

    //!Participant Listener
    DomainParticipantListener* listener_;

    mutable std::mutex mtx_gs_;
    std::condition_variable cv_gs_;

    // 发布者 map
    std::map<Publisher*, PublisherImpl*> publishers_;
    std::map<InstanceHandle_t, Publisher*> publishers_by_handle_;
    // 发布者互斥锁
    mutable std::mutex mtx_pubs_;
    // 默认发布者 Qos
    PublisherQos default_pub_qos_;

    // 订阅者 map
    std::map<Subscriber*, SubscriberImpl*> subscribers_;
    std::map<InstanceHandle_t, Subscriber*> subscribers_by_handle_;
    // 订阅者互斥锁
    mutable std::mutex mtx_subs_;
    // 默认订阅者 Qos
    SubscriberQos default_sub_qos_;

    // 话题数据类型
    std::map<std::string, TypeSupport> types_;
    mutable std::mutex mtx_types_;

    // 话题
    std::map<std::string, TopicProxyFactory*> topics_;
    std::map<InstanceHandle_t, Topic*> topics_by_handle_;

    // 内容过滤话题
    std::map<std::string, std::unique_ptr<ContentFilteredTopic>> filtered_topics_;
    std::map<std::string, IContentFilterFactory*> filter_factories_;
    DDSSQLFilter::DDSFilterFactory dds_sql_filter_factory_;

    mutable std::mutex mtx_topics_;
    std::condition_variable cond_topics_;

    // 默认 Topic Qos
    TopicQos default_topic_qos_;

    // Mutex for requests and callbacks maps.
    std::mutex mtx_request_cb_;

    // register_remote_type parent request, type_name, callback relationship.
    std::map<fastrtps::rtps::SampleIdentity,
            std::pair<std::string, std::function<void(
                const std::string& name,
                const fastrtps::types::DynamicType_ptr)>>> register_callbacks_;

    // Relationship between child and parent request
    std::map<fastrtps::rtps::SampleIdentity, fastrtps::rtps::SampleIdentity> child_requests_;

    // All parent's child requests
    std::map<fastrtps::rtps::SampleIdentity, std::vector<fastrtps::rtps::SampleIdentity>> parent_requests_;

    std::atomic<uint32_t> id_counter_;

};
```

#### 追踪类型注册
查看域参与者的类型注册的实际调用链. 
存储在 types_ 中，

```c++
/**
 * @brief 注册类型
 * @param type 收发数据类型
 * @param type_name 类型名字
 */
ReturnCode_t DomainParticipantImpl::register_type(
        const TypeSupport type,
        const std::string& type_name)
{
    if (type_name.size() <= 0)
    {
        EPROSIMA_LOG_ERROR(PARTICIPANT, "Registered Type must have a name");
        return ReturnCode_t::RETCODE_BAD_PARAMETER;
    }
    // 根据名字在 map 中查找类型
    TypeSupport t = find_type(type_name);

    if (!t.empty())
    {
        if (t == type)
        {
            return ReturnCode_t::RETCODE_OK;
        }
        // 类型不一致返回错误
        EPROSIMA_LOG_ERROR(PARTICIPANT, "Another type with the same name '" << type_name << "' is already registered.");
        return ReturnCode_t::RETCODE_PRECONDITION_NOT_MET;
    }

    // 插入
    std::lock_guard<std::mutex> lock(mtx_types_);
    types_.insert(std::make_pair(type_name, type));

    // TODO: 动态类型
    if (type->auto_fill_type_object() || type->auto_fill_type_information())
    {
        register_dynamic_type_to_factories(type);
    }

    return ReturnCode_t::RETCODE_OK;
}
```
#### DomainParticipant 创建流程

由于 DomainParticipant 只能由友元函数访问构造函数，因此该对象只能使用 Factory 进行创建。
```c++

// Factory 中有存储 域参与者的 map

 // <域名ID -> 域参与者实现>
 std::map<DomainId_t, std::vector<DomainParticipantImpl*>> participants_;


```
```c++

DomainParticipant* DomainParticipantFactory::create_participant(
          DomainId_t did,
          const DomainParticipantQos& qos,
          DomainParticipantListener* listen,
          const StatusMask& mask
        )
{
    // 记载配置
    load_profiles();
    // 是否使用 默认的 qos
    //...

    // 创建 DomainParticipant
    DomainParticipant* dom_part = new DomainParticipant(mask);
    DomainParticipantImpl* dom_part_impl = new DomainParticipantImpl(dom_part, did, pqos, listen);
   
    // GUID 是否分配成功 
    if (fastrtps::rtps::GUID_t::unknown() != dom_part_impl->guid())
    {
        {
            std::lock_guard<std::mutex> guard(mtx_participants_);
            // 查找 map 中是否存在
            auto vector_it = participants_.find(did);

            // 插入操作
            if (vector_it == participants_.end())
            {
                // Insert the vector
                std::vector<DomainParticipantImpl*> new_vector;
                auto pair_it = participants_.insert(std::make_pair(did, std::move(new_vector)));
                vector_it = pair_it.first;
            }

            vector_it->second.push_back(dom_part_impl);
        }

        // 默认创建实体
        if (factory_qos_.entity_factory().autoenable_created_entities)
        {
            // 核心函数 enable, enable 函数调用后会自动将所有的默认后台线程启动起来。之后此域参与者实体可以正常工作了
            if (ReturnCode_t::RETCODE_OK != dom_part->enable())
            {
                delete_participant(dom_part);
                return nullptr;
            }
        }
    }
    else
    {
        delete dom_part_impl;
        return nullptr;
    }

    // 最终返回 DomainParticipant 的指针，内部的 impl 已经创建完毕了
    return dom_part; 

}

```
#### enable 启动核心函数
DomainParticipant 的 enable 函数会调用 impl 的 enable 函数
```c++
/**
 * @brief 启动域参与者函数
 * @note 此函数会启动 rtps 层的实际传输类
 *
 */
ReturnCode_t DomainParticipantImpl::enable()
{
    // 确保 rtps participant 未启动
    assert(get_rtps_participant() == nullptr);
    // 确保 GUID 成功分配
    assert (guid_ != GUID_t::unknown());

    // 从 Domain 域参与者中提取出 rtps 域参与者的 Qos
    fastrtps::rtps::RTPSParticipantAttributes rtps_attr;
    utils::set_attributes_from_qos(rtps_attr, qos_);
    rtps_attr.participantID = participant_id_;

    RTPSParticipant* part = RTPSDomainImpl::clientServerEnvironmentCreationOverride(
        domain_id_,
        false,
        rtps_attr,
        &rtps_listener_);

    if (part == nullptr)
    {
        // 创建 RTPSParticipant 
        part = RTPSDomain::createParticipant(domain_id_, false, rtps_attr, &rtps_listener_);

        if (part == nullptr)
        {
            EPROSIMA_LOG_ERROR(DOMAIN_PARTICIPANT, "Problem creating RTPSParticipant");
            return ReturnCode_t::RETCODE_ERROR;
        }
    }

    guid_ = part->getGuid();

    {
        // 设置 rtps
        std::lock_guard<std::mutex> _(mtx_gs_);
        rtps_participant_ = part;

        part->set_check_type_function(
            [this](const std::string& type_name) -> bool
            {
                return find_type(type_name).get() != nullptr;
            });
    }
    // 默认创建实体
    if (qos_.entity_factory().autoenable_created_entities)
    {
        // Enable topics first
        {
            std::lock_guard<std::mutex> lock(mtx_topics_);

            for (auto topic : topics_)
            {
                topic.second->enable_topic();
            }
        }

        // Enable publishers
        {
            std::lock_guard<std::mutex> lock(mtx_pubs_);
            for (auto pub : publishers_)
            {
                pub.second->rtps_participant_ = part;
                pub.second->user_publisher_->enable();
            }
        }

        // Enable subscribers
        {
            std::lock_guard<std::mutex> lock(mtx_subs_);

            for (auto sub : subscribers_)
            {
                sub.second->rtps_participant_ = part;
                sub.second->user_subscriber_->enable();
            }
        }
    }

    part->enable();

    return ReturnCode_t::RETCODE_OK;
}
```

