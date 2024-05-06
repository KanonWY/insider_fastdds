## Subscriber

订阅者代表属于它的一个或多个 DataReader 对象进行操作。它充当一个容器，允许在订阅者的 SubscriberQos 给定的通用配置下对不同的 DataReader 对象进行分组。属于同一订阅服务器的 DataReader 对象除了订阅服务器的 SubscriberQos 之外，彼此之间没有任何其他关系，并且在其他情况下独立运行。具体来说，订阅者可以托管不同主题和数据类型的 DataReader 对象。





#### SubscriberListener

SubscriberListener 是一个抽象类，定义响应订阅服务器上的状态更改而触发的回调。默认情况下，所有这些回调都是空的并且不执行任何操作。

用户应该实现此类的专门化，以覆盖应用程序所需的回调。未被覆盖的回调将保持其空实现。

SubscriberListener 继承自 DataReaderListener。因此，它能够对报告给 DataReader 的所有事件做出反应。

由于事件始终会通知到可以处理该事件的最具体的实体侦听器，**因此只有当触发 DataReader 没有附加侦听器，或者 DataReader 上的 StatusMask 禁用了回调时**，才会调用 SubscriberListener 从 DataReaderListener 继承的回调。

此外，SubscriberListener 添加以下回调:

on_data_on_readers():

新数据在属于该订阅者的任何 DataReader 上可用。此回调的调用没有排队，这意味着如果一次接收到多个新数据更改，则只能为所有这些更改发出一个回调调用，而不是每次更改一个。如果应用程序在此回调中检索接收到的数据，则它必须继续读取数据，直到没有新的更改为止。



#### 创建订阅器

订阅者始终属于域参与者。订阅者的创建是通过 DomainParticipant 实例上的 create_subscriber() 成员函数完成的，该实例充当订阅者的工厂。