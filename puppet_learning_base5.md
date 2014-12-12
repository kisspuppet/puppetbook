#### Puppet基础篇5-如何建立master和agent之间的认证关系


Puppet注册方式基本上有三种：自动注册、手动注册和预签名注册，在《[Puppet基础篇4-安装、配置并使用Puppet](http://kisspuppet.com/2014/03/08/puppet_learning_base4/)》采用的是手动注册，不同的注册方式决定了注册的难易程度，当然安全性也会不一样。
<!--more-->

## 一、手动注册 ##
手动注册是由Agent端先发起证书申请请求，然后由Puppetserver端确认证书方可注册成功，这种注册方式安全系数中等，逐一注册（`puppet cert --sign certnmame`）在节点数量较大的情况下是比较麻烦的，效率也低，批量注册(`puppet cert --sign --all`)效率很高，一次性便可注册所有的Agent的请求，但是这种方式安全系数较低，因为错误的请求也会被注册上。

**1、节点申请注册**

	[root@agent1 ~]# puppet agent --test
	info: Creating a new SSL key for agent1_cert.kisspuppet.com
	info: Caching certificate for ca
	info: Creating a new SSL certificate request for agent1_cert.kisspuppet.com
	info: Certificate Request fingerprint (md5): 69:D2:86:E4:7F:00:E0:55:61:19:02:34:9E:9B:AF:F9
	Exiting; no certificate found and waitforcert is disabled

**2、服务器端确定认证**

	[root@puppetmaster ~]# puppet cert --list --all #查看认证情况
	  "agent1_cert.kisspuppet.com"  (69:D2:86:E4:7F:00:E0:55:61:19:02:34:9E:9B:AF:F9) #未认证
	+ "puppetmaster.kisspuppet.com" (C0:E3:6B:76:36:EC:92:93:4D:BF:F0:8F:77:00:91:C8) (alt names: "DNS:puppet", "DNS:puppet.kisspuppet.com", "DNS:puppetmaster.kisspuppet.com")
	
	[root@puppetmaster ~]# puppet cert --sign agent1_cert.kisspuppet.com #注册agent1
	notice: Signed certificate request for agent1_cert.kisspuppet.com #将请求的证书正式注册
	notice: Removing file Puppet::SSL::CertificateRequest agent1_cert.kisspuppet.com at '/var/lib/puppet/ssl/ca/requests/agent1_cert.kisspuppet.com.pem' #删除请求

	[root@puppetmaster ~]# puppet cert --list --all #再次查看认证情况
	+ "agent1_cert.kisspuppet.com"  (3E:46:4E:75:34:9A:5A:62:A6:3C:AE:BD:49:EE:C0:F5)
	+ "puppetmaster.kisspuppet.com" (C0:E3:6B:76:36:EC:92:93:4D:BF:F0:8F:77:00:91:C8) (alt names: "DNS:puppet", "DNS:puppet.kisspuppet.com", "DNS:puppetmaster.kisspuppet.com")

	[root@puppetmaster ~]# tree /var/lib/puppet/ssl/ #另外一种查看认证的方式
	/var/lib/puppet/ssl/
	├── ca
	│   ├── ca_crl.pem
	│   ├── ca_crt.pem
	│   ├── ca_key.pem
	│   ├── ca_pub.pem
	│   ├── inventory.txt
	│   ├── private
	│   │   └── ca.pass
	│   ├── requests
	│   ├── serial
	│   └── signed
	│       ├── agent1_cert.kisspuppet.com.pem  #已经注册成功
	│       └── puppetmaster.kisspuppet.com.pem
	├── certificate_requests
	├── certs
	│   ├── ca.pem
	│   └── puppetmaster.kisspuppet.com.pem
	├── crl.pem
	├── private
	├── private_keys
	│   └── puppetmaster.kisspuppet.com.pem
	└── public_keys
	    └── puppetmaster.kisspuppet.com.pem
	
	9 directories, 14 files


**3、motd模块测试**

	[root@agent1 ~]# puppet agent --test  #测试节点agent1
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1394304542'
	notice: /Stage[main]/Motd/File[/etc/motd]/content: 
	--- /etc/motd	2000-01-13 07:18:52.000000000 +0800
	+++ /tmp/puppet-file20140309-4571-1vqc18j-0	2014-03-09 02:51:47.000000000 +0800
	@@ -0,0 +1,3 @@
	+--                       --
	+--------puppet test---------
	+--                       --
	
	info: FileBucket adding {md5}d41d8cd98f00b204e9800998ecf8427e
	info: /Stage[main]/Motd/File[/etc/motd]: Filebucketed /etc/motd to puppet with sum d41d8cd98f00b204e9800998ecf8427e
	notice: /Stage[main]/Motd/File[/etc/motd]/content: content changed '{md5}d41d8cd98f00b204e9800998ecf8427e' to '{md5}87ea3a1af8650395038472457cc7f2b1'
	notice: Finished catalog run in 0.40 seconds

## 二、自动注册 ##
这种注册方式简单来讲是通过Puppetmaster端的ACL列表进行控制的，安全系统较低，也就是说符合预先定义的ACL列表中的所有节点请求不需要确认都会被自动注册上，也就是说你只需要知道ACL列表要求，其次能和PuppetMaster端通信便可轻易注册成功。当然，它的最大优点就是效率非常高。
**1、清除PuppetMaster端已经注册的agent1的证书**

	[root@puppetmaster ~]# puppet cert --clean agent1_cert.kisspuppet.com
	notice: Revoked certificate with serial 3
	notice: Removing file Puppet::SSL::Certificate agent1_cert.kisspuppet.com at '/var/lib/puppet/ssl/ca/signed/agent1_cert.kisspuppet.com.pem'
	notice: Removing file Puppet::SSL::Certificate agent1_cert.kisspuppet.com at '/var/lib/puppet/ssl/certs/agent1_cert.kisspuppet.com.pem'

	[root@puppetmaster ~]# puppet cert --list  --all #agent1证书已经删除
	+ "agent2_cert.kisspuppet.com"       (A0:CE:70:BE:A9:11:BF:F4:C8:EF:25:8E:C2:2C:3B:B7)
	+ "agent3_cert.kisspuppet.com"       (98:93:F7:0C:ED:94:81:3D:51:14:86:68:2B:F3:F1:A0)
	+ "puppetmaster.kisspuppet.com"      (C0:E3:6B:76:36:EC:92:93:4D:BF:F0:8F:77:00:91:C8) (alt names: "DNS:puppet", "DNS:puppet.kisspuppet.com", "DNS:puppetmaster.kisspuppet.com")
	+ "puppetmaster_cert.kisspuppet.com" (57:A3:D7:3D:64:2F:D6:FD:BC:2A:6C:79:68:73:EA:AB)

**2、在agent1端删除注册过的证书**

	[root@agent1 ~]# rm -rf /var/lib/puppet/ssl/*

**3、在Puppetmaster端编写ACL列表**

	[root@puppetmaster ~]# vim /etc/puppet/autosign.conf	
	*.kisspuppet.com
	[root@puppetmaster ~]# /etc/init.d/puppetmaster restart
	Stopping puppetmaster:                                     [  OK  ]
	Starting puppetmaster:                                     [  OK  ]
	[root@puppetmaster ~]# puppet cert --list  --all

**4、自动注册**

	[root@agent1 ~]# puppet agent --test #申请证书
	info: Creating a new SSL key for agent1_cert.kisspuppet.com
	info: Caching certificate for ca
	info: Creating a new SSL certificate request for agent1_cert.kisspuppet.com
	info: Certificate Request fingerprint (md5): ED:C9:C7:DF:F1:0E:53:1C:D3:73:5D:B7:D3:94:1F:60
	info: Caching certificate for agent1_cert.kisspuppet.com
	info: Caching certificate_revocation_list for ca
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1394359075'
	notice: Finished catalog run in 1.39 seconds
	[root@agent1 ~]# cat /etc/motd 
	--                       --
	--------puppet test---------
	--                       --

**5、服务器端查看**

	[root@puppetmaster ~]# puppet cert --list  --all  #agent1已经自动注册成功
	+ "agent1_cert.kisspuppet.com"       (9E:1A:2B:48:26:7D:26:8D:1D:F5:5E:34:A1:6B:13:5F)
	+ "agent2_cert.kisspuppet.com"       (A0:CE:70:BE:A9:11:BF:F4:C8:EF:25:8E:C2:2C:3B:B7)
	+ "agent3_cert.kisspuppet.com"       (98:93:F7:0C:ED:94:81:3D:51:14:86:68:2B:F3:F1:A0)
	+ "puppetmaster.kisspuppet.com"      (C0:E3:6B:76:36:EC:92:93:4D:BF:F0:8F:77:00:91:C8) (alt names: "DNS:puppet", "DNS:puppet.kisspuppet.com", "DNS:puppetmaster.kisspuppet.com")
	+ "puppetmaster_cert.kisspuppet.com" (57:A3:D7:3D:64:2F:D6:FD:BC:2A:6C:79:68:73:EA:AB)

**6、节点测试**

	[root@agent1 ~]# >/etc/motd #删除文件内容
	[root@agent1 ~]# puppet agent --test
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1394359075'
	notice: /Stage[main]/Motd/File[/etc/motd]/content: 
	--- /etc/motd	2014-03-09 17:59:02.000000000 +0800
	+++ /tmp/puppet-file20140309-3678-15tazyj-0	2014-03-09 17:59:06.000000000 +0800
	@@ -0,0 +1,3 @@
	+--                       --
	+--------puppet test---------
	+--                       --
	
	info: FileBucket got a duplicate file {md5}d41d8cd98f00b204e9800998ecf8427e
	info: /Stage[main]/Motd/File[/etc/motd]: Filebucketed /etc/motd to puppet with sum d41d8cd98f00b204e9800998ecf8427e
	notice: /Stage[main]/Motd/File[/etc/motd]/content: content changed '{md5}d41d8cd98f00b204e9800998ecf8427e' to '{md5}87ea3a1af8650395038472457cc7f2b1'
	notice: Finished catalog run in 0.42 seconds
	[root@agent1 ~]# cat /etc/motd #文件内容已经生成
	--                       --
	--------puppet test---------
	--                       --


## 三、预签名注册 ##
预签名注册是在agent端未提出申请的情况下，预先在puppetmaster端生成agent端的证书，然后复制到节点对应的目录下即可注册成功，这种方式安全系数最高，但是操作麻烦，需要提前预知所有节点服务器的certname名称，其次需要将生成的证书逐步copy到所有节点上去。不过，如果你的系统中安装了kickstart或者cobbler这样的自动化工具，倒是可以将证书部分转换成脚本集成到统一自动化部署中
**注：**生产环境中建议此方式进行注册，既安全又可靠！

**1、清除PuppetMaster端已经注册的agent1的证书**

	[root@puppetmaster ~]# puppet cert --clean agent1_cert.kisspuppet.com
	notice: Revoked certificate with serial 3
	notice: Removing file Puppet::SSL::Certificate agent1_cert.kisspuppet.com at '/var/lib/puppet/ssl/ca/signed/agent1_cert.kisspuppet.com.pem'
	notice: Removing file Puppet::SSL::Certificate agent1_cert.kisspuppet.com at '/var/lib/puppet/ssl/certs/agent1_cert.kisspuppet.com.pem'

	[root@puppetmaster ~]# puppet cert --list  --all #agent1证书已经删除
	+ "agent2_cert.kisspuppet.com"       (A0:CE:70:BE:A9:11:BF:F4:C8:EF:25:8E:C2:2C:3B:B7)
	+ "agent3_cert.kisspuppet.com"       (98:93:F7:0C:ED:94:81:3D:51:14:86:68:2B:F3:F1:A0)
	+ "puppetmaster.kisspuppet.com"      (C0:E3:6B:76:36:EC:92:93:4D:BF:F0:8F:77:00:91:C8) (alt names: "DNS:puppet", "DNS:puppet.kisspuppet.com", "DNS:puppetmaster.kisspuppet.com")
	+ "puppetmaster_cert.kisspuppet.com" (57:A3:D7:3D:64:2F:D6:FD:BC:2A:6C:79:68:73:EA:AB)

**2、在agent1端删除注册的所有信息，包括证书**

	[root@agent1 ~]# rm -rf /var/lib/puppet/*

**3、删除自动注册ACL列表**

	[root@puppetmaster ~]# mv /etc/puppet/autosign.conf{,.bak}

**4、puppetserver端预先生成agent1证书**

	[root@puppetmaster ~]# puppetca --generate agent1_cert.kisspuppet.com
	notice: agent1_cert.kisspuppet.com has a waiting certificate request
	notice: Signed certificate request for agent1_cert.kisspuppet.com
	notice: Removing file Puppet::SSL::CertificateRequest agent1_cert.kisspuppet.com at '/var/lib/puppet/ssl/ca/requests/agent1_cert.kisspuppet.com.pem'
	notice: Removing file Puppet::SSL::CertificateRequest agent1_cert.kisspuppet.com at '/var/lib/puppet/ssl/certificate_requests/agent1_cert.kisspuppet.com.pem'

**5、节点生成目录结构**

	[root@agent1 ~]# puppet agent --test --server=abc.com #随便指定server端，生成目录结构
	info: Creating a new SSL key for agent1_cert.kisspuppet.com
	err: Could not request certificate: getaddrinfo: Temporary failure in name resolution
	Exiting; failed to retrieve certificate and waitforcert is disabled
	[root@agent1 ~]# tree /var/lib/puppet/ssl/
	/var/lib/puppet/ssl/
	|-- certificate_requests
	|-- certs
	|-- private
	|-- private_keys
	|   `-- agent1_cert.kisspuppet.com.pem
	`-- public_keys
	    `-- agent1_cert.kisspuppet.com.pem
	
	5 directories, 2 files

**6、puppetmaster端copy证书到agent1上**

	[root@puppetmaster ~]# scp /var/lib/puppet/ssl/private_keys/agent1_cert.kisspuppet.com.pem  agent1.kisspuppet.com:/var/lib/puppet/ssl/private_keys/ 
	agent1_cert.kisspuppet.com.pem                                                         100% 3243     3.2KB/s   00:00    
	[root@puppetmaster ~]# scp /var/lib/puppet/ssl/certs/agent1_cert.kisspuppet.com.pem  agent1.kisspuppet.com:/var/lib/puppet/ssl/certs/
	agent1_cert.kisspuppet.com.pem                                                         100% 1944     1.9KB/s   00:00    
	[root@puppetmaster ~]# scp /var/lib/puppet/ssl/certs/ca.pem  agent1.kisspuppet.com:/var/lib/puppet/ssl/certs/
	ca.pem                                                                                 100% 1915     1.9KB/s   00:00    
	[root@puppetmaster ~]# 


**7、agent1测试**

	[root@agent1 ~]# >/etc/motd 
	[root@agent1 ~]# puppet agent --test
	info: Caching certificate_revocation_list for ca
	info: Caching catalog for agent1_cert.kisspuppet.com
	info: Applying configuration version '1394359075'
	notice: /Stage[main]/Motd/File[/etc/motd]/content: 
	--- /etc/motd	2014-03-09 18:18:10.000000000 +0800
	+++ /tmp/puppet-file20140309-4071-1gypudk-0	2014-03-09 18:18:17.000000000 +0800
	@@ -0,0 +1,3 @@
	+--                       --
	+--------puppet test---------
	+--                       --
	
	info: FileBucket adding {md5}d41d8cd98f00b204e9800998ecf8427e
	info: /Stage[main]/Motd/File[/etc/motd]: Filebucketed /etc/motd to puppet with sum d41d8cd98f00b204e9800998ecf8427e
	notice: /Stage[main]/Motd/File[/etc/motd]/content: content changed '{md5}d41d8cd98f00b204e9800998ecf8427e' to '{md5}87ea3a1af8650395038472457cc7f2b1'
	info: Creating state file /var/lib/puppet/state/state.yaml
	notice: Finished catalog run in 0.41 seconds
	[root@agent1 ~]# cat /etc/motd 
	--                       --
	--------puppet test---------
	--                       --


