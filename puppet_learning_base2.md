#### Puppet基础篇2-如何学习和使用Puppet



# 既来之，则安之。 #
对于Puppet的入门学习，其实并不难，很多人都会说Puppet是基于Ruby开发的，是不是必须要懂Ruby才能学好呢？其实并不是这样，不懂Ruby照样能学好Puppet。为什么这么说呢？
<!--more-->

1、Puppet虽然是基于Ruby开发的，但是Puppet所有的资源基本上都是基于Puppet自身语言而写的，而Puppet语言相对其他语言来说还是比较简单的，大部分都是A=>B这种格式，稍微复杂点，就加点判断语句，不过你会写一两个套用就可以了。

2、Puppet安装也是比较简单的，官方配备了详细的yum源，依赖包也很全，可以访问[http://yum.puppetlabs.com/](http://yum.puppetlabs.com/)下载系统对应的yum包puppetlabs-release-el，通过yum命令安装即可获得对应的repo。由于官方源默认安装的是最新版本的puppet版本，而最新版本由于其不稳定性并不是我们想要的，那么如何指定版本进行安装呢，请看这里[http://kisspuppet.com/2014/01/26/puppet_create_repo/](http://kisspuppet.com/2014/01/26/puppet_create_repo/)，如果你比较懒，不想自己做yum源，那就去下载KissPuppet准备的yum源吧，更全，更强大[http://kisspuppet.com/2013/12/05/puppet_repo_pak/](http://http://kisspuppet.com/2013/12/05/puppet_repo_pak/)

**这里给点建议：**官方yum源已经做的很到位了，如果条件允许，尽量通过rpm包安装而不是源码安装，至于原因自己想去。

3、通过puppet管理资源是需要写模块呢，有些人并不喜欢写模块，可以去[http://forge.puppetlabs.com/](http://forge.puppetlabs.com/)下载你需要的模块，如何下载安装呢，请点击这里[http://kisspuppet.com/2014/01/14/puppet_forge_modules/](http://kisspuppet.com/2014/01/14/puppet_forge_modules/)

4、日常学习当中，如何去查找puppet相关资料或者询问呢，这里教你9种方法去获取[http://kisspuppet.com/2014/02/10/puppet_irc/](http://kisspuppet.com/2014/02/10/puppet_irc/)

5、学习当中可别忘了看书哦，以下书籍是值得学习的

- 《pro puppet》第一版和第二版，中文版叫《精通puppet配置管理工具》，不过只有第一版，第二版只有英文版，相信不久的将来第二版也会被翻译成中文版，英语不错的童鞋可直接看英文版。
- 《Puppet 2.7 Cookbook RAW》第一版和第二版，第一版已经被翻译成中文，第二版基于3.x编写的，听说马上翻译完成了哦。
- 《puppet实战》去年年底新书，刚发布，由中国人刘宇编写，内容还是比较详细的，可系统学习。
- 《Managing Infrastructure with Puppet》，没怎么看过，呵呵！

**注：**以上书籍除了《puppet实战》外，QQ群里都有共享的pdf，可去下载

6、除此之外，KissPuppet还收集了一些有关puppet技术的网址，可直接点击查看，无需查找，节省时间，需要的点击这里[http://kisspuppet.com/2013/11/09/puppet-resource/](http://kisspuppet.com/2013/11/09/puppet-resource/)


说了这么多，真的就不需要去学ruby了么？其实并不是这样，谁都知道如果想要学到一款软件的精髓，还是要看其源代码的，至少有以下几个地方可能需要你懂点ruby

1、puppet模块中的erb模块部分，需要了解一些简单的ruby语句。

2、结合mcollecitve的plugins部分，如果需要修改或者添加新的plugin，需要懂一些ruby知识。

3、代码调试需要懂ruby

4、新的资源开发需要懂ruby

...

接下来我会带着大家一步一步搭建属于自己的Puppet架构，特别适合零基础学习的人。


