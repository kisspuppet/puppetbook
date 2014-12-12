#### MCollective架构篇4-MCollective各种插件的部署及测试


MCollective只是一个框架，如果需要在上面发挥各种作用，那就需要各种插件的支持。官方提供了很多这方面的插件，除此之外，还有第三方的插件，比如shell插件等，下面会介绍各种插件的安装，以及插件之间如何组合进行使用。

<!--more-->

## 1、在mcollective client端和server端安装各种官网plugins ##
首先去官网下载各个插件 [http://yum.puppetlabs.com](http://yum.puppetlabs.com)

**1.1 下载collective-client端**

	[root@linuxmaster1poc ~]# rpm -qa | grep mco
	mcollective-service-common-3.1.2-1.noarch
	mcollective-client-2.2.4-1.el6.noarch
	mcollective-service-client-3.1.2-1.noarch
	mcollective-common-2.2.4-1.el6.noarch
	mcollective-iptables-common-3.0.1-1.noarch
	mcollective-filemgr-client-1.0.1-1.noarch
	mcollective-nrpe-client-3.0.2-1.noarch
	mcollective-puppet-client-1.6.0-1.noarch
	mcollective-nrpe-common-3.0.2-1.noarch
	mcollective-filemgr-common-1.0.1-1.noarch
	mcollective-iptables-client-3.0.1-1.noarch
	mcollective-puppet-common-1.6.0-1.noarch
	mcollective-facter-facts-1.0.0-1.noarch
	mcollective-package-client-4.2.0-1.noarch
	mcollective-package-common-4.2.0-1.noarch


**1.2 下载mcollecitve-server端**

	[root@linux57poc ~]# rpm -qa | grep mco
	mcollective-nrpe-common-3.0.2-1
	mcollective-puppet-common-1.6.0-1
	mcollective-iptables-common-3.0.1-1
	mcollective-iptables-agent-3.0.1-1
	mcollective-2.2.4-1.el5
	mcollective-package-common-4.2.0-1
	mcollective-service-common-3.1.2-1
	mcollective-service-agent-3.1.2-1
	mcollective-puppet-agent-1.6.0-1
	mcollective-package-agent-4.2.0-1
	mcollective-filemgr-common-1.0.1-1
	mcollective-common-2.2.4-1.el5
	mcollective-facter-facts-1.0.0-1
	mcollective-filemgr-agent-1.0.1-1
	mcollective-nrpe-agent-3.0.2-1

**以上安装可写个package模块执行，以下只针对mcollective server端，安装完成之后记得重启服务，如果写了service模块可以自动刷新**

**1.3 编写plugins.pp**

	class mcollective::plugins{
	  include mcollective::plugins_puppet,
        mcollective::plugins_facter,
        mcollective::plugins_filemgr,
        mcollective::plugins_iptables,
	#   mcollective::plugins_nettest, #这个安装需要依赖包 ruby-net-ping，没找到
        mcollective::plugins_nrpe,
        mcollective::plugins_package,
        mcollective::plugins_service
    }
	
	#mco-client need install mcollective-puppet-client and mcollective-puppet-common
	class mcollective::plugins_puppet{
	  package { ['mcollective-puppet-agent','mcollective-puppet-common']:
	    ensure  => installed,
        require => Class["mcollective::install"]
	  }
    }
	
	
	#mco-client need install mcollective-facter-facts
	class mcollective::plugins_facter{
	  package { 'mcollective-facter-facts':
	    ensure  => installed,
        require => Class["mcollective::install"]
	  }
    }
	
	#mco-client need install mcollective-filemgr-client and mcollective-filemgr-common
	class mcollective::plugins_filemgr{
	  package { ['mcollective-filemgr-agent','mcollective-filemgr-common']:
	    ensure  => installed,
        require => Class["mcollective::install"]
	  }
    }
	
	
	#mco-client need install mcollective-iptables-client and mcollective-iptables-common
	class mcollective::plugins_iptables{
	  package { ['mcollective-iptables-agent','mcollective-iptables-common']:
	    ensure  => installed,
        require => Class["mcollective::install"]
	  }
    }
	
	#mco-client need install mcollective-nettest-client and mcollective-nettest-common
	class mcollective::plugins_nettest{
	  package { ['mcollective-nettest-agent','mcollective-nettest-common']:
	    ensure  => installed,
        require => Class["mcollective::install"]
	  }
    }
	
	#mco-client need install mcollective-nrpe-client and mcollective-nrpe-common
	class mcollective::plugins_nrpe{
	  package { ['mcollective-nrpe-agent','mcollective-nrpe-common']:
	    ensure  => installed,
        require => Class["mcollective::install"]
	  }
    }
	
	#mco-client need install mcollective-package-client and mcollective-package-common
	class mcollective::plugins_package{
	  package { ['mcollective-package-agent','mcollective-package-common']:
	    ensure  => installed,
        require => Class["mcollective::install"]
	  }
    }
	
	#mco-client need install mcollective-service-client and mcollective-service-common
	class mcollective::plugins_service{
	  package { ['mcollective-service-agent','mcollective-service-common']:
	    ensure  => installed,
        require => Class["mcollective::install"]
	  }
    }

**1.4 编写conf.pp**

	class mcollective::service{
	  service { 'mcollective':
	    ensure     => running,
	    hasstatus  => true,
	    hasrestart => true,
	    enable     => true,
	    subscribe  => Class['mcollective::config'],
	  }
    }

**1.5 mcollective-client端安装好之后，可通过mco命令查看**

	[root@linuxmaster1poc ~]# mco 
	The Marionette Collective version 2.2.4
	
	usage: /usr/bin/mco command <options>
	
	Known commands:
	
	   completion           facts                filemgr             
	   find                 help                 inventory           
	   iptables             nrpe                 package             
	   ping                 plugin               puppet              
	   rpc                  service              shell               
	
	Type '/usr/bin/mco help' for a detailed list of commands and '/usr/bin/mco help command'
	to get detailed help for a command


**1.6 mcollective-server端安装好之后，可在mco-client端查看**

	[root@linuxmaster1poc ~]# mco inventory linux57poc
	Inventory for linux57poc:
	
	   Server Statistics:
	                      Version: 2.2.4
	                   Start Time: Fri Dec 13 08:15:46 +0800 2013
	                  Config File: /etc/mcollective/server.cfg
	                  Collectives: mcollective
	              Main Collective: mcollective
	                   Process ID: 23268
	               Total Messages: 16
	      Messages Passed Filters: 16
	            Messages Filtered: 0
	             Expired Messages: 0
	                 Replies Sent: 15
	         Total Processor Time: 0.71 seconds
	                  System Time: 0.15 seconds
	
	   Agents: #都加载上了
	      discovery       filemgr         nrpe           
	      package         puppet          rpcutil        
	      service         shell                          
	
	   Data Plugins:
	      agent           fstat           nrpe           
	      puppet          resource        service        
	
	   Configuration Management Classes:
	      No classes applied
	
	   Facts:
	      architecture => x86_64
	      augeasversion => 0.10.0
	      bios_release_date => 06/22/2012
	      bios_vendor => Phoenix Technologies LTD
	      bios_version => 6.00
	      blockdevice_fd0_size => 4096
	      blockdevice_hdc_size => 3834736640
		。。。


**注意：** 接下来测试各种命令的操作组合，这里只举一些例子，更多信息可参考--help或者参考官网

## 2、安装shell插件 ##

插件下载地址：https://github.com/kisspuppet/mcollective-plugins，有github客户端的童鞋可直接clone https://github.com/kisspuppet/mcollective-plugins.git

**2.1、下载插件放在对应的目录里即可**

	mcollective-client端
	
	[root@linuxmaster1poc ~]# ll /usr/libexec/mcollective/mcollective/application/ | grep shell
	-rw-r--r-- 1 root root  1601 Aug  6 06:36 shell.rb
	[root@linuxmaster1poc ~]# ll /usr/libexec/mcollective/mcollective/agent/ | grep shell
	-rw-r--r-- 1 root root 1017 Aug  6 06:36 shell.ddl
	-rw-r--r-- 1 root root  862 Aug  6 06:36 shell.rb
	
	mcollective-server端
	
	[root@linux57poc agent]# ll /usr/libexec/mcollective/mcollective/agent/ | grep shell
	-rw-r--r-- 1 root root 1017 Aug  6 06:36 shell.ddl
	-rw-r--r-- 1 root root  862 Aug  6 06:36 shell.rb

备注：mcollective-server端部署完成之后，记得重启mcollective服务。

**2.2、 查看shell插件是否加载成功**

从下面可以看出mcollective-client端shell插件已经有了 

	[root@linuxmaster1poc ~]# mco The Marionette Collective version 2.2.4 usage: /usr/bin/mco command Known commands: completion facts find
	help inventory ping
	plugin puppet rpc
	shell #shell插件加载OK
	Type '/usr/bin/mco help' for a detailed list of commands and '/usr/bin/mco help command' to get detailed help for a command

从下面可以看出mcollective-server端shell插件也加载了

	[root@linuxmaster1poc ~]# mco inventory linux57poc 
	Inventory for linux57poc:
	   Server Statistics:
	                      Version: 2.2.4
	                   Start Time: Fri Dec 13 01:14:14 +0800 2013
	                  Config File: /etc/mcollective/server.cfg
	                  Collectives: mcollective
	              Main Collective: mcollective
	                   Process ID: 23898
	               Total Messages: 10
	      Messages Passed Filters: 10
	            Messages Filtered: 0
	             Expired Messages: 0
	                 Replies Sent: 9
	         Total Processor Time: 0.73 seconds
	                  System Time: 0.17 seconds
	   Agents:
	      discovery       puppet          rpcutil        
	      shell  #shell插件加载OK                                          
	   Data Plugins:
	      agent           fstat           puppet         
	      resource                                       
	   Configuration Management Classes:
	      No classes applied
	   Facts:
	      architecture => x86_64
	      augeasversion => 0.10.0
	      bios_release_date => 06/22/2012
	      bios_vendor => Phoenix Technologies LTD
	      bios_version => 6.00
	      blockdevice_fd0_size => 4096
	      blockdevice_hdc_size => 3834736640
	      blockdevice_sda_model => Virtual disk
	      blockdevice_sda_size => 42949672960
	    。。。

**2.3、通过shell插件执行shell命令**

	mco shell帮助信息
	
	[root@linuxmaster1poc ~]# mco shell --help
	MCollective Distributed Shell
	Usage:   mco shell <CMD>
	  The CMD is a string
	  EXAMPLES:
	    mco shell uptime
	        --np, --no-progress          Do not show the progress bar
	    -1, --one                        Send request to only one discovered nodes
	        --batch SIZE                 Do requests in batches
	        --batch-sleep SECONDS        Sleep time between batches
	        --limit-seed NUMBER          Seed value for deterministic random batching
	        --limit-nodes, --ln, --limit COUNT
	                                     Send request to only a subset of nodes, can be a percentage
	    -j, --json                       Produce JSON output
	        --display MODE               Influence how results are displayed. One of ok, all or failed
	    -c, --config FILE                Load configuratuion from file rather than default
	    -v, --verbose                    Be verbose
	    -h, --help                       Display this screen
	Common Options
	    -T, --target COLLECTIVE          Target messages to a specific sub collective
	        --dt, --discovery-timeout SECONDS
	                                     Timeout for doing discovery
	    -t, --timeout SECONDS            Timeout for calling remote agents
	    -q, --quiet                      Do not be verbose
	        --ttl TTL                    Set the message validity period
	        --reply-to TARGET            Set a custom target for replies
	        --dm, --disc-method METHOD   Which discovery method to use
	        --do, --disc-option OPTION   Options to pass to the discovery method
	        --nodes FILE                 List of nodes to address
	Host Filters
	    -W, --with FILTER                Combined classes and facts filter
	    -S, --select FILTER              Compound filter combining facts and classes
	    -F, --wf, --with-fact fact=val   Match hosts with a certain fact
	    -C, --wc, --with-class CLASS     Match hosts with a certain config management class
	    -A, --wa, --with-agent AGENT     Match hosts with a certain agent
	    -I, --wi, --with-identity IDENT  Match hosts with a certain configured identity
	The Marionette Collective 2.2.4

显示对端uptime命令负载情况

	[root@linuxmaster1poc ~]# mco shell "uptime" 
	Do you really want to send this command unfiltered? (y/n): y
	Discovering hosts using the mc method for 2 second(s) .... 3
	Host: linux58poc
	Statuscode: 0
	Output:
	 02:45:02 up 21:10,  2 users,  load average: 0.00, 0.00, 0.00
	Host: linux64poc
	Statuscode: 0
	Output:
	 02:45:02 up 20:59,  1 user,  load average: 0.00, 0.00, 0.00
	Host: linux57poc
	Statuscode: 0
	Output:
	 02:45:02 up 21:04,  3 users,  load average: 0.00, 0.00, 0.00

显示所有节点/etc/password文件中puppet用户哪一行

	[root@linuxmaster1poc ~]# mco shell "cat /etc/passwd | grep puppet" 
	Do you really want to send this command unfiltered? (y/n): y
	Discovering hosts using the mc method for 2 second(s) .... 3
	Host: linux58poc
	Statuscode: 0
	Output:
	puppet:x:52:52:Puppet:/var/lib/puppet:/sbin/nologin
	Host: linux64poc
	Statuscode: 0
	Output:
	puppet:x:52:52:Puppet:/var/lib/puppet:/sbin/nologin
	Host: linux57poc
	Statuscode: 0
	Output:
	puppet:x:52:52:Puppet:/var/lib/puppet:/sbin/nologin

修改其中一台主机的root密码

	[root@linuxmaster1poc ~]# mco shell "echo redhat | passwd root --stdin" -I linux57poc 
	Host: linux57poc
	Statuscode: 0
	Output:
	Changing password for user root.
	passwd: all authentication tokens updated successfully.

**备注：**更多操作步骤可参考mco shell --help帮助。

**警告：**基于mcollective的shell插件虽然功能很强大，除了动态显示的命令之外，其它root能操作的，它基本上都能操作。所以操作也非常危险，可根据生产环境实际情况而定。

**注意：** 接下来测试各种命令的操作组合，这里只举一些例子，更多信息可参考--help或者参考官网

## 3、组合mcollective各种plugins完成各种任务组合 ##

**3.1、停止操作系统为RHEL5.x服务器的crond任务**

先查看5.x系统crond的状态，使用插件 service、facts

	[root@linuxmaster1poc ~]# mco service crond status -F operatingsystemmajrelease=5
	
	 * [ ============================================================> ] 2 / 2
	
	   linux57poc: running
	   linux58poc: running
	
	Summary of Service Status:
	
	   running = 2
	
	
	Finished processing 2 / 2 hosts in 184.79 ms

然后通过service插件停止服务，使用插件 service、facts

	[root@linuxmaster1poc ~]# mco service crond stop -F operatingsystemmajrelease=5
	
	 * [ ============================================================> ] 2 / 2
	
	
	Summary of Service Status:
	
	   stopped = 2
	
	
	Finished processing 2 / 2 hosts in 914.76 ms

再次查看过滤的主机crond服务是否被停掉，使用插件 service、facts

	[root@linuxmaster1poc ~]# mco service crond status -F operatingsystemmajrelease=5
	
	 * [ ============================================================> ] 2 / 2
	
	   linux57poc: stopped
	   linux58poc: stopped
	
	Summary of Service Status:
	
	   stopped = 2
	
	
	Finished processing 2 / 2 hosts in 125.87 ms

也可以通过shell插件实现，使用到插件为shell、service、facts

	[root@linuxmaster1poc ~]# mco shell "service crond status" -F operatingsystemmajrelease=5
	Discovering hosts using the mc method for 2 second(s) .... 2
	Host: linux57poc
	Statuscode: 3
	Output:
	crond is stopped
	Host: linux58poc
	Statuscode: 3
	Output:
	crond is stopped

**3.2、使用mco对自定义fact_apply4=app的主机做一次变更，要求环境为testing，模式为noop**

首先查看下那些主机具备有这个自定义fact，使用的插件为find、inventory

	[root@linuxmaster1poc ~]# for i in `mco find` ; do echo $i; mco inventory $i | grep fact_apply4; done
	linux58poc
	  fact_apply4 => app
	linux57poc
	linux64poc
	  fact_apply4 => app

其次按要求做变更即可，使用到的插件为puppet，facts

	[root@linuxmaster1poc ~]# mco puppet -v runonce --environment=testing --noop -F fact_apply4=app
	Discovering hosts using the mc method for 2 second(s) .... 2
	
	 * [ ============================================================> ] 2 / 2
	
	
	linux64poc                              : OK
	    {:summary=>      "Started a background Puppet run using the 'puppet agent --onetime --daemonize --color=false --splay --splaylimit 30 --noop --environment testing' command"}
	
	linux58poc                              : OK
	    {:summary=>      "Started a background Puppet run using the 'puppet agent --onetime --daemonize --color=false --splay --splaylimit 30 --noop --environment testing' command"}
	
	
	
	---- rpc stats ----
	           Nodes: 2 / 2
	     Pass / Fail: 2 / 0
	      Start Time: Fri Dec 13 09:10:50 +0800 2013
	  Discovery Time: 2003.32ms
	      Agent Time: 884.34ms
	      Total Time: 2887.67ms

变更完成后，迅速查看节点运行情况，使用到的插件为puppet
	
	[root@linuxmaster1poc ~]# mco puppet status 
	
	 * [ ============================================================> ] 3 / 3
	
	   linux64poc: Currently idling; last completed run 54 seconds ago
	   linux58poc: Currently applying a catalog; last completed run 1 minutes 12 seconds ago
	   linux57poc: Currently stopped; last completed run 22 minutes 57 seconds ago
	
	Summary of Applying:
	
	   false = 2
	    true = 1
	
	Summary of Daemon Running:
	
	   running = 2
	   stopped = 1
	
	Summary of Enabled:
	
	   enabled = 3
	
	Summary of Idling:
	
	   false = 2
	    true = 1
	
	Summary of Status:
	
	               idling = 1
	              stopped = 1
	   applying a catalog = 1
	
	
	Finished processing 3 / 3 hosts in 263.72 ms

---

**3.3、远程改所有系统为RHEL6.4主机root的密码，使用到的插件为shell，facts**

	[root@linuxmaster1poc ~]# mco shell "echo redhat | passwd root --stdin" -F operatingsystemrelease=6.4
	Discovering hosts using the mc method for 2 second(s) .... 1
	Host: linux64poc
	Statuscode: 0
	Output:
	Changing password for user root.
	passwd: all authentication tokens updated successfully.


**3.4、查看所有节点puppet和facter安装包的版本信息，使用到的插件为package**

	[root@linuxmaster1poc ~]# mco package status puppet
	
	 * [ ============================================================> ] 3 / 3
	
	   linux64poc: puppet-2.7.23-1.el6.noarch
	   linux57poc: puppet-2.7.23-1.el5.noarch
	   linux58poc: puppet-2.7.23-1.el5.noarch
	
	Summary of Arch:
	
	   noarch = 3
	
	Summary of Ensure:
	
	   2.7.23-1.el5 = 2
	   2.7.23-1.el6 = 1
	
	
	Finished processing 3 / 3 hosts in 635.21 ms

	[root@linuxmaster1poc ~]# mco package status facter
	
	 * [ ============================================================> ] 3 / 3
	
	   linux58poc: facter-1.7.3-1.el5.x86_64
	   linux64poc: facter-1.7.3-1.el6.x86_64
	   linux57poc: facter-1.7.3-1.el5.x86_64
	
	Summary of Arch:
	
	   x86_64 = 3
	
	Summary of Ensure:
	
	   1.7.3-1.el5 = 2
	   1.7.3-1.el6 = 1
	
	
	Finished processing 3 / 3 hosts in 124.99 ms

更多的功能可通过以下方式查看：

	[root@linuxmaster1poc ~]# mco puppet -h
	
	Schedule runs, enable, disable and interrogate the Puppet Agent
	
	Usage: mco puppet [OPTIONS] [FILTERS] <ACTION> [CONCURRENCY|MESSAGE]
	Usage: mco puppet <count|enable|status|summary>
	Usage: mco puppet disable [message]
	Usage: mco puppet runonce [PUPPET OPTIONS]
	Usage: mco puppet resource type name property1=value property2=value
	Usage: mco puppet runall [--rerun SECONDS] [PUPPET OPTIONS]
	
	The ACTION can be one of the following:
	
	    count    - return a total count of running, enabled, and disabled nodes
	    enable   - enable the Puppet Agent if it was previously disabled
	    disable  - disable the Puppet Agent preventing catalog from being applied
	    resource - manage individual resources using the Puppet Type (RAL) system
	    runall   - invoke a puppet run on matching nodes, making sure to only run
	               CONCURRENCY nodes at a time
	    runonce  - invoke a Puppet run on matching nodes
	    status   - shows a short summary about each Puppet Agent status
	    summary  - shows resource and run time summaries
	
	        --force                      Bypass splay options when running
	        --server SERVER              Connect to a specific server or port
	        --tags, --tag TAG            Restrict the run to specific tags
	        --noop                       Do a noop run
	        --no-noop                    Do a run with noop disabled
	        --environment ENVIRONMENT    Place the node in a specific environment for this run
	        --splay                      Splay the run by up to splaylimit seconds
	        --no-splay                   Do a run with splay disabled
	        --splaylimit SECONDS         Maximum splay time for this run if splay is set
	        --ignoreschedules            Disable schedule processing
	        --rerun SECONDS              When performing runall do so repeatedly with a minimum run time of SECONDS
	
	        --np, --no-progress          Do not show the progress bar
	    -1, --one                        Send request to only one discovered nodes
	        --batch SIZE                 Do requests in batches
	        --batch-sleep SECONDS        Sleep time between batches
	        --limit-seed NUMBER          Seed value for deterministic random batching
	        --limit-nodes, --ln, --limit COUNT
	                                     Send request to only a subset of nodes, can be a percentage
	    -j, --json                       Produce JSON output
	        --display MODE               Influence how results are displayed. One of ok, all or failed
	    -c, --config FILE                Load configuratuion from file rather than default
	    -v, --verbose                    Be verbose
	    -h, --help                       Display this screen
	
	Common Options
	    -T, --target COLLECTIVE          Target messages to a specific sub collective
	        --dt, --discovery-timeout SECONDS
	                                     Timeout for doing discovery
	    -t, --timeout SECONDS            Timeout for calling remote agents
	    -q, --quiet                      Do not be verbose
	        --ttl TTL                    Set the message validity period
	        --reply-to TARGET            Set a custom target for replies
	        --dm, --disc-method METHOD   Which discovery method to use
	        --do, --disc-option OPTION   Options to pass to the discovery method
	        --nodes FILE                 List of nodes to address
	
	Host Filters
	    -W, --with FILTER                Combined classes and facts filter
	    -S, --select FILTER              Compound filter combining facts and classes
	    -F, --wf, --with-fact fact=val   Match hosts with a certain fact
	    -C, --wc, --with-class CLASS     Match hosts with a certain config management class
	    -A, --wa, --with-agent AGENT     Match hosts with a certain agent
	    -I, --wi, --with-identity IDENT  Match hosts with a certain configured identity
	
	The Marionette Collective 2.2.4


























## 2、组合mcollective各种plugins完成各种任务组合 ##

**2.1、停止操作系统为RHEL5.x服务器的crond任务**

先查看5.x系统crond的状态，使用插件 service、facts

	[root@linuxmaster1poc ~]# mco service crond status -F operatingsystemmajrelease=5
	
	 * [ ============================================================> ] 2 / 2
	
	   linux57poc: running
	   linux58poc: running
	
	Summary of Service Status:
	
	   running = 2
	
	
	Finished processing 2 / 2 hosts in 184.79 ms

然后通过service插件停止服务，使用插件 service、facts

	[root@linuxmaster1poc ~]# mco service crond stop -F operatingsystemmajrelease=5
	
	 * [ ============================================================> ] 2 / 2
	
	
	Summary of Service Status:
	
	   stopped = 2
	
	
	Finished processing 2 / 2 hosts in 914.76 ms

再次查看过滤的主机crond服务是否被停掉，使用插件 service、facts

	[root@linuxmaster1poc ~]# mco service crond status -F operatingsystemmajrelease=5
	
	 * [ ============================================================> ] 2 / 2
	
	   linux57poc: stopped
	   linux58poc: stopped
	
	Summary of Service Status:
	
	   stopped = 2
	
	
	Finished processing 2 / 2 hosts in 125.87 ms

也可以通过shell插件实现，使用到插件为shell、service、facts

	[root@linuxmaster1poc ~]# mco shell "service crond status" -F operatingsystemmajrelease=5
	Discovering hosts using the mc method for 2 second(s) .... 2
	Host: linux57poc
	Statuscode: 3
	Output:
	crond is stopped
	Host: linux58poc
	Statuscode: 3
	Output:
	crond is stopped

**2.2、使用mco对自定义fact_apply4=app的主机做一次变更，要求环境为testing，模式为noop**

首先查看下那些主机具备有这个自定义fact，使用的插件为find、inventory

	[root@linuxmaster1poc ~]# for i in `mco find` ; do echo $i; mco inventory $i | grep fact_apply4; done
	linux58poc
	  fact_apply4 => app
	linux57poc
	linux64poc
	  fact_apply4 => app

其次按要求做变更即可，使用到的插件为puppet，facts

	[root@linuxmaster1poc ~]# mco puppet -v runonce --environment=testing --noop -F fact_apply4=app
	Discovering hosts using the mc method for 2 second(s) .... 2
	
	 * [ ============================================================> ] 2 / 2
	
	
	linux64poc                              : OK
	    {:summary=>      "Started a background Puppet run using the 'puppet agent --onetime --daemonize --color=false --splay --splaylimit 30 --noop --environment testing' command"}
	
	linux58poc                              : OK
	    {:summary=>      "Started a background Puppet run using the 'puppet agent --onetime --daemonize --color=false --splay --splaylimit 30 --noop --environment testing' command"}
	
	
	
	---- rpc stats ----
	           Nodes: 2 / 2
	     Pass / Fail: 2 / 0
	      Start Time: Fri Dec 13 09:10:50 +0800 2013
	  Discovery Time: 2003.32ms
	      Agent Time: 884.34ms
	      Total Time: 2887.67ms

变更完成后，迅速查看节点运行情况，使用到的插件为puppet
	
	[root@linuxmaster1poc ~]# mco puppet status 
	
	 * [ ============================================================> ] 3 / 3
	
	   linux64poc: Currently idling; last completed run 54 seconds ago
	   linux58poc: Currently applying a catalog; last completed run 1 minutes 12 seconds ago
	   linux57poc: Currently stopped; last completed run 22 minutes 57 seconds ago
	
	Summary of Applying:
	
	   false = 2
	    true = 1
	
	Summary of Daemon Running:
	
	   running = 2
	   stopped = 1
	
	Summary of Enabled:
	
	   enabled = 3
	
	Summary of Idling:
	
	   false = 2
	    true = 1
	
	Summary of Status:
	
	               idling = 1
	              stopped = 1
	   applying a catalog = 1
	
	
	Finished processing 3 / 3 hosts in 263.72 ms

---

**3、远程改所有系统为RHEL6.4主机root的密码，使用到的插件为shell，facts**

	[root@linuxmaster1poc ~]# mco shell "echo redhat | passwd root --stdin" -F operatingsystemrelease=6.4
	Discovering hosts using the mc method for 2 second(s) .... 1
	Host: linux64poc
	Statuscode: 0
	Output:
	Changing password for user root.
	passwd: all authentication tokens updated successfully.


**4、查看所有节点puppet和facter安装包的版本信息，使用到的插件为package**

	[root@linuxmaster1poc ~]# mco package status puppet
	
	 * [ ============================================================> ] 3 / 3
	
	   linux64poc: puppet-2.7.23-1.el6.noarch
	   linux57poc: puppet-2.7.23-1.el5.noarch
	   linux58poc: puppet-2.7.23-1.el5.noarch
	
	Summary of Arch:
	
	   noarch = 3
	
	Summary of Ensure:
	
	   2.7.23-1.el5 = 2
	   2.7.23-1.el6 = 1
	
	
	Finished processing 3 / 3 hosts in 635.21 ms

	[root@linuxmaster1poc ~]# mco package status facter
	
	 * [ ============================================================> ] 3 / 3
	
	   linux58poc: facter-1.7.3-1.el5.x86_64
	   linux64poc: facter-1.7.3-1.el6.x86_64
	   linux57poc: facter-1.7.3-1.el5.x86_64
	
	Summary of Arch:
	
	   x86_64 = 3
	
	Summary of Ensure:
	
	   1.7.3-1.el5 = 2
	   1.7.3-1.el6 = 1
	
	
	Finished processing 3 / 3 hosts in 124.99 ms

更多的功能可通过以下方式查看：

	[root@linuxmaster1poc ~]# mco puppet -h
	
	Schedule runs, enable, disable and interrogate the Puppet Agent
	
	Usage: mco puppet [OPTIONS] [FILTERS] <ACTION> [CONCURRENCY|MESSAGE]
	Usage: mco puppet <count|enable|status|summary>
	Usage: mco puppet disable [message]
	Usage: mco puppet runonce [PUPPET OPTIONS]
	Usage: mco puppet resource type name property1=value property2=value
	Usage: mco puppet runall [--rerun SECONDS] [PUPPET OPTIONS]
	
	The ACTION can be one of the following:
	
	    count    - return a total count of running, enabled, and disabled nodes
	    enable   - enable the Puppet Agent if it was previously disabled
	    disable  - disable the Puppet Agent preventing catalog from being applied
	    resource - manage individual resources using the Puppet Type (RAL) system
	    runall   - invoke a puppet run on matching nodes, making sure to only run
	               CONCURRENCY nodes at a time
	    runonce  - invoke a Puppet run on matching nodes
	    status   - shows a short summary about each Puppet Agent status
	    summary  - shows resource and run time summaries
	
	        --force                      Bypass splay options when running
	        --server SERVER              Connect to a specific server or port
	        --tags, --tag TAG            Restrict the run to specific tags
	        --noop                       Do a noop run
	        --no-noop                    Do a run with noop disabled
	        --environment ENVIRONMENT    Place the node in a specific environment for this run
	        --splay                      Splay the run by up to splaylimit seconds
	        --no-splay                   Do a run with splay disabled
	        --splaylimit SECONDS         Maximum splay time for this run if splay is set
	        --ignoreschedules            Disable schedule processing
	        --rerun SECONDS              When performing runall do so repeatedly with a minimum run time of SECONDS
	
	        --np, --no-progress          Do not show the progress bar
	    -1, --one                        Send request to only one discovered nodes
	        --batch SIZE                 Do requests in batches
	        --batch-sleep SECONDS        Sleep time between batches
	        --limit-seed NUMBER          Seed value for deterministic random batching
	        --limit-nodes, --ln, --limit COUNT
	                                     Send request to only a subset of nodes, can be a percentage
	    -j, --json                       Produce JSON output
	        --display MODE               Influence how results are displayed. One of ok, all or failed
	    -c, --config FILE                Load configuratuion from file rather than default
	    -v, --verbose                    Be verbose
	    -h, --help                       Display this screen
	
	Common Options
	    -T, --target COLLECTIVE          Target messages to a specific sub collective
	        --dt, --discovery-timeout SECONDS
	                                     Timeout for doing discovery
	    -t, --timeout SECONDS            Timeout for calling remote agents
	    -q, --quiet                      Do not be verbose
	        --ttl TTL                    Set the message validity period
	        --reply-to TARGET            Set a custom target for replies
	        --dm, --disc-method METHOD   Which discovery method to use
	        --do, --disc-option OPTION   Options to pass to the discovery method
	        --nodes FILE                 List of nodes to address
	
	Host Filters
	    -W, --with FILTER                Combined classes and facts filter
	    -S, --select FILTER              Compound filter combining facts and classes
	    -F, --wf, --with-fact fact=val   Match hosts with a certain fact
	    -C, --wc, --with-class CLASS     Match hosts with a certain config management class
	    -A, --wa, --with-agent AGENT     Match hosts with a certain agent
	    -I, --wi, --with-identity IDENT  Match hosts with a certain configured identity
	
	The Marionette Collective 2.2.4

