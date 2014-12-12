#### MCollective架构篇1-MCollective架构的引入



Marionette Collective（MCollective）是一个与Puppet关系密切的服务运行框架。Puppet擅长管理系统的状态，但agent默认的30分钟间隔的运行方式使它不合适作为实时管理控制工具使用，而MCollective的功能定位正式面向大规模主机群的实时任务并行处理。它离线消息中间件技术实现检点间的信息传递，大量主机可以基于自身的某些固有属性（元数据）而非主机名进行分组，这意味着用这些信息按照不同标准将集群分为多个群组，任务执行的目标是一个群组，而不是一台主机。
也可以参考纸飞机的博客关于mcollective的介绍 [http://junqili.com/](http://junqili.com/)

<!--more-->

## MCollective特点： ##

    能够与小到大型服务器集群交互

    使用广播范式(broadcast paradigm)来进行请求分发，所有服务器会同时收到请求，而只有与请求所附带的过滤器匹配的服务器才会去执行这些请求。没有中心数据库来进行同步，网络是唯一的真理

    打破了以往用主机名作为身份验证手段的复杂命名规则。使用每台机器自身提供的丰富的目标数据来定位它们。目标数据来自于：Puppet, Chef, Facter, Ohai 或者自身提供的插件

    使用命令行调用远程代理

    能够写自定义的设备报告

    大量的代理来管理包，服务和其他来自于社区的通用组件

    允许写 SimpleRPC 风格的代理、客户端和使用 Ruby 实现 Web UIs

    外部可插件化(pluggable)实现本地需求

    中间件系统已有丰富的身份验证和授权模型，利用这些作为控制的第一道防线。

    重用中间件来做集群、路由和网络隔离以实现安全和可扩展安装。

MCollective 就是一个框架，一个空壳。它除了 MCO 命令之外都可以被替换被自定义。

**备注：**更多信息请参考http://docs.puppetlabs.com/

## Middleware（RabbitMQ、ActiveMQ）介绍 ##

RabbitMQ是一个实现了高级消息排队协议（AMQP）的消息队列服务。RabbitMQ基于OTP(Open Telecom Platform，开发电信平台)进行构建，并使用Erlang语言和运行时环境来实现。 ActiveMQ 是Apache出品，最流行的，能力强劲的开源消息总线。ActiveMQ 是一个完全支持JMS1.1和J2EE 1.4规范的 JMS Provider实现

备注：MCollective是基于Apache ActiveMQ中间件来进行开发和测试的，然而其对java和XML格式的配置文件的依赖使我们将更多的注意力和兴趣转移到RabbitMQ中间件服务上。如果考虑到性能和扩展性，部署ActivemMQ是一个更好的选择。

## 工作原理图 ##

![mcollective触发更新图](http://kisspuppet.com/img/mcollective-1.png)

备注：更多详细信息请参考 http://docs.puppetlabs.com/mcollective/reference/basic/messageflow.html 

## 部署介绍 ##

- MCollective安装分client安装和server端安装，其次需要安装MQ，本实验选择RabbitMQ，安装好之后需要进行相应的设置，然后进行通信。
- 如何和puppet进行整合，需要通过puppet插件实现。
- 本实验采用的版本为
mcollective 2.2.4
rabbitmq  3.1.5

