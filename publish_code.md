## publisher

发布者属于`DomainEntity`。
```c++
class Publisher: public DomainEntity {
protected:
    ////////////////////////////////////////////////
    // 友元类
    ////////////////////////////////////////////////
    friend class PublishImpl;
    frient class DomainParticipantImpl;

    ////////////////////////////////////////////////
    // 构造函数, 保护，只能由友元类访问构造
    ////////////////////////////////////////////////
    Publisher(
            PublisherImpl* p,
            const StatusMask& mask = StatusMask::all());

    Publisher(
            DomainParticipant* dp,
            const PublisherQos& qos = PUBLISHER_QOS_DEFAULT,
            PublisherListener* listener = nullptr,
            const StatusMask& mask = StatusMask::all());

//.....

protected:
    PublisherImpl* impl_;
};
```

#### 发布者实现

```c++
class PublisherImpl
{
protected:
    ////////////////////////////////////////////////
    // 友元类
    ////////////////////////////////////////////////
    friend class DomainParticipantImpl;
    friend class DataWriterImpl;

    ////////////////////////////////////////////////
    // 构造函数, 保护，只能由友元类访问构造
    ////////////////////////////////////////////////
 
    PublisherImpl(
            DomainParticipantImpl* p,
            const PublisherQos& qos,
            PublisherListener* p_listen = nullptr);

protected:

    // 域参与者实现
    DomainParticipantImpl* participant_;
    // 发布者 Qos
    PublisherQos qos_;
    //  <话题名, <写入器列表>>
    std::map<std::string, std::vector<DataWriterImpl*>> writers_;
    mutable std::mutex mtx_writers_;

    // 发布者监听回调
    PublisherListener* listener_;

    class PublisherWriterListener : public DataWriterListener
    {
    public:

        PublisherWriterListener(
                PublisherImpl* p)
            : publisher_(p)
        {
        }

        virtual ~PublisherWriterListener() override
        {
        }

        void on_publication_matched(
                DataWriter* writer,
                const PublicationMatchedStatus& info) override;

        void on_offered_deadline_missed(
                DataWriter* writer,
                const fastrtps::OfferedDeadlineMissedStatus& status) override;

        void on_liveliness_lost(
                DataWriter* writer,
                const LivelinessLostStatus& status) override;

        PublisherImpl* publisher_;
    }
    publisher_listener_; // 发布写入回调

    // 指向的发布者指针
    Publisher* user_publisher_;
    
    // 指向的 rtps域参与者指针
    fastrtps::rtps::RTPSParticipant* rtps_participant_;

    // 默认的数据写入器 Qos
    DataWriterQos default_datawriter_qos_;

    fastrtps::rtps::InstanceHandle_t handle_;
  
};
```
