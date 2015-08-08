## 远程过程调用（RPC）

**（Python客户端 —— 使用 pika 0.9.8）**

在第二篇教程中我们介绍了如何使用工作队列（work queue）在多个工作者（woker）中间分发耗时的任务。

可是如果我们需要将一个函数运行在远程计算机上并且等待从那儿获取结果时，该怎么办呢？这就是另外的故事了。这种模式通常被称为远程过程调用（Remote Procedure Call）或者 RPC。

这篇教程中，我们会使用 RabbitMQ 来构建一个 RPC 系统：包含一个客户端和一个 RPC 服务器。现在的情况是，我们没有一个值得被分发的足够耗时的任务，所以接下来，我们会创建一个模拟RPC服务来返回斐波那契数列。

## 客户端接口

为了展示 RPC 服务如何使用，我们创建了一个简单的客户端类。它会暴露出一个名为 “call” 的方法用来发送一个 RPC 请求，并且在收到回应前保持阻塞。
  
```
fibonacci_rpc = FibonacciRpcClient()
result = fibonacci_rpc.call(4)
print "fib(4) is %r" % (result,)  
```  

### 关于 RPC 的注意事项：

尽管 RPC 在计算领域是一个常用模式，但它也经常被诟病。当一个问题被抛出的时候，程序员往往意识不到这到底是由本地调用还是由较慢的 RPC 调用引起的。同样的困惑还来自于系统的不可预测性和给调试工作带来的不必要的复杂性。跟软件精简不同的是，滥用 RPC 会导致不可维护的[面条代码](http://zh.wikipedia.org/wiki/%E9%9D%A2%E6%9D%A1%E5%BC%8F%E4%BB%A3%E7%A0%81).

考虑到这一点，牢记以下建议：

确保能够明确的搞清楚哪个函数是本地调用的，哪个函数是远程调用的。给你的系统编写文档。保持各个组件间的依赖明确。处理错误案例。明了客户端改如何处理 RPC 服务器的宕机和长时间无响应情况。

当对避免使用 RPC 有疑问的时候。如果可以的话，你应该尽量使用异步管道来代替RPC 类的阻塞。结果被异步地推送到下一个计算场景。

## 回调队列

一般来说通过 RabbitMQ 来实现 RPC 是很容易的。一个客户端发送请求信息，服务器端将其应用到一个回复信息中。为了接收到回复信息，客户端需要在发送请求的时候同时发送一个回调队列（callback queue）的地址。我们试试看：
  
```
result = channel.queue_declare(exclusive=True)
callback_queue = result.method.queue

channel.basic_publish(exchange='',
                      routing_key='rpc_queue',
                      properties=pika.BasicProperties(
                            reply_to = callback_queue,
                            ),
                      body=request)

# ... and some code to read a response message from the callback_queue ...  
```  

### 消息属性

AMQP 协议给消息预定义了一系列的14个属性。大多数属性很少会用到，除了以下几个：

- delivery_mode（投递模式）：将消息标记为持久的（值为2）或暂存的（除了2之外的其他任何值）。第二篇教程里接触过这个属性，记得吧？
- content_type（内容类型）:用来描述编码的 mime-type。例如在实际使用中常常使用 application/json 来描述 JOSN 编码类型。
- reply_to（回复目标）：通常用来命名回调队列。
- correlation_id（关联标识）：用来将RPC的响应和请求关联起来。  

## 关联标识

上边介绍的方法中，我们建议给每一个 RPC 请求新建一个回调队列。这不是一个高效的做法，幸好这儿有一个更好的办法 —— 我们可以为每个客户端只建立一个独立的回调队列。

这就带来一个新问题，当此队列接收到一个响应的时候它无法辨别出这个响应是属于哪个请求的。correlation\_id 就是为了解决这个问题而来的。我们给每个请求设置一个独一无二的值。稍后，当我们从回调队列中接收到一个消息的时候，我们就可以查看这条属性从而将响应和请求匹配起来。如果我们接手到的消息的correlation\_id 是未知的，那就直接销毁掉它，因为它不属于我们的任何一条请求。

你也许会问，为什么我们接收到未知消息的时候不抛出一个错误，而是要将它忽略掉？这是为了解决服务器端有可能发生的竞争情况。尽管可能性不大，但RPC服务器还是有可能在已将应答发送给我们但还未将确认消息发送给请求的情况下死掉。如果这种情况发生，RPC 在重启后会重新处理请求。这就是为什么我们必须在客户端优雅的处理重复响应，同时RPC也需要尽可能保持幂等性。

## 总结

![](images/16.png)

我们的 RPC 如此工作:

- 当客户端启动的时候，它创建一个匿名独享的回调队列。
- 在 RPC 请求中，客户端发送带有两个属性的消息：一个是设置回调队列的 reply\_to 属性，另一个是设置唯一值的 correlation\_id 属性。
- 将请求发送到一个 rpc\_queue 队列中。
- RPC 工作者（又名：服务器）等待请求发送到这个队列中来。当请求出现的时候，它执行他的工作并且将带有执行结果的消息发送给 reply\_to 字段指定的队列。
- 客户端等待回调队列里的数据。当有消息出现的时候，它会检查 correlation\_id 属性。如果此属性的值与请求匹配，将它返回给应用。
整合到一起

rpc_server.py代码：
  
```
\#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))

channel = connection.channel()

channel.queue_declare(queue='rpc_queue')

def fib(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fib(n-1) + fib(n-2)

def on_request(ch, method, props, body):
    n = int(body)

    print " [.] fib(%s)"  % (n,)
    response = fib(n)

    ch.basic_publish(exchange='',
                     routing_key=props.reply_to,
                     properties=pika.BasicProperties(correlation_id = \
                                                     props.correlation_id),
                     body=str(response))
    ch.basic_ack(delivery_tag = method.delivery_tag)

channel.basic_qos(prefetch_count=1)
channel.basic_consume(on_request, queue='rpc_queue')

print " [x] Awaiting RPC requests"
channel.start_consuming()   
```  

服务器端代码相当简单：

- （4）像往常一样，我们建立连接，声明队列
- （11）我们声明我们的 fibonacci 函数，它假设只有合法的正整数当作输入。（别指望这个函数能处理很大的数值，函数递归你们都懂得...）
- （19）我们为 basic_consume 声明了一个回调函数，这是 RPC 服务器端的核心。它执行实际的操作并且作出响应。
- （32）或许我们希望能在服务器上多开几个线程。为了能将负载平均地分摊到多个服务器，我们需要将 prefetch\_count 设置好。
rpc\_client.py 代码:
  
```
\#!/usr/bin/env python
import pika
import uuid

class FibonacciRpcClient(object):
    def __init__(self):
        self.connection = pika.BlockingConnection(pika.ConnectionParameters(
                host='localhost'))

        self.channel = self.connection.channel()

        result = self.channel.queue_declare(exclusive=True)
        self.callback_queue = result.method.queue

        self.channel.basic_consume(self.on_response, no_ack=True,
                                   queue=self.callback_queue)

    def on_response(self, ch, method, props, body):
        if self.corr_id == props.correlation_id:
            self.response = body

    def call(self, n):
        self.response = None
        self.corr_id = str(uuid.uuid4())
        self.channel.basic_publish(exchange='',
                                   routing_key='rpc_queue',
                                   properties=pika.BasicProperties(
                                         reply_to = self.callback_queue,
                                         correlation_id = self.corr_id,
                                         ),
                                   body=str(n))
        while self.response is None:
            self.connection.process_data_events()
        return int(self.response)

fibonacci_rpc = FibonacciRpcClient()

print " [x] Requesting fib(30)"
response = fibonacci_rpc.call(30)
print " [.] Got %r" % (response,)  
```  

客户端代码稍微有点难懂：

- （7）建立连接、通道并且为回复（replies）声明独享的回调队列。
- （16）我们订阅这个回调队列，以便接收 RPC 的响应。
- （18）“on_response” 回调函数对每一个响应执行一个非常简单的操作，检查每一个响应消息的 correlation\_id 属性是否与我们期待的一致，如果一致，将响应结果赋给 self.response，然后跳出 consuming 循环。
- （23）接下来，我们定义我们的主要方法 call 方法。它执行真正的 RPC 请求。
- （24）在这个方法中，首先我们生成一个唯一的 correlation\_id 值并且保存起来，'on_response' 回调函数会用它来获取符合要求的响应。
- （25）接下来，我们将带有 reply\_to 和 correlation\_id 属性的消息发布出去。
- （32）现在我们可以坐下来，等待正确的响应到来。
- （33）最后，我们将响应返回给用户。  

我们的 RPC 服务已经准备就绪了，现在启动服务器端：
  
```
$ python rpc_server.py
 [x] Awaiting RPC requests  
```  

运行客户端，请求一个fibonacci队列。
  
```
$ python rpc_client.py
 [x] Requesting fib(30)  
```  

此处呈现的设计并不是实现 RPC 服务的唯一方式，但是他有一些重要的优势：

- 如果 RPC 服务器运行的过慢的时候，你可以通过运行另外一个服务器端轻松扩展它。试试在控制台中运行第二个 rpc_server.py 。
- 在客户端，RPC 请求只发送或接收一条消息。不需要像 queue_declare 这样的异步调用。所以RPC客户端的单个请求只需要一个网络往返。  

我们的代码依旧非常简单，而且没有试图去解决一些复杂（但是重要）的问题，如：

- 当没有服务器运行时，客户端如何作出反映。
- 客户端是否需要实现类似RPC超时的东西。
- 如果服务器发生故障，并且抛出异常，应该被转发到客户端吗？
- 在处理前，防止混入无效的信息（例如检查边界）  

如果你想做一些实验，你会发现 [rabbitmq-management plugin](http://www.rabbitmq.com/plugins.html) 在观测队列方面是很有用处的。

（完整的 [rpc\_client.py](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/rpc_client.py) 和 [rpc\_server.py](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/rpc_server.py) 代码)