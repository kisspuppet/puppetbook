Foreman架构的引入10-hostgroup如何转换为本地的fact


在Foreman上可以根据业务逻辑设置多个主机组（Host Groups），并且可以将不同的节点加入到不同的主机组，这样在每次操作“puppet run”的时候，只需要在搜索按钮里搜索对应的主机组即可找到里面包含的所有节点，如下图所示

![Foreman安装](http://kisspuppet.com/img/foreman10-1.png)

但是，foreman目前在`puppet run`上对mcollective的集成度很低，基本就是只能运行一条命令。那么如果要在shell终端上通过mco命令去对这些自定义的`Host Groups`进行操作应该如何做呢。答案是转换为facter。

<!--more-->

自定义facter有四种方式，如下：[http://kisspuppet.com/2014/03/30/puppet_learning_base10/](http://kisspuppet.com/2014/03/30/puppet_learning_base10/)

这里介绍第三种方式将Foreman上设置的主机组（Host Groups）转换为每个节点自己的facter

## 1、首先创建主机组 ##

![Foreman安装](http://kisspuppet.com/img/foreman10-2.png)


## 2、查看节点的主机组信息 ##

其实相当于自定义了一个外部变量，变量名叫hostgroup，值为节点加入的组名称

![Foreman安装](http://kisspuppet.com/img/foreman10-3.png)

![Foreman安装](http://kisspuppet.com/img/foreman10-4.png)

## 3、编写一个fact模块 ##

模块的功能就是将Foreman上的变量“hostgroup”落地到每个节点的/etc/facter/facts.d/${hostname}.txt文件中，内容为fact的标准格式。

	#模块结构
	[root@puppetmaster162 modules]# tree fact
	fact
	├── files
	├── manifests
	│   ├── config.pp
	│   ├── fact.pp
	│   ├── init.pp
	│   └── params.pp
	└── templates
	    └── hostgroup.erb
	
	3 directories, 5 files
	
	#模块主配置文件init.pp
	[root@puppetmaster162 modules]# cat fact/manifests/init.pp 
	class fact {
	  tag("puppet_env")
	  require fact::params
	  $hostgroup_erb = $fact::params::hostgroup_erb
	  include fact::config
	  include fact::facter
	}

	#创建目录以及文件
	[root@puppetmaster162 modules]# cat fact/manifests/config.pp 
	class fact::config{
	  file { '/etc/facter' :
	    ensure  => directory,
	    owner   => 'root',
	    group   => 'root',
	    mode    => '0644',
	  }
	  file { '/etc/facter/facts.d' :
	    ensure  => directory,
	    owner   => 'root',
	    group   => 'root',
	    mode    => '0644',
	    require => File['/etc/facter']
	  }
	  file{ "/etc/facter/facts.d/$hostname.txt":
	    owner   => "root",
	    group   => "root",
	    mode    => 0400,
	    content => template($fact::hostgroup_erb),
	    require => File['/etc/facter/facts.d'],
	  }
	}

	#定义变量
	[root@puppetmaster162 modules]# cat fact/manifests/params.pp 
	class fact::params{
	  $hostgroup_erb = 'fact/hostgroup.erb'
	}

	#定义fact模板（原因可参考http://kisspuppet.com/2013/11/10/mcollective-middleware/）
	[root@puppetmaster162 manifests]# cat fact.pp 
	class fact::facter{
	file{"/etc/mcollective/facts.yaml":
	    owner    => root,
	    group    => root,
	    mode     => 0440,
	    loglevel => debug, # reduce noise in Puppet reports
	    content  => inline_template('<%= scope.to_hash.reject { |k,v| k.to_s =~ /(uptime.*|path|timestamp|free|.*password.*|.*psk.*|.*key)/ }.to_yaml %>'),
	  }
	}

	#设置文件模板
	[root@puppetmaster162 modules]# cat fact/templates/hostgroup.erb 
	hostgroup=<%= @hostgroup %> 
	foreman_env=<%= @foreman_env %>


## 4、Foreman上管理主机组和模块fact ##

先导入类，然后在主机组里进行关联即可，由于fact模块是针对所有主机的，建议关联到1级主机组，加入的节点会自动继承。关联完成后的效果如下

![Foreman安装](http://kisspuppet.com/img/foreman10-5.png)

![Foreman安装](http://kisspuppet.com/img/foreman10-6.png)


## 5、在Foreman上对两个节点执行“puppet run”操作 ##

![Foreman安装](http://kisspuppet.com/img/foreman10-7.png)


## 6、查看facter信息是否生成 ##

	[root@foreman163 ~]# facter hostgroup
	prd 
	
	[root@puppetmaster162 ~]# facter hostgroup
	prd/kisspuppet

## 7、通过mco命令结合fact进行过滤查看 ##

	[root@puppetmaster162 ~]# mco ping  -F hostgroup=prd
	foreman163.kisspuppet.com                time=98.55 ms
	
	
	---- ping statistics ----
	1 replies max: 98.55 min: 98.55 avg: 98.55 
	[root@puppetmaster162 ~]# mco ping  -F hostgroup=prd/kisspuppet
	puppetmaster162.kisspuppet.com           time=94.14 ms
	
	
	---- ping statistics ----
	1 replies max: 94.14 min: 94.14 avg: 94.14 
	[root@puppetmaster162 ~]# mco puppet -v runonce -F hostgroup=prd/kisspuppet
	Discovering hosts using the mc method for 2 second(s) .... 1
	
	 * [ ============================================================> ] 1 / 1
	
	
	puppetmaster162.kisspuppet.com          : OK
	    {:summary=>      "Started a Puppet run using the 'puppet agent --test --color=false --splay --splaylimit 30' command"}
	
	
	
	---- rpc stats ----
	           Nodes: 1 / 1
	     Pass / Fail: 1 / 0
	      Start Time: Thu Dec 18 15:13:09 +0800 2014
	  Discovery Time: 2004.07ms
	      Agent Time: 85.19ms
	      Total Time: 2089.26ms

**注：**以上方式只是提供了一种思路，更多的方式还需要根据具体的实际环境而改变，总之一点，fact很强大，看你怎么用。