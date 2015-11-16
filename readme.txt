Starting! See ....
^_^
--
http://www.cnblogs.com/zhiranok/archive/2012/07/30/fflib_framework.html
C++多进程并发框架FFLIB
         三年来一直从事服务器程序开发，一直都是忙忙碌碌，不久前结束了职业生涯的第一份工作，有了一个礼拜的休息时间，终于可以写写总结了。于是把以前的开源代码做了整理和优化，这就是FFLIB。虽然这边总结看起来像日记，有很多废话，但是此文仍然是有很大针对性的。针对服务器开发中常见的问题，如多线程并发、消息转发、异步、性能优化、单元测试，提出自己的见解。

面对的问题
         从事开发工程中，遇到过不少问题，很多时候由于时间紧迫，没有使用优雅的方案。在跟业内的一些朋友交流过程中，我也意识到有些问题是大家都存在的。简单列举如下：

多线程与并发
异步消息/接口调用
消息的序列化与Reflection
性能优化
单元测试
多线程与并发
         现在是多核时代，并发才能实现更高的吞吐量、更快的响应，但也是把双刃剑。总结如下几个用法：

多线程+显示锁；接口是被多线程调用的，当被调用时，显示加锁，再操作实体数据。悲剧的是，工程师为了优化会设计多个锁，以减少锁的粒度，甚至有些地方使用了原子操作。这些都为领域逻辑增加了额外的设计负担。最坏的情况是会出现死锁。
多线程+任务队列；接口被多线程调用，但请求会被暂存到任务队列，而任务队列会被单线程不断执行，典型生产者消费者模式。它的并发在于不同的接口可以使用不同的任务队列。这也是我最常用的并发方式。
　　这是两种最常见的多线程并发，它们有个天生的缺陷——Scalability。一个机器的性能总是有瓶颈的。两个场景的逻辑虽然由多个线程实现了并发，但是运算量十分有可能是一台机器无法承载的。如果是多进程并发，那么可以分布式把其部署到其他机器（也可部署在一台机器）。所以多进程并发比多线程并发更加Scalability。另外采用多进程后，每个进程单线程设计，这样的程序更加Simplicity。多进程的其他优点如解耦、模块化、方便调试、方便重用等就不赘言了。

异步消息/接口调用
         提到分布式，就要说一下分布式的通讯技术。常用的方式如下：

类RPC；包括WebService、RPC、ICE等，特点是远程同步调用。远程的接口和本地的接口非常相似。但是游戏服务器程序一般非常在意延迟和吞吐量，所以这些阻塞线程的同步远程调用方式并不常用。但是我们必须意识到他的优点，就是非常利于调用和测试。
全异步消息；当调用远程接口的时候，异步发送请求消息，接口响应后返回一个结果消息，调用方的回调函数处理结果消息继续逻辑操作。所以有些逻辑就会被切割成ServiceStart和ServiceCallback两段。有时异步会讲领域逻辑变得支离破碎。另外消息处理函数中一般会写一坨的switch/case 处理不同的消息。最大的问题在于单元测试，这种情况传统单元测试根本束手无策。
消息的序列化与Reflection
         实现消息的序列化和反序列化的方式有很多，常见的有Struct、json、Protobuff等都有很成功的应用。我个人倾向于使用轻量级的二进制序列化，优点是比较透明和高效，一切在掌握之中。在FFLIB 中实现了bin_encoder_t 和 bin_decoder_t 轻量级的消息序列化，几十行代码而已。

性能优化
         已经写过关于性能方面的总结，参见

         http://www.cnblogs.com/zhiranok/archive/2012/06/06/cpp_perf.html

　　   有的网友提到profiler、cpuprofiler、callgrind等工具。这些工具我都使用过，说实话，对于我来说，我太认同它有很高的价值。第一他们只能用于开发测试阶段，可以初步得到一些性能上参考数据。第二它们如何实现跟踪人们无从得知。运行其会使程序变慢，不能反映真实数据。第三重要的是，开发测试阶段性能和上线后的能一样吗？Impossible ！

　　   关于性能，原则就是数据说话，详见博文，不在赘述。

