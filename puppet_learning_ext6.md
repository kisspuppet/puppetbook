**Puppet扩展篇6-通过横向扩展puppetmaster增加架构的灵活性**

puppetmaster横向扩展将采用以下架构进行部署，也可以参考《puppet实战》第246页的内容。

![puppet集群扩展架构图](http://kisspuppet.com/img/puppet_learning_ext1.png)

<!--more-->

主机IP地址信息机用途表

![puppet集群扩展架构图](http://kisspuppet.com/img/puppet_learning_ext2.png)         



**工作原理：**

- 客户端通过配置ca_server指定CA服务器，以达到独立CA服务器的目的。
- CA服务器可以部署在多个机房。
- Master集群可以在同一机房配置负载均衡器，也可以使用DNS解析Puppet Master域名到不同机房的多台服务器，通过DNS实现负载均衡。

## 1、配置前的准备工作 ##

**1.1、版本**

puppet采用版本3.4.3
负载均衡器采用nginx或haproxy进行测试部署

**1.2、主机名解析**

	/etc/hosts
	192.168.10.10    kspupt-ca1
	192.168.10.9     kspupt-ca2	
	192.168.10.20    kspupt-ca	
	192.168.10.13    kspupt-lvs1	
	192.168.10.11    kspupt-m1	
	192.168.10.12    kspupt-m2	

**1.3、时间统一**
（略）

## 2、PuppetCA认证服务器部署 ##

PuppetCA的唯一目的就是签署和撤销证书。当PuppetCA服务不可用时，新的客户端将不能获得证书，从而会影响使用，而已签发证书的客户端缺不受影响。因此将CA进行独立队架构，这对容错性而言是非常有必要的。

**2.1 安装软件包**

	[root@kspupt-ca1 ~]# groupadd -g 3000 puppet
	[root@kspupt-ca1 ~]# useradd -u 3000 -g 3000 puppet
	[root@kspupt-ca1 ~]# yum install puppet puppet-server -y

**2.2 临时配置VIP地址**

	[root@kspupt-ca1 ~]# ip addr add 192.168.10.20/24 dev eth0

**注：**后期CA配置成了高可用后，将VIP地址添加到高可用资源中即可，临时先绑定在CA1上。

**2.3 生成证书**

使用puppet cert命令生成CA服务器与服务器域名证书。生成puppetca和puppetmaster两个域名的授权证书文件。

	[root@kspupt-ca1 ~]# puppet  cert --generate --dns_alt_names puppetca:puppet puppetca
	[root@kspupt-ca1 ~]# puppet  cert --generate --dns_alt_names puppetmaster:puppet puppetmaster
	[root@kspupt-ca1 ~]# puppet  cert --list --all  验证
	+ "puppetca"     (SHA256) 76:1D:C1:90:23:45:43:A2:41:4B:3B:92:32:C4:BE:31:38:61:5B:42:03:D0:22:28:53:5B:6F:5E:99:5A:B8:94 (alt names: "DNS:puppetca", "DNS:puppetca:puppet")
	+ "puppetmaster" (SHA256) 0A:A2:DC:22:B8:4C:EB:31:B0:52:8F:B0:21:72:DD:EB:C7:B4:05:97:45:B3:EA:19:3A:28:69:29:04:35:0F:E7 (alt names: "DNS:puppetmaster", "DNS:puppetmaster:puppet")
	
**2.4 配置puppet.conf，添加标签[master]**

	[root@kspupt-ca1 ~]# vim /etc/puppet/puppet.conf 
	[master]
	    confdir   = /etc/puppet
	    certname  = puppetca
	    ca        = true    #开启CA认证

**2.5 启动puppetmaster，CA部署完成**

	[root@kspupt-ca1 ssl]# /etc/init.d/puppetmaster start
	[root@kspupt-ca1 ssl]# chkconfig puppetmaster on

**kspupt-ca2配置（略）**

## 3、PuppetMaster服务器部署 ##

PuppetMaster服务器部署可采用默认的WebRick方式，也可以采用apache+passenger或nginx+passenger方式。


**3.1 WebRick方式：**

**3.1.1 安装软件包**

	[root@kspupt-m1 ~]# groupadd -g 3000 puppet
	[root@kspupt-m1 ~]# useradd -u 3000 -g 3000 puppet
	[root@kspupt-m1 ~]# yum install puppet puppet-server -y

**3.1.2 设置hosts文件**

	[root@kspupt-m1 ~]# vim /etc/hosts
	192.168.10.20 puppetca
	192.168.10.11 puppetmaster

**3.1.3 创建证书目录**

	[root@kspupt-m1 ~]# mkdir /var/lib/puppet/ssl/{certs,ca,private_keys} -p

**3.1.4 将puppetca上生成的puppetmaster公钥、私钥和根证书复制到kspupt-m1**

	[root@kspupt-m1 ssl]# scp -r root@192.168.10.39:/var/lib/puppet/ssl/ca/signed/puppetmaster.pem /var/lib/puppet/ssl/certs/puppetmaster.pem 
	[root@kspupt-m1 ssl]# scp -r root@192.168.10.39:/var/lib/puppet/ssl/ca/ca_crt.pem  /var/lib/puppet/ssl/certs/ca.pem  
	[root@kspupt-m1 ssl]# scp -r root@192.168.10.39:/var/lib/puppet/ssl/private_keys/puppetmaster.pem /var/lib/puppet/ssl/private_keys/puppetmaster.pem
	[root@kspupt-m1 gem]# scp -r root@192.168.10.39:/var/lib/puppet/ssl/ca/ca_crl.pem /var/lib/puppet/ssl/ca/ca_crl.pem

**3.1.5 配置puppet.conf，添加标签[master]，关闭ca**

	[root@kspupt-m1 ~]# vim /etc/puppet/puppet.conf
	[master]
	    certname = puppetmaster
	    ca       = false   #关闭CA认证

**3.1.6 配置puppet.conf，修改标签[agent]，增加server和ca_server字段**

	[root@kspupt-m1 ~]# vim /etc/puppet/puppet.conf
	[agent]
	    server      = puppetmaster
	    ca_server   = puppetca

**3.1.7 启动puppetmaster服务，Puppetmaster部署完成**

	[root@kspupt-m1 ~]# /etc/init.d/puppetmaster start

**3.1.8 运行puppet命令进行本地证书申请**

	[root@kspupt-m1 ~]# puppet  agent -t
	Info: Creating a new SSL key for kspupt-m1
	Info: csr_attributes file loading from /etc/puppet/csr_attributes.yaml
	Info: Creating a new SSL certificate request for kspupt-m1
	Info: Certificate Request fingerprint (SHA256): 78:A5:F2:6C:F6:EE:0C:25:0C:EF:96:B8:B4:E6:78:74:A6:AA:67:81:6B:8F:36:AC:B2:37:B5:E0:C1:F0:11:67
	Exiting; no certificate found and waitforcert is disabled

**3.1.9 登录puppetca进行证书签发**

	[root@kspupt-ca ~]# puppet  cert --sign kspupt-m1
	Notice: Signed certificate request for kspupt-m1
	Notice: Removing file Puppet::SSL::CertificateRequest kspupt-m1 at '/var/lib/puppet/ssl/ca/requests/kspupt-m1.pem'

**3.1.10 再次运行puppet命令进行测试连通性**

	[root@kspupt-m1 ~]# puppet  agent -t
	Info: Caching certificate for kspupt-m1
	Info: Caching certificate_revocation_list for ca
	Info: Caching certificate for kspupt-m1
	Info: Retrieving pluginfacts
	Info: Retrieving plugin
	Info: Caching catalog for kspupt-m1
	Info: Applying configuration version '1409296030'
	Info: Creating state file /var/lib/puppet/state/state.yaml
	Notice: Finished catalog run in 0.02 seconds

**3.1.11 在kspupt-ca上申请本地证书**

	[root@kspupt-ca ~]# vim /etc/puppet/puppet.conf
	[agent]
	    server    = puppetmaster
	    ca_server = puppetca
	[root@kspupt-ca ~]# puppet agent -t
	[root@kspupt-ca ~]# puppet cert --sign kspupt-ca
	[root@kspupt-ca ~]# puppet agent -t


## 3.2 Nginx+Passenger方式： ##

**注：**可参考 [http://kisspuppet.com/2014/10/20/puppet_learning_ext4/](http://kisspuppet.com/2014/10/20/puppet_learning_ext4/)

**3.2.1、安装相关开发包**

	[root@kspupt-m1 ~]# groupadd -g 3001 nginx
	[root@kspupt-m1 ~]# useradd -u 3001 -g 3001 nginx
	[root@kspupt-m1 ~]# yum install ruby-devel gcc make pcre-devel zlib-devel openssl-devel pam-devel curl-devel rpm-build

**3.2.2、安装passenger（将gem软件包copy到本地）**

	[root@kspupt-m1 gem]# gem install rake rack passenger --no-rdoc --no-ri

**3.2.3、解压nginx、pcre源码包**

	[root@kspupt-m1 gem]# tar xf pcre-8.32.tar.gz -C /usr/local/src/
	[root@kspupt-m1 gem]# tar xf nginx-1.4.2.tar.gz -C /usr/local/src/

**3.2.4、编译并安装nginx**

	[root@kspupt-m1 ~]# cd /usr/local/src/nginx-1.4.2/
	[root@kspupt-m1 nginx-1.4.2]# ./configure --user=nginx --group=nginx --prefix=/etc/nginx --with-http_stub_status_module --with-http_ssl_module --with-pcre=/usr/local/src/pcre-8.32 --add-module=`passenger-config --root`/ext/nginx
	[root@kspupt-m1 nginx-1.4.2]# make && make install

**3.2.5、与passenger结合**

	[root@kspupt-m1 nginx-1.4.2]# mkdir  -p /etc/puppet/rack/public
	[root@kspupt-m1 nginx-1.4.2]# cp /usr/share/puppet/ext/rack/config.ru  /etc/puppet/rack/
	[root@kspupt-m1 nginx-1.4.2]# chown -R puppet. /etc/puppet/rack/

**3.2.6、复制启动脚本到**

	[root@kspupt-m1 init.d]# cp /root/gem/nginx /etc/init.d/
	[root@kspupt-m1 ~]# chmod a+x /etc/init.d/nginx 

**3.2.7、配置nginx**

	[root@kspupt-m1 gem]# vim /etc/nginx/conf/nginx.conf
	user  nginx nginx;
	worker_processes  1;
	pid        /var/run/nginx.pid;
	
	events {
	    worker_connections  1024;
	}
	
	http {
	    passenger_root /usr/lib/ruby/gems/1.8/gems/passenger-4.0.19;
	    passenger_ruby /usr/bin/ruby;
	
	    include       mime.types;
	    default_type  application/octet-stream;
	
	    sendfile        on;
	    keepalive_timeout  65;
	    
	    server {
	        listen 8140                ssl;
		server_name                puppetmaster;
		
		passenger_enabled          on;
		passenger_set_cgi_param    HTTP_X_CLIENT_DN $ssl_client_s_dn;
		passenger_set_cgi_param    HTTP_X_CLIENT_VERIFY $ssl_client_verify;
		proxy_buffer_size 4000k;
		proxy_buffering on;
		proxy_buffers 32 1280k;
		proxy_busy_buffers_size 17680k;
		client_max_body_size 10m;
		client_body_buffer_size 4096k;
		
		access_log /var/log/nginx/puppet_access.log;
		error_log /var/log/nginx/puppet_error.log;
		
		root /etc/puppet/rack/public;
		
		ssl off;
		ssl_session_timeout 5m;
		ssl_certificate /var/lib/puppet/ssl/certs/puppetmaster.pem;
		ssl_certificate_key /var/lib/puppet/ssl/private_keys/puppetmaster.pem;
		ssl_client_certificate /var/lib/puppet/ssl/certs/ca.pem;
		ssl_crl /var/lib/puppet/ssl/ca/ca_crl.pem;
		ssl_verify_client optional;
		
		ssl_ciphers SSLv2:-LOW:-EXPORT:RC4+RSA;
		ssl_prefer_server_ciphers on;
		ssl_verify_depth 1;
		ssl_session_cache shared:SSL:128m;
			
		# File sections
		location /production/file_content/files/ {
		  types { }
		  default_type application/x-raw;
		  alias /etc/puppet/files/;
	  }
	 }
	}

**3.2.8、配置puppet.conf**

	[root@kspupt-m1 ~]# vim /etc/puppet/puppet.conf 
	[master]
	    certname = puppetmaster
	    ca       = false
	    ssl_client_verify_header = HTTP_X_CLIENT_VERIFY
	    ssl_client_header = HTTP_X_CLIENT_DN

**3.2.9、启动nginx**

	[root@kspupt-m1 gem]# mkdir /var/log/nginx/
	[root@kspupt-m1 nginx-1.4.2]# /etc/init.d/puppetmaster stop
	[root@kspupt-m1 nginx-1.4.2]# chkconfig puppetmaster off
	[root@kspupt-m1 nginx-1.4.2]# /etc/init.d/nginx start
	[root@kspupt-m1 nginx-1.4.2]# chkconfig nginx on

**3.2.10、测试**

在多个节点发起puppet agent -t命令动作

	[root@kspupt-ca ~]# puppet  agent -t
	[root@kspupt-m1 ~]# puppet  agent -t
	[root@kspupt-m1 ~]# tailf  /var/log/nginx/puppet_access.log

**tkpupt-m2安装(略)**

## 4 Puppet LB负载均衡器部署 ##

**4.1 puppet认证建立**

**4.1.1、安装软件包**

	[root@kspupt-lvs1 ~]# groupadd -g 3000 puppet
	[root@kspupt-lvs1 ~]# useradd -u 3000 -g 3000 puppet
	[root@kspupt-lvs1 ~]# yum install puppet

**4.1.2、编辑hosts文件**

	[root@kspupt-lvs1 ~]# vim /etc/hosts
	192.168.10.20 puppetca
	192.168.10.11 puppetmaster
	192.168.10.13 kspupt-lvs1

**4.1.3、创建证书目录**

	[root@kspupt-lvs1 ~]# mkdir /var/lib/puppet/ssl/{certs,ca,private_keys} -p

**4.1.4、将kspupt-ca上生成的puppetmaster公钥、私钥和根证书复制到kspupt-lvs1**

	[root@kspupt-lvs1 ssl]# scp -r root@192.168.10.10:/var/lib/puppet/ssl/ca/signed/puppetmaster.pem /var/lib/puppet/ssl/certs/puppetmaster.pem 
	[root@kspupt-lvs1 ssl]# scp -r root@192.168.10.10:/var/lib/puppet/ssl/ca/ca_crt.pem  /var/lib/puppet/ssl/certs/ca.pem  
	[root@kspupt-lvs1 ssl]# scp -r root@192.168.10.10:/var/lib/puppet/ssl/private_keys/puppetmaster.pem /var/lib/puppet/ssl/private_keys/puppetmaster.pem
	[root@kspupt-lvs1 ssl]# scp -r root@192.168.10.10:/var/lib/puppet/ssl/ca/ca_crl.pem /var/lib/puppet/ssl/ca/

**4.1.5、配置puppet.conf，修改标签[agent]，增加server和ca_server字段**

	[root@kspupt-lvs1 ~]# vim /etc/puppet/puppet.conf
	[agent]
	    server      = puppetmaster
	    ca_server   = puppetca

**4.1.6、运行puppet命令进行本地证书申请**

	[root@kspupt-lvs1 ~]# puppet  agent -t 

**4.1.7、登录kspupt-ca进行证书签发**

	[root@kspupt-ca1 ~]# puppet  cert --sign kspupt-lvs1

**4.1.8、再次运行puppet命令进行测试连通性**

	[root@kspupt-lvs1 ~]# puppet  agent -t
	Info: Retrieving pluginfacts
	Info: Retrieving plugin
	Info: Caching catalog for kspupt-lvs1
	Info: Applying configuration version '1409210667'

**4.2 安装并配置nginx负载均衡器**

**4.2.1、安装nginx软件**

	[root@kspupt-lvs1 ~]# groupadd -g 3001 nginx
	[root@kspupt-lvs1 ~]# useradd -u 3001 -g 3001 nginx
	[root@kspupt-lvs1 ~]# yum install nginx

**4.2.2、临时设置VIP地址（后面通过高可用软件代替）**

	[root@kspupt-lvs1 ~]# ip addr add 192.168.10.18/24 dev eth0


**4.2.3、配置nginx虚拟主机，添加upstrem**

	[root@kspupt-lvs1 ~]# vim /etc/nginx/conf.d/puppetmaster.conf
	upstream puppet-master {
	   server 192.168.10.11:8140;
	   server 192.168.10.12:8140;
	}
	server {
	   listen         8140 ssl;
	   server_name    puppetmaster;
	
	   access_log     /var/log/nginx/puppet_access.log;
	   error_log      /var/log/nginx/puppet_error.log;
	
	   ssl_protocols SSLv3 TLSv1;
	   ssl_ciphers ALL:!ADH:RC4+RSA:+HIGH:+MEDIUM:-LOW:-SSLv2:-EXP;
	
	   proxy_set_header             X-SSL-Subject  $ssl_client_s_dn;
	   proxy_set_header             X-Client-DN  $ssl_client_s_dn;
	   proxy_set_header             X-Client-Verify  $ssl_client_verify;
	   client_max_body_size 100m;
	   client_body_buffer_size 1024k;
	   proxy_buffer_size 100m;
	   proxy_buffers 8 100m;
	   proxy_busy_buffers_size 100m;
	   proxy_temp_file_write_size 100m;
	   proxy_read_timeout 500;
	   
	   ssl                     on;
	   ssl_session_timeout     5m;
	   ssl_certificate         /var/lib/puppet/ssl/certs/puppetmaster.pem;
	   ssl_certificate_key     /var/lib/puppet/ssl/private_keys/puppetmaster.pem;
	   ssl_client_certificate  /var/lib/puppet/ssl/certs/ca.pem;
	   ssl_crl                 /var/lib/puppet/ssl/ca/ca_crl.pem;
	   ssl_verify_client       optional;
	
	   ssl_prefer_server_ciphers  on;
	   ssl_verify_depth           1;
	   ssl_session_cache          shared:SSL:128m;
	   location / {
	         proxy_redirect    off;
	         proxy_pass        https://puppet-master;
	  }
	}

**4.2.4、编辑hosts文件，puppetmaster解析指向VIP**

	[root@kspupt-lvs1 ~]# vim /etc/hosts
	192.168.10.20 puppetca
	192.168.10.18 puppetmaster
	192.168.10.13 kspupt-lvs1


**4.2.5、修改kspupt-ca和kspupt-m1的hosts文件puppetmaster解析**

	[root@kspupt-ca1 ~]# vim /etc/hosts
	192.168.10.20 puppetca
	192.168.10.18 puppetmaster
	[root@kspupt-m1 ~]# vim /etc/hosts
	192.168.10.20 puppetca
	192.168.10.18 puppetmaster

**4.2.6、启动nginx服务器**

	[root@kspupt-lvs1 ~]# /etc/init.d/nginx start

**4.2.7、再次运行puppet命令进行测试连通性**

	[root@kspupt-ca1 ~]# puppet  agent -t
	[root@kspupt-m1 ~]# puppet  agent -t
	[root@kspupt-lvs1 ~]# puppet  agent -t
	[root@kspupt-m1 ~]# tailf  /var/log/nginx/puppet_access.log
	[root@kspupt-lvs1 ~]# tailf /var/log/nginx/puppet_access.log

**kspupt-lvs2(略)**

**4.3 HAproxy负载均衡配置参考**

	[root@kspupt-lvs2 ~]# cat /etc/haproxy/haproxy.cfg
	listen admin_stats 
	    bind 0.0.0.0:8080
	    mode http
	    stats refresh 5s
	    stats enable
	    stats hide-version
	    stats realm Haproxy\ Statistics
	    stats uri /haproxy
	    stats auth admin:password
	
	listen puppetmaster *:8140
	    mode tcp
	    option ssl-hello-chk
	#    option tcplog
	    #balance source
	#    balance roundrobin
	    balance source
	    server kspupt-m1 kspupt-m1:8140 check inter 2000 fall 3
	    server kspupt-m2 kspupt-m2:8140 check inter 2000 fall 3


