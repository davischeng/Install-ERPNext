## Install ERPNext on CentOS 8
在CentOS 8上安装ERPNext
##### Some quick notes on installing ERPNext on CentOS 8.
有关在CentOS 8上安装ERPNext的一些快速注意事项。

##### Download  CentOS8 on http://isoredirect.centos.org/centos/8/isos/x86_64/.
下载CentOS8
##### Install CentOS8.
安装系统Centos8
##### Install CentOS 8. We chose the "minimal" install for this guide.
选择最小化安装<带GUI的服务器>
##### After install, login and ensure your installation is up to date by running :
安装完成请更新系统
#####   sudo yum update -y
Install the extra packages repository:
添加epel-release源
#####   sudo yum install -y epel-release
替换成国内源
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


##### Prepare OS for ERPNext  

##### Install required packages:
  sudo dnf groupinstall "Development Tools"  
  sudo yum install -y gcc make git mariadb mariadb-server nginx supervisor python3 python3-devel python2 python2-devel redis nodejs  
  sudo npm install -g yarn  
##### Create a user for ERPNext to run as, allowing it sudo access too:
  sudo useradd -m erp -G wheel  
(Optional) Configure sudo so it doesn't need a password:  
This step is optional but it might save you quite a bit of typing. You might want to cut'n'paste this one!  

  sudo sed -i 's/^#\s*\(%wheel\s\+ALL=(ALL)\s\+NOPASSWD:\s\+ALL\)/\1/' /etc/sudoers  
  
##### 关闭firewall
  systemctl status firewalld #显示服务的状态  
  systemctl disable firewalld #在开机时禁用服务  
  
##### 关闭Sulinux
  sestatus -v 或getenforce  #显示服务的状态  
  永久关闭selinux  
  vi /etc/selinux/config   
  SELINUX=disabled  默认值是: #SELINUX=enforcing  
  
##### 设置一些内核参数：
Set some kernel parameters:  
  echo "vm.overcommit_memory = 1" | sudo tee -a /etc/sysctl.conf  
  echo "echo never > /sys/kernel/mm/transparent_hugepage/enabled" | sudo tee -a /etc/rc.d/rc.local  
  sudo chmod 755 /etc/rc.d/rc.local   
  
Reboot:  
This will allow the updates to settle and the kernel parameters to get set.  

  sudo reboot  
  
##### Prepare MariaDB (mysql) for ERPNext

Edit the MariaDB configuration to set the correct character set:  
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

Enable and start the MariaDB service:  
  systemctl enable mariadb  
  systemctl start mariadb  
Secure the service:  
Start the secure script:  

  mysql_secure_installation  
This is an interactive script that will ask you questions.  

Options are:  

Current root password is none - just press enter.  
Enter a new password for the root password - remember it!  
Remove anonymous users - Y  
Disallow remote root - Y  
Remove test database - Y  
Reload priv tables now - Y  
Done!  

##### Install ERPNext

Switch to the ERP user (or login as it) and change to home directory:  
  su erp  
  cd  
Install frappe-bench with pip and initialise:  
This step takes a while so get yourself a beer. It reaches out to the Internet and downloads a bunch of stuff and then builds it.  

  pip3 install --user frappe-bench  
  bench init frappe-bench --frappe-branch version-12  
For the second command, a red error message appears early on about an "editable requirement." Ignore it.  

When it's done you should get the message in green text:  

  SUCCESS: Bench frappe-bench initialized  
Create a new frappe site:  
Prerequisites:  

You need a name for your site. We called ours erpdev.softwaretohardware.com
You'll need your MariaDB root password from earlier.
First we temporarily start the frappe development server:

  cd frappe-bench
  sed -i '/web:/ s/$/ --noreload/' Procfile
  bench start >/tmp/bench_log &
Then we create a new site. Substitute your own name.

  bench new-site erpdev.softwaretohardware.com
You will be prompted for the mysql password and a bit later, for the adminstrator password for your new site.

NOTE: Don't visit your new site with a browser just yet!

Install the ERPNext application
  bench get-app erpnext --branch version-12
  bench install-app erpnext
At the end of this step, the temporary server will stop and the exception message looks bad. You can ignore it.

Bring back your temporary server
  bench start >/tmp/bench_log &
  bench update
You now have an ERPNext instance listening on port 8000.

Visit it with a browser to set it up.

When you're done, bring the server to the foreground and press Ctrl+C

  fg
You can start it again at any time.

(Optional) Setup in production mode

Ensure the test server from above is not running.

Create the production configuration files for supervisor and nginx:
  bench setup supervisor
  bench setup nginx
Set permissions including relaxing SELinux a bit
  chmod 755 /home/erp
  chcon -t httpd_config_t config/nginx.conf
  sudo setsebool -P httpd_can_network_connect=1
  sudo setsebool -P httpd_enable_homedirs=1
  sudo setsebool -P httpd_read_user_content=1
Link the new configuration files to their respective services:
  sudo ln -s `pwd`/config/supervisor.conf /etc/supervisord.d/frappe-bench.ini
  sudo ln -s `pwd`/config/nginx.conf /etc/nginx/conf.d/frappe-bench.conf
Enable services to start at boot:
  sudo systemctl enable supervisord
  sudo systemctl enable nginx
Reboot:
  sudo reboot
After this your server should be accessible on port 80. You'll need to use the domain name you specified above when creating the site, otherwise you'll see the default nginx page.
