## UDP Transport

UDP 是一种无连接传输，其中接收 DomainParticipant 必须打开侦听传入消息的 UDP 端口，并且发送 DomainParticipant 将消息发送到此端口。



#### UDPTransportDescriptor

```c++
struct UDPTransportDescriptor : public SocketTransportDescriptor
{
		uint16_t m_output_udp_socket;		//发数据的端口
		bool non_blocking_send = false;	// 是否非阻塞
};

// 基于socket 的传输层描述，UDP 或者 TCP 都会继承于此
struct SocketTransportDescriptor : public PortBasedTransportDescriptor
{
  	//发送缓冲区大小
     uint32_t sendBufferSize;
    //接收缓冲区大小
    uint32_t receiveBufferSize;
    //网络接口白名单
    std::vector<std::string> interfaceWhiteList;
    //netmask 过滤配置
    NetmaskFilterKind netmask_filter;
    std::vector<AllowedNetworkInterface> interface_allowlist;
    std::vector<BlockedNetworkInterface> interface_blocklist;
    uint8_t TTL;
};

// 基于端口的传输层描述-> 继承了传输层额描述
// 其中设置了对应端口线程的亲和设置
class PortBasedTransportDescriptor : public TransportDescriptorInterface
{
  
  	//默认线程的线程设置
  	ThreadSettings default_reception_threads_;

    // 对应端口线程的设置
    ReceptionThreadsConfigMap reception_threads_;
};

using ReceptionThreadsConfigMap = std::map<uint32_t, ThreadSettings>;
// 线程设置类
struct RTPS_DllAPI ThreadSettings
{
    // 调度策略
    int32_t scheduling_policy = -1;
		// 优先级
    int32_t priority = std::numeric_limits<int32_t>::min();
  	// 线程亲和
    uint64_t affinity = 0;
		// 栈大小
    int32_t stack_size = -1;
};
```

#### UDPTransportInterface

```c++
class UDPTransportInterface : public TransportInterface
{
    asio::io_service io_service_;			//asio句柄
    mutable std::recursive_mutex mInputMapMutex;
    std::map<uint16_t, std::vector<UDPChannelResource*>> mInputSockets;
  
    uint32_t mSendBufferSize;
    uint32_t mReceiveBufferSize;
    eprosima::fastdds::statistics::rtps::OutputTrafficManager statistics_info_;
 	  bool first_time_open_output_channel_;
   	NetmaskFilterKind netmask_filter_;
    std::vector<AllowedNetworkInterface> allowed_interfaces_;
  	
  	//核心函数
  		// 发送数据到指定远端定位中
      bool send(
            const fastrtps::rtps::octet* send_buffer,
            uint32_t send_buffer_size,
            eProsimaUDPSocket& socket,
            const Locator& remote_locator,
            bool only_multicast_purpose,
            bool whitelisted,
            const std::chrono::microseconds& timeout);
};
```

#### UDPv4TransportDescriptor

UDPv4 和 UDPv6 都有一个描述符类，继承了 UDPTransportDescriptor。可以用于创建 UDPv4Transport / UDPv6Transport

```c++
struct UDPv4TransportDescriptor : public UDPTransportDescriptor
{
  	// 创建 UDP 传输层
    virtual TransportInterface* create_transport() const override
    {
         return new UDPv4Transport(*this);
    }
};
```

#### UDPv4Transport

UDP 传输层实现，继承了 UDPTransportInterface

```c++
class UDPv4Transport : public UDPTransportInterface
{	
  	 // 需要 UDPv4 传输描述符构建
     RTPS_DllAPI UDPv4Transport(
            const UDPv4TransportDescriptor&);
  	
  	// 传输层描述符作为配置
  	UDPv4TransportDescriptor configuration_;
  	// interface 白名单
  	std::vector<asio::ip::address_v4> interface_whitelist_;
};

const char* const DEFAULT_METATRAFFIC_MULTICAST_ADDRESS = "239.255.0.1";
```

#### 简单使用

```c++
// 使用域参与者 Qos 设置
DomainParticipantQos qos;

// 创建一个 UDPv4的传输层描述
auto udp_transport = std::make_shared<UDPv4TransportDescriptor>();
udp_transport->sendBufferSize = 9216;
udp_transport->receiveBufferSize = 9216;
udp_transport->non_blocking_send = true;

// 设置线程相关属性，亲和性，优先级，栈大小，调度策略等等
udp_transport->default_reception_threads(eprosima::fastdds::rtps::ThreadSettings{2, 2, 2, 2});
udp_transport->set_thread_config_for_port(12345, eprosima::fastdds::rtps::ThreadSettings{3, 3, 3, 3});

// qos 中加入
qos.transport().user_transports.push_back(udp_transport);

// 禁用默认的传输层
qos.transport().use_builtin_transports = false;
```