单元测试
         关于单元测试，前边已经谈论了一些。游戏服务器程序一般都比较庞大，但是不可思议的是，鄙人从来没见有项目（c++ 后台架构的）有完整单元测试的。由于存在着异步和多线程，传统的单元测试框架无法胜任，而开发支持异步的测试框架又是不现实的。我们必须看到的是，传统的单元测试框架已经取得了非常大的成功。据我了解，使用web 架构的游戏后台已经对于单元测试的使用已经非常成熟，取得了极其好的效果。所以我的思路是利用现有的单元测试框架，将异步消息、多线程的架构做出调整。

         已经多次谈论单元测试了。其实在开发FFLIB的思路很大程度来源于此，否则可能只是一个c++ 网络库而已。我决定尝试去解决这个问题的时候，把FFLIB 定位于框架。

         先来看一段非常简单的单元测试的代码 ：

         Assert(2 == Add(1, 1));

         请允许我对这行代码做些解释，对Add函数输入参数，验证返回值是否是预期的结果。这不就是单元测试的本质吗？在想一下我们异步发送消息的过程，如果每个输入消息约定一个结果消息包，每次发送请求时都绑定一个回调函数接收和验证结果消息包。这样的话就恰恰满足了传统单元测试的步骤了。最后还需解决一个问题，Assert是不能处理异步的返回值的。幸运的是，future机制可以化异步为同步。不了解future 模式的可以参考这里：

         http://blog.chinaunix.net/uid-23093301-id-190969.html

         http://msdn.microsoft.com/zh-cn/library/dd764564.aspx#Y300

 

         来看一下在FFLIB框架下远程调用echo 服务的示例：

复制代码
struct lambda_t
{
　　static void callback(echo_t::out_t& msg_)
　　{
　　　　echo_t::in_t in;
　　　　in.value = "XXX_echo_test_XXX";
　　　　singleton_t<msg_bus_t>::instance()
       .get_service_group("echo")
       ->get_service(1)->async_call(in, &lambda_t::callback);
　　}
};
echo_t::in_t in;
in.value = "XXX_echo_test_XXX";
singleton_t<msg_bus_t>::instance().get_service_group("echo")->get_service(1)->async_call(in, &lambda_t::callback);
复制代码
 

　　当需要调用远程接口时，async_call(in, &lambda_t::callback); 异步调用必须绑定一个回调函数，回调函数接收结果消息，可以触发后续操作。这样的话，如果对echo 的远程接口做单元测试，可以这样做：

复制代码
rpc_future_t< echo_t::out_t> rpc_future;
echo_t::in_t in;
in.value = "XXX_echo_test_XXX";
const echo_t::out_t& out = rpc_future.call(
    singleton_t<msg_bus_t>::instance()
        .get_service_group("echo")->get_service(1), in);
Assert(in.value == out.value);
复制代码
这样所有的远程接口都可以被单元测试覆盖。

FFLIB 介绍
 FFLIB 结构图



         如图所示，Client 不会直接和Service 相连接，而是通过Broker 中间层完成了消息传递。关于Broker 模式可以参见：http://blog.chinaunix.net/uid-23093301-id-90459.html

　　　进程间通信采用TPC，而不是多线程使用的共享内存方式。Service 一般是单线程架构的，通过启动多进程实现相对于多线程的并发。由于Broker模式天生石分布式的，所以有很好的Scalability。

消息时序图


如何注册服务和接口
　　来看一下Echo 服务的实现：

复制代码
struct echo_service_t
{
public:
    void echo(echo_t::in_t& in_msg_, rpc_callcack_t<echo_t::out_t>& cb_)
    {
        logtrace((FF, "echo_service_t::echo done value<%s>", in_msg_.value.c_str()));
        echo_t::out_t out;
        out.value = in_msg_.value;
        cb_(out);
    }
};
 

int main(int argc, char* argv[])
{
    int g_index = 1;
    if (argc > 1)
    {
        g_index = atoi(argv[1]);
    }
    char buff[128];
    snprintf(buff, sizeof(buff), "tcp://%s:%s", "127.0.0.1", "10241");
    
    msg_bus_t msg_bus;
    assert(0 == singleton_t<msg_bus_t>::instance().open("tcp://127.0.0.1:10241") && "can't connnect to broker");
 
    echo_service_t f;
 
    singleton_t<msg_bus_t>::instance().create_service_group("echo");
    singleton_t<msg_bus_t>::instance().create_service("echo", g_index)
            .bind_service(&f)
            .reg(&echo_service_t::echo);
    
    signal_helper_t::wait();
 
    singleton_t<msg_bus_t>::instance().close();
    //usleep(1000);
    cout <<"\noh end\n";
    return 0;
}
复制代码
 

create_service_group 创建一个服务group，一个服务组可能有多个并行的实例
create_service 以特定的id 创建一个服务实例
reg 为该服务注册接口
接口的定义规范为void echo(echo_t::in_t& in_msg_, rpc_callcack_t<echo_t::out_t>& cb_)，第一个参数为输入的消息struct，第二个参数为回调函数的模板特例，模板参数为返回消息的struct 类型。接口无需知道发送消息等细节，只需将结果callback 即可。
注册到Broker 后，所有Client都可获取该服务
消息定义的规范
    我们约定每个接口（远程或本地都应满足）都包含一个输入消息和一个结果消息。来看一下echo 服务的消息定义：

