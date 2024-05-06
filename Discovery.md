## Discovery

Fast DDS 提供了发现机制允许跨越域内节点自动查找和匹配数据读写器，以便他们开始共享数据。

对于所有的机制，发现分为两个阶段。

- 1、参与者发现阶段（PDP)

在该阶段内，域内节点互相发现其存在。为此，每一个域内节点，定期发送公告消息。当两个给定的 DomainParticipant 存在于同一 DDS 域中时，它们将匹配。默认情况下，使用众所周知的多播地址和端口发送公告消息。此外，可以指定使用单播发送公告的地址列表（请参阅初始对等点）。此外，还可以配置此类公告的周期（请参阅发现配置）。



- 2、端点发现阶段（EDP）

在此阶段，数据写入器和数据读取器相互确认。为此，域内节点使用 PDP 期间建立的通信通道相互共享有关其 DataWriter 和 DataReader 的信息。

该信息包含主题和数据类型等内容（请参阅主题）。为了使两个端点匹配，它们的主题和数据类型必须一致。一旦 DataWriter 和 DataReader 匹配，它们就准备好发送/接收用户数据流量。

这与 eCAL 的发现机制也比较类似，但是目前 eCAL 没有对于不同的发现域进行特定的实现。

## 发现机制

- 简单发现

默认发现机制，它支持 PDP 和 EDP 的 RTPS 标准，因此提供与任何其他 DDS 和 RTPS 实现的兼容性。

- 静态发现

允许手动配置数据读写器和话题。

- 发现服务器

类似于 ros1 的master 机制，允许一个节点充当数据元节点。

- 手动发现

该机制禁用 PDP 阶段，让用户自己指定域内节点之间的消息。



## 发现设置

- 使用 Qos 配置发现机制

```c++
DomainParticipantQos pqos;
pqos.wire_protocol().buildin.discovery_config.discoveryProtocol = DiscoverProtocol_t::SIMPLE;
```

- 支持流量过滤

  [<官方支持过滤选项>](https://fast-dds.docs.eprosima.com/en/latest/fastdds/discovery/general_disc_settings.html)

```c++
DomainParticipantQos pqos;

// 当前主机的发现流量将被过滤
pqos.wire_protocol().builtin.discovery_config.ignoreParticipantFlags =
        static_cast<eprosima::fastrtps::rtps::ParticipantFilteringFlags_t>(
    ParticipantFilteringFlags_t::FILTER_DIFFERENT_PROCESS |
    ParticipantFilteringFlags_t::FILTER_SAME_PROCESS);
```

- 租赁期限

指示远程 DomainParticipant 应在多长时间内认为本地 DomainParticipant 处于活动状态。如果本地DomainParticipant的活跃度在这段时间内没有被断言，则远程DomainParticipant认为本地DomainParticipant已死亡并销毁有关本地DomainParticipant及其所有端点的所有信息。

每当远程 DomainParticipant 从本地 DomainParticipan 接收到任何类型的流量时，都会在远程 DomainParticipant 上断言本地 DomainParticipant 的活跃性。

租约持续时间使用 Duration_t 指定为以秒和纳秒表示的时间。

```c++
DomainParticipantQos pqos;

pqos.wire_protocol().builtin.discovery_config.leaseDuration = Duration_t(10, 20);
```

- 公告周期

设置 PDP 公告的周期，

为了保持活跃性，建议公告周期短于租用期限，这样即使没有数据流量，DomainParticipant 的活跃性也能得到保证。值得注意的是，公告周期的设置涉及到一个权衡，即过于频繁的公告会使元流量使网络膨胀，但过于稀少的公告会延迟发现迟到的加入者。

DomainParticipant 的公告周期使用 Duration_t 指定为以秒和纳秒表示的时间。

```c++
DomainParticipantQos pqos;

pqos.wire_protocol().builtin.discovery_config.leaseDuration_announcementperiod = Duration_t(1, 2);
```



### 简单发现设置

[简单发现设置与属性](https://fast-dds.docs.eprosima.com/en/latest/fastdds/discovery/simple.html)

- SPDP

简单参与者发现协议：

指定DomainParticipants如何在网络中发现彼此；它宣布并检测同一域中是否存在 DomainParticipants。

- SEDP

简单端点发现协议：

定义了所发现的DomainParticipant所采用的用于信息交换的协议，以便发现它们中包含的DDS实体，即DataWriter和DataReader。

#### Initial Announcements

RTPS 标准简单发现机制要求 DomainParticipants 发送其在域中存在的公告。这些公告不是以可靠的方式传递的，并且可以由网络处理。

为了避免消息处理引起的发现延迟，可以将初始通告设置为多次发射，以增加正确接收的机会。请参阅初始公告配置。

#### Sample EDP Attribute

```c++
DomainParticipantQos pqos;

pqos.wire_protocol().builtin.discovery_config.use_SIMPLE_EndpointDiscoveryProtocol = true;
pqos.wire_protocol().builtin.discovery_config.m_simpleEDP.use_PublicationWriterANDSubscriptionReader = true;
pqos.wire_protocol().builtin.discovery_config.m_simpleEDP.use_PublicationReaderANDSubscriptionWriter = false;
```

#### Initial peers

每个 RTPSParticipant 必须在两个不同的端口中侦听传入的参与者发现协议 (PDP) 发现元流量，一个端口链接到多播地址，另一个端口链接到单播地址。

fastDDS 允许配置初始对等点列表，其中包含一个或多个与远程 DomainParticipants PDP 发现侦听资源相对应的 IP 端口地址对，这样，本地 DomainParticipant 不仅会将其 PDP 流量发送到其域指定的默认多播地址端口，还会发送到初始对等点列表中指定的所有 IP 端口地址对。

简而言之就是支持将PDP流向发送到指定的额外端口。

DomainParticipant 的初始对等方列表包含与其通信的所有其他 DomainParticipant 的 IP 端口地址对列表。它是 DomainParticipant 将在单播发现机制中一起使用或作为多播发现的替代方案使用的地址列表。

因此，该方法也适用于那些组播功能不可用的场景。

根据 RTPS 标准（第 9.6.1.1 节），RTPSParticipants 的发现流量单播侦听端口使用以下公式计算：

7400 + 250 * 域ID + 10 + 2 * 参与者ID。

例如，如果 RTPSParticipant 在域 0（默认域）中运行且其 ID 为 1，则其发现流量单播侦听端口将为：7400 + 250 * 0 + 10 + 2 * 1 = 7412。默认情况下，eProsima Fast DDS 使用作为初始端口对等元流量多播定位器。

## 发现配置

