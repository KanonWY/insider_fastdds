## Transport API

#### 1、TransportDescriptorInterface

传输介质描述符抽象接口，继承该类的类被称为传输介质描述，可以使用传输介质描述来创建对应的传输层。

```c++
struct TransportDescriptorInterface
{
    //传输中单个消息的最大大小。
    uint32_t maxMessageSize;

    //与每个初始远程对等点打开的通道数。
    uint32_t maxInitialPeersRange;
private:
  	mutable std::mutex mtx_;
};
```

TransportDescriptorInterface 的任何实现都应根据需要添加尽可能多的数据成员，以完全配置它所描述的传输。

#### 2、TransportInterface

Transport 是任何实现 TransportInterface 的类。它是负责实际物理传输执行消息分发的对象。

每个 Transport 类定义了自己的`类型`，一个唯一的标识符，用于检查 Locator 与 Transport 的兼容性，即确定 Locator 是否引用 Transport。

```c++
// 传输通道抽象类
class RTPS_DllAPI TransportInterface{
		int32_t transport_kind_;		//传输类型的唯一标识符。
};
```

标识符的对应关系

| Identifier              | Value | Transport type                                               |
| ----------------------- | ----- | ------------------------------------------------------------ |
| `LOCATOR_KIND_RESERVED` | 0     | None. Reserved value for internal use.                       |
| `LOCATOR_KIND_UDPv4`    | 1     | [UDP Transport](https://fast-dds.docs.eprosima.com/en/latest/fastdds/transport/udp/udp.html#transport-udp-udp) over IPv4. |
| `LOCATOR_KIND_UDPv6`    | 2     | [UDP Transport](https://fast-dds.docs.eprosima.com/en/latest/fastdds/transport/udp/udp.html#transport-udp-udp) over IPv6. |
| `LOCATOR_KIND_TCPv4`    | 4     | [TCP Transport](https://fast-dds.docs.eprosima.com/en/latest/fastdds/transport/tcp/tcp.html#transport-tcp-tcp) over IPv4. |
| `LOCATOR_KIND_TCPv6`    | 8     | [TCP Transport](https://fast-dds.docs.eprosima.com/en/latest/fastdds/transport/tcp/tcp.html#transport-tcp-tcp) over IPv6. |
| `LOCATOR_KIND_SHM`      | 16    | [Shared Memory Transport](https://fast-dds.docs.eprosima.com/en/latest/fastdds/transport/shared_memory/shared_memory.html#transport-sharedmemory-sharedmemory). |

#### 3、Locator

Locator_t 唯一地标识与特定传输的远程对等点的通信通道。例如，在 UDP 传输上，定位器将包含远程对等点的 IP 地址和端口信息。所谓的定位器就是可以通过该对象定位到远端。

该 Locator 没有做抽象的类继承处理，有点类似于 Unit 对象或者 variant 对象，基于不同的传输层实现有不同的内容。

```c++
using Locator = eprosima::fastrtps::rtps::Locator_t;
class Locator_t
{
  	    /**
     * @brief Specifies the locator type. Valid values are:
     *
     * LOCATOR_KIND_UDPv4
     *
     * LOCATOR_KIND_UDPv6
     *
     * LOCATOR_KIND_TCPv4
     *
     * LOCATOR_KIND_TCPv6
     *
     * LOCATOR_KIND_SHM
     */
    int32_t kind;
    /// Network port
    uint32_t port;
    /// IP address
    octet address[16];
};
```

原本 port 使用 short（uint16_t)存储就应该足够，但是这里是 4 字节，因为需要做额外的处理。

对于 TCP 而言，它把 port 分为了两个部分：

- 物理端口是网络设备使用的端口，是操作系统理解的真实端口。它存储在成员端口的两个最低有效字节中。

- 逻辑端口是RTPS端口。 RTPS协议使用它来区分不同的实体。它存储在成员端口的两个最高有效字节中。

在 TCP 中，这种区别允许使用不同 RTPS 端口（逻辑端口）的多个 DDS 应用程序共享相同的物理端口，因此只需要为所有通信打开一个端口。 

对于 UDP 而言只有物理端口，也就是RTPS端口，存储在成员端口的最低两个字节中。

定位器地址以 16 字节表示，根据使用的协议是 IPv4 还是 IPv6，其管理方式有所不同。

（1）IPv6 地址使用 16 个可用字节来表示唯一的全局地址。

（2）IPv4 地址将这 16 个字节分为以下三个部分，按重要性从最小到最大的顺序排列：

- 4 bytes LAN IP: Local subnet identification (UDP and TCP).
- 4 bytes WAN IP: Public IP (TCP only).
- 8 bytes unused.

```c++
                        Locator IPv4 address
+--------+-----------------------------+-----------------------------+
| Unused | WAN address (62.128.41.210) | LAN address (192.168.0.113) |
+--------+-----------------------------+-----------------------------+
 8 bytes       (TCP only) 4 bytes                 4 bytes
```

#### 4、IPLocator

IPLocator 是一个辅助静态类，提供操作基于 IP 的定位器的方法。设置新的 UDP 传输或 TCP 传输时非常方便，因为它简化了 IPv4 和 IPv6 地址的设置或端口的操作。IPLocator 无需也无法创建，因为它删除了构造和析构函数。

```c++
// 它就是一个静态的方法对象
class IPLocator 
{
  	    // TCP
    //! Sets locator's logical port (as in RTCP protocol)
    RTPS_DllAPI static bool setLogicalPort(
            Locator_t& locator,
            uint16_t port);

    //! Gets locator's logical port (as in RTCP protocol)
    RTPS_DllAPI static uint16_t getLogicalPort(
            const Locator_t& locator);
  
  private:
    IPLocator() = delete;
    ~IPLocator() = delete;
};
```

官方文档上的简单使用的例子：

```c++
// 配置 TCP 定位器
Locator_t locator;

// 获取一个定位器的物理端口
uint16_t physical_port = IPLocator::getPhysicalPort(locator);
// 设置一个定位器的物理端口
IPLocator::setPhysicalPort(locator, 5555);

// 获取逻辑端口
uint16_t logical_port = IPLocator::getLogicalPort(locator);
// 设置端口
IPLocator::setLogicalPort(locator, 7400);

// 设置公网地址
IPLocator::setWan(locator, "80.88.75.55");
```

此外 IPLocator 还支持根据主机可用的 DNS 来解析域名

```c++
Locator_t locator;
// 从本机 DNS 中解析
auto response = eprosima::fastrtps::rtps::IPLocator::resolveNameDNS("localhost");
// Get the first returned IPv4
if (response.first.size() > 0)
{
    IPLocator::setIPv4(locator, response.first.begin()->data());
    locator.port = 11811;
}
```

#### ChainingTransportDescriptor

在某些用例中，用户需要在将输出信息发送到网络之前以及在接收到传入信息后对其进行预处理。Transport API 提供了两个接口来实现此类功能：ChainingTransportDescriptor 和 ChainingTransport。

简而言之就是支持数据接收后首先处理和发送前处理。

```c++
typedef struct ChainingTransportDescriptor : public TransportDescriptorInterface
{
  //低层次的传输通路
  std::shared_ptr<TransportDescriptorInterface> low_level_descriptor;
};
```

```c++
class ChainingTransport : public TransportInterface
{
  
    std::unique_ptr<TransportInterface> low_level_transport_;
};
```



基本使用

```c++

DomainParticipantQos qos;

//创建一个 UDPv4 传输通路
auto udp_transport = std::make_shared<UDPv4TransportDescriptor>();

// 创建 custom 通路
auto custom_transport = std::make_shared<CustomChainingTransportDescriptor>(udp_transport);
qos.transport().user_transports.push_back(custom_transport);
qos.transport().use_builtin_transports = false;
```

