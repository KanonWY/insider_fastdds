## resource

```c++
// channel 资源基类
class ChannelResource
{
    // 接收到的消息，CDRMessage_t 是消息的封装
  	fastrtps::rtps::CDRMessage_t message_buffer_;
  	// 是否存活
    std::atomic<bool> alive_;
    eprosima::thread thread_;				
};


class UDPChannelResource : public ChannelResource
{		
 	// 构建 
  UDPChannelResource(
            UDPTransportInterface* transport,
            eProsimaUDPSocket& socket,
            uint32_t maxMsgSize,
            const Locator& locator,
            const std::string& sInterface,
            TransportReceiverInterface* receiver,
            const ThreadSettings& thread_config);
  
  
  // 接收数据
 bool Receive(
            fastrtps::rtps::octet* receive_buffer,
            uint32_t receive_buffer_capacity,
            uint32_t& receive_buffer_size,
            Locator& remote_locator);
  
  	// writer 和 reader 关联
    TransportReceiverInterface* message_receiver_; 
    eProsimaUDPSocket socket_;
    bool only_multicast_purpose_;
    std::string interface_;
    UDPTransportInterface* transport_;		//对应的 UDP 传输层
};
```

UDPChannelResource 如何构建的？