---
title: Python3 Asyncio 中发布-订阅者队列的实现
date: 2022-04-26 16:40:26
tags: ["python", "设计模式", "协程"]
categories: ["python"]
author: "Haidong Pang"
---

## 发布者-订阅者模式

在讲发布者-订阅者队列前，不得不先提到其所依据的发布-订阅模式。

### 作用

盲目照搬设计模式是愚蠢的。在使用一种设计模式去进行软件设计时，我们必须要清楚这种方法所解决的问题场景是什么。发布-订阅模式的官方名词解释已经足够明了，总结为一个点，就是对依赖关系进行功能解耦。

若一个主体被一个或多个主体所依赖，对于被依赖主体所产生的事件，所有依赖主体必须做出响应。而被依赖主体无须关心下游功能细节。由此便实现了依赖关系的解耦。

讲到这里，你能想到什么例子？很常见的便是 Vue 中的响应式系统。每个引用了响应式数据对象的主体，在响应式数据对象被更改时都会被更新。而引用主体并不关心是谁更改了他们依赖的响应式数据。

## 发布-订阅者队列实现

### 我需要解决的问题

在大多计算机科班生深恶痛绝的 Operating System 课程中，我需要完成一个任务调度的仿真软件设计。

在处理时钟中断时，我需要保证我设计的所有计算机硬件仿真实例在同一个时钟脉冲下同步（很明显，较真的话，这是仿真软件所不能完成的。在任何软件运行时下，单处理器做并行任务或多或少都有进程切换的时间成本）。所有依赖于时钟脉冲的仿真实例在逻辑上订阅了时钟发布的消息。这是个非常典型的发布-订阅者模式的应用场景。

### 基本结构

![pub-sub queue](https://img.blog.panghaidong.com/202210050006931.svg)

-   Publisher：发布者，产生消息（事件），并将其加入 Input Channel 中；

-   Input Channel：输入管道，事实上我更愿意称之为发布者消息介质，接收发布者产生的消息，并将其推送到 Message Broker；

-   Message Broker：消息代理，在接收到输入管道中传递来的消息时，将其转发到订阅了该消息源的所有订阅者的消息介质中；

-   Output Channel：输出管道，事实上我更愿意称之为订阅者消息介质，顺序存储所有消息代理转发来的订阅源消息，供订阅者读取；

-   Subscriber：订阅者，整个数据链路的末端，从输出管道中获取其所订阅的消息。

### 设计实现

为了能够更轻易地讲明白我的思路，在下面的代码实现中，我并不会遵循==**基本结构**==中所陈述的组件顺序。

#### I/O Channel

输入管道是一个满足队列特性的存储介质。因此，你可以使用任意你所熟悉或匹配当前业务的存储介质，例如 SQLite、Redis 或内存。我们可以使用抽象类型去描述它。

```python
from abc import ABCMeta, abstractmethod

class BaseChannel(metaclass=ABCMeta):
    """
    Abstract class of channel implementation .
    The specific implementation of channel can plugin with any storage ,
    such as Redis, SQLite and native queue .
    """

    @abstractmethod
    async def get(self):
        raise NotImplementedError

    @abstractmethod
    async def put(self, *args, **kwargs):
        raise NotImplementedError
```

显而易见，这个抽象类描述了一个 FIFO 队列的标准行为。协程并不是必要条件，但为了匹配我其他模块的实现，我全部使用 async 关键字来定义管道基类的方法。

那么，我们来实现一个简单的管道子类，这里我使用 Asyncio 中提供的 Queue 作为存储介质。

```python
from asyncio import Queue

class AsyncIOChannel(BaseChannel):
    """
    Simple Asyncio implementation of channel
    """
    messages: Queue

    def __init__(self):
        self.messages = Queue()

    async def get(self):
        """
        Returns the next message available in the queue. Returns None if queue is empty.
        """
        message = await self.messages.get()
        return message

    async def put(self, message):
        """
        Puts the message in the queue.
        """
        await self.messages.put(message)
```

#### Publisher

Publisher 作为整个通信系统的最上游，只需要实现一个 publish 方法，将产生的消息加入 Input Channel 中。在仅传递简单对象（字符串，整形数等）时，其实不需要考虑序列化的问题。但在某些特殊情况下，传递复杂对象也许需要再去编写消息实体的序列化和反序列化方法。

```python
from typing import Callable, Coroutine

class Publisher:
    """
    The publisher class is bound to a channel.
    """
    _channel: BaseChannel
    on_publish: Callable

    def __init__(self, channel: BaseChannel, on_publish: Callable[[], Coroutine]):
        self._channel = channel
        self.on_publish = on_publish

    async def publish(self, message: any):
        """
        The method can be used to publish messages to the binding queue.
        """
        await self._channel.put(message)
        await self.on_publish()
```

这里有个小把戏，我为 Publisher 的 publish 动作注册了一个生命周期回调函数 o n_publish。这个函数的作用就是当 Publisher 完成了 publish 动作后，通知 MessageBroker，对消息进行转发。从而简化了 MessageBroker 的监听机制。

#### Subscriber

每个 Subscriber 实例可以订阅一个信道，且在每个实例内部都会维护一个 Output Channel 。得益于“鸭子”协议，我们可以将 Subscriber 实现为一个可迭代对象，并为它实现异步迭代器的接口。

```python
class Subscriber:
    _channel: BaseChannel

    def __init__(self):
        """
        Instantiate a Output Channel.
        """
        self._channel = AsyncIOChannel()

    def __aiter__(self):
        """
        asynchronous iterator .
        """
        return self

    async def __anext__(self):
        """
        asynchronous generator .
        implementing the interface specified in the asynchronous generator protocol

        You can use asynchronous syntax like this:
            async for message in Subscriber():
                do_something(message)
                ...

        When the queue is empty, the asynchronous generator will automatically block instead of throwing an exception.
        """
        return await self._channel.get()

    async def put(self, message: any):
        """
        ! Method used by Broker only.
        Forward messages from the Input Channel to the Output Channel.
        """

        await self._channel.put(message)
```

在我们实现的异步迭代接口中，由于使用的 Channel 介质是我们在 I/O Channel 中实现的 AsyncIOChannel ，在从 Subscriber 对象中进行异步迭代时，若 Output Channel 为空，则当前协程上下文将在 AsyncIO 的事件循环中被挂起。在逻辑上可以看作被阻塞。

看到这，似乎发现，Output Channel 似乎并不是仅一条信道，它是一个订阅源的 Input Channel 在 Subscriber 实例上的映射。对于订阅了相同订阅源的 Subscriber，它们都各自拥有一个内容完全相同的 Out Channel 。

这就是发布-订阅者队列区别于生产-消费者对列的区别，对于发布-订阅者队列，要保证上游产生的消息会被下游所有实例响应。而生产-消费者队列仅需保证消息被响应即可。

#### MessageBroker

MessageBroker 的实现是整个系统的核心部件，