复制代码
struct echo_t
{
    struct in_t: public msg_i
    {
        in_t():
            msg_i("echo_t::in_t")
        {}
        virtual string encode()
        {
            return (init_encoder() << value).get_buff();
        }
        virtual void decode(const string& src_buff_)
        {
            init_decoder(src_buff_) >> value;
        }
        
        string value;
    };
    struct out_t: public msg_i
    {
        out_t():
            msg_i("echo_t::out_t")
        {}
        virtual string encode()
        {
            return (init_encoder() << value).get_buff();
        }
        virtual void decode(const string& src_buff_)
        {
            init_decoder(src_buff_) >> value;
        }
        
        string value;
    };
};
复制代码
 每个接口必须包含in_t消息和out_t消息，并且他们定义在接口名（如echo _t）的内部
所有消息都继承于msg_i, 其封装了二进制的序列化、反序列化等。构造时赋予类型名作为消息的名称。
每个消息必须实现encode 和 decode 函数
　　这里需要指出的是，FFLIB 中不需要为每个消息定义对应的CMD。当接口如echo向Broker 注册时，reg接口通过C++ 模板的类型推断会自动将该msg name 注册给Broker， Broker为每个msg name 分配唯一的msg_id。Msg_bus 中自动维护了msg_name 和msg_id 的映射。Msg_i 的定义如下：

复制代码
struct msg_i : public codec_i
{
    msg_i(const char* msg_name_):
        cmd(0),
        uuid(0),
        service_group_id(0),
        service_id(0),
        msg_id(0),
        msg_name(msg_name_)
    {}
    
    void set(uint16_t group_id, uint16_t id_, uint32_t uuid_, uint16_t msg_id_)
    {
        service_group_id = group_id;
        service_id       = id_;
        uuid             = uuid_;
        msg_id           = msg_id_;
    }
 
    uint16_t cmd;
    uint16_t get_group_id()   const{ return service_group_id; }
    uint16_t get_service_id() const{ return service_id;       }
    uint32_t get_uuid()       const{ return uuid;             }
    
    uint16_t get_msg_id()     const{ return msg_id;           }
    const string& get_name()  const
    {
        if (msg_name.empty() == false)
        {
            return msg_name;
        }
        return singleton_t<msg_name_store_t>::instance().id_to_name(this->get_msg_id());
    }
    
    void     set_uuid(uint32_t id_)   { uuid = id_;  }
    void     set_msg_id(uint16_t id_) { msg_id = id_;}
    void     set_sgid(uint16_t sgid_) { service_group_id = sgid_;}
    void     set_sid(uint16_t sid_)   { service_id = sid_; }
    uint32_t uuid;
    uint16_t service_group_id;
    uint16_t service_id;
    uint16_t msg_id;
    string   msg_name;
 
    virtual string encode(uint16_t cmd_)
    {
        this->cmd = cmd_;
        return encode();
    }
    virtual string encode() = 0;
    bin_encoder_t& init_encoder()
    {
        return encoder.init(cmd)  << uuid << service_group_id << service_id<< msg_id;
    }
    bin_encoder_t& init_encoder(uint16_t cmd_)
    {
        return encoder.init(cmd_) << uuid << service_group_id << service_id << msg_id;
    }
    bin_decoder_t& init_decoder(const string& buff_)
    {
        return decoder.init(buff_) >> uuid >> service_group_id >> service_id >> msg_id;
    }
    bin_decoder_t decoder;
    bin_encoder_t encoder;
};
复制代码
 

关于性能
         由于远程接口的调用必须通过Broker， Broker会为每个接口自动生成性能统计数据，并每10分钟输出到perf.txt 文件中。文件格式为CSV，参见:

http://www.cnblogs.com/zhiranok/archive/2012/06/06/cpp_perf.html

总结
FFLIB框架拥有如下的特点：

使用多进程并发。Broker 把Client 和Service 的位置透明化
Service 的接口要注册到Broker， 所有连接Broker的Client 都可以调用（publisher/ subscriber）
远程调用必须绑定回调函数
利用future 模式实现同步，从而支持单元测试
消息定义规范简单直接高效
所有service的接口性能监控数据自动生成，免费的午餐
Service 单线程话，更simplicity
源代码：

Svn co http://ffown.googlecode.com/svn/trunk/

运行示例：

Cd example/broker && make && ./app_broker –l http://127.0.0.1:10241
Cd example/echo_server && make && ./app_echo_server
Cd example/echo_client && make && ./app_echo_client
