# Install ERPNext on CentOS 8
在CentOS 8上安装ERPNext软件  
http://erpnext.com/
Some quick notes on installing ERPNext on CentOS 8.
在CentOS 8上安装ERPNext的一些注意事项。  
Just so you know:

 * This is a minimal working installation.

 * We don't cover making the box secure - that's up to you!

## Install CentOS8.
安装CentOS 8。
1) Download  CentOS8 on http://isoredirect.centos.org/centos/8/isos/x86_64/.
下载CentOS 8
2) Install CentOS 8. We chose the "minimal" install for this guide.
选择最小化安装<带GUI的服务器>  
3) After install, login and ensure your installation is up to date
   by running :
安装完成请更新系统
```sh
  sudo yum update -y
```
3) Install the extra packages repository:
添加epel-release源 
```sh
  sudo yum install -y epel-release
```
3) Replace packages repository(china):
替换成国内源
```sh
   #####   # file: /etc/yum.repos.d/CentOS-AppStream.repo
   [AppStream]  
   name=CentOS-$releasever - AppStream  
   baseurl=http://mirrors.aliyun.com/centos/$releasever/AppStream/$basearch/os/   
   gpgcheck=1   
   enabled=1   
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial   

   ##### # file: /etc/yum.repos.d/CentOS-Base.repo
   [BaseOS]  
   name=CentOS-$releasever - Base  
   baseurl=http://mirrors.aliyun.com/centos/$releasever/BaseOS/$basearch/os/  
   gpgcheck=1  
   enabled=1  
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial  

   ##### # file: /etc/yum.repos.d/CentOS-Epel.repo
   [epel]  
   name=CentOS-$releasever - Epel  
   baseurl=http://mirrors.aliyun.com/epel/8/Everything/$basearch  
   enabled=1  
   gpgcheck=0  

   ##### # file: /etc/yum.repos.d/CentOS-Media.repo
   [c8-media-BaseOS]  
   name=CentOS-BaseOS-$releasever - Media  
   baseurl=file:///media/CentOS/BaseOS/  
   gpgcheck=1  
   enabled=1  
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial  
   [c8-media-AppStream]  
   name=CentOS-AppStream-$releasever - Media  
   baseurl=file:///media/CentOS/AppStream/  
   gpgcheck=1  
   enabled=1  
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial  

或者  

   cd /etc/yum.repos.d/  
   mv /etc/yum.repos.d/CentOS-AppStream.repo /etc/yum.repos.d/CentOS-AppStream.repo.bak  
   mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak  
   mv /etc/yum.repos.d/CentOS-centosplus.repo /etc/yum.repos.d/CentOS-centosplus.repo.bak  
   mv /etc/yum.repos.d/CentOS-Extras.repo /etc/yum.repos.d/CentOS-Extras.repo.bak  
   mv /etc/yum.repos.d/CentOS-PowerTools.repo /etc/yum.repos.d/CentOS-PowerTools.repo.bak  

   curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-8.repo  
   yum makecache  
   yum -y update  
```

## Prepare OS for ERPNext
为安装ERPNext准备  
1) Install required packages:
安装依赖  
```sh
  curl --silent --location https://dl.yarnpkg.com/rpm/yarn.repo | sudo tee /etc/yum.repos.d/yarn.repo
  curl --silent --location https://rpm.nodesource.com/setup_14.x | sudo bash -
  sudo dnf groupinstall -y "Development Tools"  
  sudo yum install -y redhat-lsb-core git openssl-devel libffi-devel gcc make git mariadb mariadb-server 
  sudo yum install -y python39 python39-devel python39-setuptools python39-pip redis nodejs
  sudo npm install -g yarn
  python36 升级到39
  删除软连接
  rm -rf /usr/bin/python3
  rm -rf /usr/bin/pip3
  创建软连接
  ln -s /usr/bin/python3.9  /usr/bin/python3
  ln -s /usr/bin/pip3.9 /usr/bin/pip3
  python3 -V
  pip3 -V
  
  pip3 install --upgrade pip setuptools
  pip3 install ansible
  
  yarn add air-datepicker@github:frappe/air-datepicker
  yarn add node-sass
  ```
#### Install wkhtmltopdf: 
  ```sh
   wget https://github.com/wkhtmltopdf/wkhtmltopdf/releases  *.rpm
```

2) Create a user for ERPNext to run as, allowing it sudo access too:
添加用户  
```sh
   adduser bench -d /opt/bench
   passwd bench
   usermod -aG wheel bench
   sudo su - bench
```

3) (Optional) Configure sudo so it doesn't need a password:
不启用密码  
This step is optional but it might save you quite a bit of typing.
You might want to cut'n'paste this one!

```sh
  sudo sed -i 's/^#\s*\(%wheel\s\+ALL=(ALL)\s\+NOPASSWD:\s\+ALL\)/\1/' /etc/sudoers
```

4) Open the firewall:
开户防火墙策略（如果已经关闭了请忽略)  
```sh
  sudo firewall-cmd --zone=public --add-port=80/tcp
  sudo firewall-cmd --zone=public --add-port=443/tcp
  sudo firewall-cmd --zone=public --add-port=8000/tcp
  sudo firewall-cmd --runtime-to-permanent
```

5) Set some kernel parameters:
设置内核  
```sh
  echo "vm.overcommit_memory = 1" | sudo tee -a /etc/sysctl.conf
  echo "echo never > /sys/kernel/mm/transparent_hugepage/enabled" | sudo tee -a /etc/rc.d/rc.local
  sudo chmod 755 /etc/rc.d/rc.local 
```

