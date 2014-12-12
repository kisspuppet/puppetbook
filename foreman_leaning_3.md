#### foreman架构的引入3-安装Foreman1.5.3架构（all-in-one）



**注意：**本实验是在离线情况下安装的，所以需要在本地创建自己的yum仓库，创建方法可参考《[如何根据版本制作属于自己的puppet yum源](http://kisspuppet.com/2014/01/26/puppet_create_repo/)》，如何你实在是比较懒或者搞不定rpm包之间的依赖关系，那就去我的github上下载吧：[https://github.com/kisspuppet/foreman-repo](https://github.com/kisspuppet/foreman-repo)

更多安装细节请参考官网：[http://theforeman.org/manuals/1.5/index.html#Releasenotesfor1.5.4](http://theforeman.org/manuals/1.5/index.html#Releasenotesfor1.5.4)

<!--more-->

以下all-in-one安装方式跟官方安装的有所区别，官方安装可能只需要一条命令就可以安装成功，在我测试下来发现会出现有时候成功，有时候不成功的现象，所以改成了以下方式安装，而且每次都能成功，条例也比较清晰，为后面拆分puppetmaster能够提供很好的帮助。


## 1、软件包的选型如下： ##
- **puppet-server     3.6.2**
- **puppet            3.6.2**
- **facter            2.0.2**
- **mcollective       2.2.4**
- **rabbitmq-server   3.2.4**
- **foreman           1.5.3**
- **foreman-proxy     1.5.4**

## 2、系统环境准备 ##

**系统版本：**

	[root@foreman02 yum.repos.d]# cat /etc/redhat-release 
	Red Hat Enterprise Linux Server release 6.5 (Santiago)

**网络参数：**

	[root@foreman02 yum.repos.d]# ip addr
	1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN 
	    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
	    inet 127.0.0.1/8 scope host lo
	    inet6 ::1/128 scope host 
	       valid_lft forever preferred_lft forever
	2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
	    link/ether 00:50:56:a6:5c:70 brd ff:ff:ff:ff:ff:ff
	    inet 192.168.10.159/24 brd 192.168.10.255 scope global eth0
	    inet6 fe80::250:56ff:fea6:5c70/64 scope link 
	       valid_lft forever preferred_lft forever
 
**主机名称：**
      
	[root@foreman02 yum.repos.d]# hostname -f
	foreman02.kisspuppet.com
	[root@foreman02 yum.repos.d]# cat /etc/hosts
	127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
	::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
	192.168.10.159  foreman02.kisspuppet.com  foreman02
       
**平台环境：**
       
	[root@foreman02 yum.repos.d]# uname -r
	2.6.32-431.el6.x86_64

**yum仓库：**

	[root@foreman02 yum.repos.d]# cat foreman153.repo 
	[foreman]
	name=Foreman
	baseurl=ftp://192.168.10.254/blog/foreman
	enabled=1
	gpgcheck=0
	
	[puppet]
	name=puppet
	baseurl=ftp://192.168.10.254/blog/puppet-el6
	enabled=1
	gpgcheck=0
	
	[rhel]
	name=RHEL
	baseurl=ftp://192.168.10.254/rhel6.5
	enabled=1
	gpgcheck=0

**网络安全环境：**

	[root@foreman02 ~]# /etc/init.d/iptables status
	iptables: Firewall is not running.
	[root@foreman02 ~]# getenforce 
	Disabled

## 3、安装Foreman ##

**3.1、安装puppetmaster，并生成CA和证书**

	[root@foreman02 ~]# yum install foreman-installer
	[root@foreman02 ~]# yum install puppet-server puppet facter
	[root@foreman02 ~]# vim /etc/puppet/puppet.conf
	[master]
	    certname = foreman02.kisspuppet.com
    
	[root@foreman02 ~]# /etc/init.d/puppetmaster start
	Starting puppetmaster:                                     [  OK  ]
	[root@foreman02 ~]# puppet  cert --list --all
	+ "foreman02.kisspuppet.com" (SHA256) 1D:7E:90:F5:16:7D:01:67:77:37:EE:31:3F:46:AD:0A:47:80:B6:DF:6A:5E:25:A8:DE:BA:78:45:C9:09:D6:BD (alt names: "DNS:foreman02.kisspuppet.com", "DNS:puppet", "DNS:puppet.kisspuppet.com")
	[root@foreman02 ~]# /etc/init.d/puppetmaster stop
	Stopping puppetmaster:                                     [  OK  ]

**3.2、安装foreman及依赖包**

	[root@foreman02 ~]# yum install foreman  mod_passenger mod_ssl ruby193-rubygem-passenger-native mysql mysql-server foreman-mysql2  

**3.3、通过foreman-installer安装foreman**

foreman默认安装选择的数据库为postgresql，这里选用mysql进行安装。

	[root@foreman02 ~]# foreman-installer --foreman-db-adapter mysql2 --foreman-db-type mysql --no-enable-puppet --no-enable-foreman-proxy --foreman-configure-epel-repo=false
	Installing             Done                                               [100%] [...................]
	  Success!
	  * Foreman is running at https://foreman02.kisspuppet.com
	      Default credentials are 'admin:changeme'
	  The full log is at /var/log/foreman-installer/foreman-installer.log

安装完成之后，通过火狐或者谷歌浏览器访问看是否安装成功https://192.168.10.159
 
![Foreman安装](http://kisspuppet.com/img/foreman02-1.png)

![Foreman安装](http://kisspuppet.com/img/foreman02-2.png)


**3.4、安装foreman-proxy及依赖包**

	[root@foreman02 ~]# yum install tftp-server syslinux foreman-proxy

**3.5、安装foreman-proxy，并通过foreman-installer重新安装foreman和puppetmaster**

**注意：**以下方式是安装后会代理TFTP, DNS, DHCP, Puppet, and Puppet CA，并且puppetmaster会以apache+passenger的方式安装运行。

	[root@foreman02 ~]# foreman-installer --enable-foreman --enable-foreman-proxy --enable-puppet  --puppet-server=true --foreman-proxy-puppetrun=true  --foreman-proxy-puppetca=true   --foreman-proxy-dhcp=true  --foreman-proxy-tftp=true  --foreman-proxy-dns=true --foreman-proxy-dns-interface=eth0 --foreman-proxy-dns-zone=kisspuppet.com  --foreman-proxy-dns-reverse=10.168.192.in-addr.arpa  --foreman-proxy-dns-forwarders=8.8.8.8 --foreman-proxy-dns-forwarders=8.8.4.4 --foreman-configure-epel-repo=false  --foreman-proxy-register-in-foreman=false 
	
	Installing             Done                                               [100%] [...................]
	  Success!
	  * Foreman is running at https://foreman02.kisspuppet.com
	      Default credentials are 'admin:changeme'
	  * Foreman Proxy is running at https://foreman02.kisspuppet.com:8443
	  * Puppetmaster is running at port 8140
	  The full log is at /var/log/foreman-installer/foreman-installer.log

如果只代理puppet和puppetCA，可以通过以下方式安装

	[root@foreman02 ~]# foreman-installer --enable-foreman --enable-foreman-proxy --enable-puppet  --puppet-server=true --foreman-proxy-puppetrun=true  --foreman-proxy-puppetca=true    --foreman-configure-epel-repo=false  --foreman-proxy-register-in-foreman=false 
  
## 4、检查foreman、foreman-proxy、puppetmaster是否安装成功 ##

	[root@foreman02 ~]# /etc/init.d/httpd status
	httpd (pid  25433) is running...
	[root@foreman02 ~]# /etc/init.d/foreman-proxy status
	foreman-proxy (pid  25605) is running...
	
	[root@foreman02 ~]# netstat -naltp | grep 8443
	tcp        0      0 0.0.0.0:8443                0.0.0.0:*                   LISTEN      25605/ruby          
	[root@foreman02 ~]# netstat -naltp | grep 80
	tcp        0      0 :::80                       :::*                        LISTEN      25433/httpd         
	[root@foreman02 ~]# netstat -naltp | grep 8140
	tcp        0      0 :::8140                     :::*                        LISTEN      25433/httpd    

## 5、在Foreman上注册foreman-proxy ##

如果要管理puppet、puppetca等软件，是需要通过foreman-proxy去代理才能够正常使用的，关于代理的开启和关闭可以修改它的配置文件`/etc/foreman-proxy/settings.yml ` 


![Foreman安装](http://kisspuppet.com/img/foreman02-3.png)

![Foreman安装](http://kisspuppet.com/img/foreman02-4.png)

![Foreman安装](http://kisspuppet.com/img/foreman02-5.png)

![Foreman安装](http://kisspuppet.com/img/foreman02-6.png)


