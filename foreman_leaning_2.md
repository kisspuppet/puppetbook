#### foreman架构的引入2-安装前环境准备


Foreman官网提供了每个版本非常完善的安装步骤，无论是源码安装还是rpm包安装都变得非常方便。而且Foreman通过puppet模块对安装步骤进行了封装并提供了大量的安装参数可以传输，相当的方便。不过由于其体系过大，代理很多软件，安装的软件包超多，安装过程也并非那么简单。

<!--more-->

以下是需要考虑的问题及解决方法

**特别说明：**接下来的所有的推荐说明、操作和测试都是基于目前最稳定版本1.5.3进行的，而1.6和1.7版本不太稳定，仅做安装介绍。

关于Foreman1.5.3版本介绍及安装方法可参考官网 [http://theforeman.org/manuals/1.5/index.html#Releasenotesfor1.5.3](http://theforeman.org/manuals/1.5/index.html#Releasenotesfor1.5.3)

## 操作系统的选型 ##

Foreman官网yum仓库只提供了el6和f19的rpm（[http://yum.theforeman.org/](http://yum.theforeman.org/)）包，Debian的deb包（[http://deb.theforeman.org/](http://deb.theforeman.org/)），并未提供低版本或者其它系统的rpm包。还有源码包的下载方式：`git clone https://github.com/theforeman/foreman.git -b 1.5-stable`
所以，如果你考虑使用rpm包安装，请使用以下系统及版本：

RHEL6.*

CentOS6.*

Fedora19

如果你考虑使用deb包安装，请使用以下系统及版本

Debian Linux 7.0 (Wheezy)

Debian Linux 6.0 (Squeeze)

Ubuntu Linux 14.04 LTS (Trusty Tahr)

Ubuntu Linux 12.04 LTS (Precise Pangolin)

如果你并不打算使用以上系统，比如现在很多金融行业使用的SLES系统等，需要考虑使用源码包安装，源码包安装通过bundle命令完成，不过很难安装，而且即使安装好，接下来走的路还很艰辛。

##安装包准备##

安装Foreman依赖的包比较多，需要从以下三个网站获取

**1、Foreman官网：**[ http://yum.theforeman.org/]( http://yum.theforeman.org/)

**2、EPEL官网：** [http://fedoraproject.org/wiki/EPEL](http://fedoraproject.org/wiki/EPEL)

**3、PuppetLabs官网：** [http://yum.puppetlabs.com/](http://yum.puppetlabs.com/)

**4、RabbitMQ官网：**[http://www.rabbitmq.com/download.html](http://www.rabbitmq.com/download.html)

**思考：**以上四个官网安装包那么多，如果能够获得到安装Foreman的包呢？

如果你确实比较懒，可以去我的Github上下载 [https://github.com/kisspuppet/foreman-repo](https://github.com/kisspuppet/foreman-repo)


## 软件包的选型如下： ##


- **puppet-server     3.6.2**
- **puppet            3.6.2**
- **facter            2.0.2**
- **mcollective       2.2.4**
- **rabbitmq-server   3.2.4**
- **foreman           1.5.3**
- **foreman-proxy     1.5.4**


##操作系统配置注意事项##

**1、操作系统版本必须是RHEL6版本以上，建议使用6.4或6.5。**

**2、主机名必须符合完全合格的FQDN名称，其次必须小写**（大写名称在安装MySQL的时候会提示授权问题不能通过）
eg. foreman.kisspuppet.com

**3、安装之前，必须先安装puppet客户端，并且和puppetmaster进行签名认证。**

**4、系统时间和puppetmaster端保持一致，防火墙、selinux记得关闭。**