6) Reboot:
重启  
This will allow the updates to settle and the kernel parameters to get set.

```sh
  sudo reboot
```

## Prepare MariaDB (mysql) for ERPNext
为ERPNext准备数据库（MariaDB 10)  
1) Edit the MariaDB configuration to set the correct character set:
编辑配置文件    
```sh
  cat <<EOF >/etc/my.cnf.d/erpnext.cnf
[mysqld]
innodb-file-format=barracuda
innodb-file-per-table=1
innodb-large-prefix=1
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[mysql]
default-character-set = utf8mb4
EOF
```

2) Enable and start the MariaDB service:
启动数据库  
```sh
  systemctl enable mariadb
  systemctl start mariadb
```

3) Secure the service:
设置数据库  
Start the secure script:

```sh
  mysql_secure_installation
```

This is an interactive script that will ask you questions.
设置密码  
Options are:
  * Current root password is none - just press enter.
  * Enter a new password for the root password - remember it!
  * Remove anonymous users - Y
  * Disallow remote root - Y
  * Remove test database - Y
  * Reload priv tables now - Y

Done!
3) Simple command:
常用命令  
```sh
   mysql -uroot -p
   create database 数据库名;
   drop database 数据库名;
   show databases;
   use 数据库名;
   drop table 表名;
   mysqldump -u 用户名 -p 数据库名 > 导出的文件名;
   exit;   
```
## Install ERPNext
安装ERPNext  

1) Switch to the ERP user (or login as it) and change to home directory:
切换用户并返回主目录  

```sh
  su erp
  cd
```

2) Install frappe-bench with pip and initialise:
安装Frappe-bench  

This step takes a while so get yourself a beer. It reaches out to the Internet
and downloads a bunch of stuff and then builds it.

```sh
  pip3 install --user frappe-bench
  bench init frappe-bench --frappe-branch version-12
```

For the second command, a red error message appears early on about an "editable
requirement." Ignore it.

When it's done you should get the message in green text:

```
  SUCCESS: Bench frappe-bench initialized
```

3) Create a new frappe site:
新增Frappe站点  

Prerequisites:
  * You need a name for your site. We called ours erpdev.softwaretohardware.com
  * You'll need your MariaDB root password from earlier.

First we temporarily start the frappe development server:

```sh
  cd frappe-bench
  sed -i '/web:/ s/$/ --noreload/' Procfile
  bench start >/tmp/bench_log &
```

Then we create a new site. Substitute your own name.

```sh
  bench new-site erpdev.leanbench.com # Create a new site
```

You will be prompted for the mysql password and a bit later, for the
adminstrator password for your new site.

NOTE: Don't visit your new site with a browser just yet!

4) Install the ERPNext application
安装ERPNext应用  

```sh
  bench get-app erpnext --branch version-12  # Add ERPNext to your bench apps  
  bench --site erpdev.leanbench.com install-app erpnext # Install ERPNext for the site
```

At the end of this step, the temporary server will stop and the exception
message looks bad. You can ignore it.

5) Bring back your temporary server

```sh
  bench start >/tmp/bench_log &
  bench update
```

You now have an ERPNext instance listening on port 8000.

Visit it with a browser to set it up.

When you're done, bring the server to the foreground and press Ctrl+C

```sh
  fg
```

You can start it again at any time.

## (Optional) Setup in production mode
生产环境设置  
Ensure the test server from above is not running.


1) Create the production configuration files for supervisor and nginx:在没有安装nginx&supervisor时安装

```sh
  bench setup supervisor
  bench setup nginx
```
2) setup nginx supervisor
```sh  
   yum install -y nginx supervisor
```
3) Set permissions including relaxing SELinux a bit:要在SELinux打开时设置设置权限和文件连接

```sh
  chmod 755 /home/erp
  chcon -t httpd_config_t config/nginx.conf
  sudo setsebool -P httpd_can_network_connect=1
  sudo setsebool -P httpd_enable_homedirs=1
  sudo setsebool -P httpd_read_user_content=1
```


4) Link the new configuration files to their respective services:要在SELinux打开时，将新配置文件链接到各自的服务

```sh
  sudo ln -s `pwd`/config/supervisor.conf /etc/supervisord.d/frappe-bench.ini
  sudo ln -s `pwd`/config/nginx.conf /etc/nginx/conf.d/frappe-bench.conf
```
5) Closed Firewall&SULinux:关闭Firewall&SULinux
```sh
  systemctl status firewalld #显示服务的状态  
  systemctl disable firewalld #在开机时禁用服务 
```
   sestatus -v 或getenforce  #显示服务的状态  
   永久关闭selinux    
```sh  
  vi /etc/selinux/config   
  SELINUX=disabled  默认值是: #SELINUX=enforcing  
```
4) Enable services to start at boot:
```sh
  sudo systemctl enable supervisord
  sudo systemctl enable nginx
```

5) Reboot:
```sh
  sudo reboot
```

After this your server should be accessible on port 80. You'll need to use the domain name you specified above when creating the site, otherwise you'll see the default nginx page.

## Bench Manager
工作台管理模组安装  
Bench Manager is a GUI frontend for Bench with the same functionalties. You can install it by executing the following command:
```sh
$ bench setup manager
```
Note: This will create a new site to setup Bench Manager, if you want to set it up on an existing site, run the following commands:
```sh
$ bench get-app https://github.com/frappe/bench_manager.git
$ bench --site <sitename> install-app bench_manager
```
