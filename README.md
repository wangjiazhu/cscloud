# CSCloud 部署

### 准备环境

#### 关闭防火墙等安全组件

（CS节点、MS节点、DS节点、FSG节点）

```shell
systemctl stop firewalld
systemctl disable firewalld

setenforce 0

cp /etc/selinux/config{,.bak}
sed -i 's/#SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```

#### LAMP环境

（CS节点、MS节点、DS节点）

- 更换安装源

```shell
# 备份原来的安装源
mkdir -p /etc/yum.repos.d/bak
find /etc/yum.repos.d/ -type f -exec mv {} /etc/yum.repos.d/bak/ \;

# 更换安装源为阿里源
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
```

- 配置Apache

```shell
# 安装
yum install httpd httpd-devel -y

# 启动Apache并设置开机自启
systemctl start httpd
systemctl enable httpd
```

测试Apache是否成功，输入地址 http://ip/

![](https://raw.githubusercontent.com/wjzcscloud/MarkTextImg/master/2022/03/27-21-06-57-20190312214639742.png)

- 配置并安装mysql

```shell
# 安装
yum install mariadb mariadb-server mariadb-libs mariadb-devel -y

# 启动mysql并设置开机自启
systemctl start mariadb
systemctl enable mariadb

# 设置mysql密码（密码必须是 111111）
mysql_secure_installation
# Set root password? [Y/n] y
# New password:111111
# Re-enter new password:111111
# Remove anonymous users? [Y/n] y
# Disallow root login remotely? [Y/n] n
# Remove test database and access to it? [Y/n] y
# Reload privilege tables now? [Y/n] y


# 登录mysql并验证是否成功
mysql -uroot -p111111
show databases;
```

- 安装PHP

```shell
yum install php -y
yum install php-mysql -y
yum install -y php-gd php-ldap php-odbc php-pear php-xml php-xmlrpc php-mbstring php-snmp php-soap curl-devel php-bcmath
```

- 测试LAMP环境是否安装成功

按下列命令顺序创建info.php文件

```shell
cd /var/www/html
touch info.php


cat >info.php <<EOF
<?php
phpinfo();
?>
EOF
```

输入 http://ip/info.php   出现以下页面表示LAMP环境搭建成功。

![](https://raw.githubusercontent.com/wjzcscloud/MarkTextImg/master/2022/03/27-21-12-59-20190312222122350.png)

#### expect工具包

（CS节点、MS节点、DS节点、FSG节点）

```shell
yum install expect -y
```

#### 安装必要的软件或工具，并进行相关配置

##### FSG节点

(以下操作在FSG节点进行)

1. 安装python环境以及相应的pip包

```shell
yum install python3 python3-pip -y
pip3 install fusepy==3.0.1
pip3 install celery==5.1.2
pip3 install eventlet==0.31.1
pip3 install PyMySQL==1.0.2
pip3 install requests==2.25.1
pip3 install urllib3==1.26.4
```

    2. 补充安装源，安装消息队列服务

```shell
touch /etc/yum.repos.d/rabbitmq.repo

cat >/etc/yum.repos.d/rabbitmq.repo <<EOF
[rabbitmq]
name=rabbitmq repo
baseurl=https://mirrors.aliyun.com/centos-vault/centos/7.4.1708/cloud/x86_64/openstack-ocata/
gpgcheck=0
enabled=1
EOF

# 安装消息队列
yum install rabbitmq-server -y

# 启动rabbitmq服务并设置开机自启动
systemctl start rabbitmq-server.service
systemctl enable rabbitmq-server.service

# 创建fsg用户并授权
rabbitmqctl add_user fsg 111111
rabbitmqctl set_permissions -p "/" fsg ".*" ".*" ".*"
```

    3. 补充安装源，安装PHP5.6开发环境

```shell
rpm -Uvh https://mirror.webtatic.com/yum/el7/epel-release.rpm
rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm

yum install -y php56w.x86_64 php56w-cli.x86_64 php56w-common.x86_64 php56w-gd.x86_64 php56w-ldap.x86_64 php56w-mbstring.x86_64 php56w-mcrypt.x86_64 php56w-mysql.x86_64 php56w-pdo.x86_64 php56w-fpm
```



##### CS节点、MS节点

(安装MySQL-python在CS和MS节点进行)

```shell
yum install MySQL-python -y
```



### CSCloud集群各节点部署

#### CS节点

- 将csc文件夹拷贝到服务器的/root目录

- 创建数据库csc

```shell
echo 'create database csc charset=utf8;'|mysql -uroot -p111111
```

- 进入目录  /root/csc/1.0/configmanager/scripts，并执行脚本

```shell
cd /root/csc/1.0/configmanager/scripts

sh csc_prepareDistro
```

<img title="" src="https://raw.githubusercontent.com/wjzcscloud/MarkTextImg/master/2022/03/27-21-11-22-uTools_1648214477761.png" alt="" width="811">

        执行以上脚本后，根目录下多出以下两个目录

![](https://raw.githubusercontent.com/wjzcscloud/MarkTextImg/master/2022/03/27-21-12-07-install_diectory.png)

        进入第一个目录,执行脚本（Suse服务器传入参数suse、深威为 rh，centos7也为rh，mysql密码必须为111111 ）

```shell
sh csc_admin_install.sh suse    #（有chown报错，再执行一遍）
```

![](https://raw.githubusercontent.com/wjzcscloud/MarkTextImg/master/2022/03/27-21-12-33-install_diectory2.png)

- 修改配置文件 /etc/httpd/conf/httpd.conf，修改默认站点，并配置cgi，配置内容如下：（注意：可将原配置文件备份，然后直接将准备好的httpd.conf配置文件拷贝到该处）

```shell
DocumentRoot "/srv/www/htdocs"

<Directory "/srv/www">
    AllowOverride None
    # Allow open access:
    Require all granted
</Directory>


<Directory "/srv/www/htdocs">
    #
    # Possible values for the Options directive are "None", "All",
    # or any combination of:
    #   Indexes Includes FollowSymLinks SymLinksifOwnerMatch ExecCGI MultiViews
    #
    # Note that "MultiViews" must be named *explicitly* --- "Options All"
    # doesn't give it to you.
    #
    # The Options directive is both complicated and important.  Please see
    # http://httpd.apache.org/docs/2.4/mod/core.html#options
    # for more information.
    #
    Options Indexes FollowSymLinks

    #
    # AllowOverride controls what directives may be placed in .htaccess files.
    # It can be "All", "None", or any combination of the keywords:
    #   Options FileInfo AuthConfig Limit
    #
    AllowOverride None

    #
    # Controls who can get stuff from this server.
    #
    Require all granted
</Directory>


ScriptAlias /cgi-bin/  "/srv/www/htdocs/"
<Directory "/srv/www/htdocs/">
    AllowOverride None
    #Options None
    Require all granted
    Options  FollowSymLinks ExecCGI
    AddHandler cgi-script .cgi .pl .py
    #Order allow,deny
    Allow from all
</Directory>
```

- 浏览器进入 http://cs节点IP地址/configmanager/login.php

| 用户名   | 密码     |
| ----- | ------ |
| admin | 123456 |

#### MS节点

- 登录到CS节点的前端管理台，点击“服务器配置”中的添加新服务器按钮。进行如下配置后，点击添加（IP地址为MS节点的IP）

![](https://raw.githubusercontent.com/wjzcscloud/MarkTextImg/master/2022/03/27-21-13-35-ms%E9%85%8D%E7%BD%AE.png)

完成以上添加后，进入“服务器安装”栏，点击“安装ManageServer”按钮后等待安装完毕。

- 进入http://ms节点IP地址/install/index.php 页面，创建csc数据库和管理员用户。

- 添加完毕后，MS节点的用户界面和管理员界面地址如下：

| 用户登录界面                    | 管理员登录界面               |
| ------------------------- | --------------------- |
| http://ms节点IP地址/login.php | http://ms节点IP地址/admin |

用户界面和管理员界面分别如下图所示;

| ![](https://raw.githubusercontent.com/wjzcscloud/MarkTextImg/master/2022/03/27-21-13-48-uTools_1648216666286.png) | ![](https://raw.githubusercontent.com/wjzcscloud/MarkTextImg/master/2022/03/27-21-13-59-uTools_1648216763644.png) |
| ----------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |

#### DS节点

- CS管理界面前端添加，配置如下:完成后，点击添加按钮即可。

![](https://raw.githubusercontent.com/wjzcscloud/MarkTextImg/master/2022/03/27-21-14-12-uTools_1648217054944.png)

完成以上添加后，进入“服务器安装”栏，点击“安装FileServer”按钮后等待安装完毕。

- 修改/etc/php.ini配置文件，内容如下：

| upload_max_filesize = 8M | post_max_size = 10M | memory_limit = 20M |
| ------------------------ | ------------------- | ------------------ |

#### FSG节点

- 创建目录/application,并将fsg文件夹整个拷贝到该目录下。

![](https://raw.githubusercontent.com/wjzcscloud/MarkTextImg/master/2022/03/27-21-14-25-uTools_1648217798660.png)

- 创建目录/data、/mountdir

```shell
mkdir -p /data 
mkdir -p /mountdir
mkdir -p /var/log/fsg
touch /var/log/fsg/fsg_{create,delete,download,rename,truncate}.log
```

- 修改配置文件 /application/fsg/conf/csc.conf

```shell
[csc]
cs_ip = 192.168.1.115    #cs节点IP
ms_ip = 192.168.1.116    #ms节点IP
obs = 001@qq.com,002@qq.com,003@qq.com,004@qq.com,005@qq.com,006@qq.com,007@qq.com,008@qq.com,009@qq.com,010@qq.com,011@qq.com,012@qq.com
pool = 8
size = 104857600
smallfile_size = 67108864
obsid_limit_small = 10
obsid_limit_big = 5
split_sum = 0.3
obsid_big_sum = 38
obsid_small_sum = 38

[path]
fsmonitor = /data    # 指定将要挂载的目录

[db]                # 数据库（不用改）
db_port = 3306
db_user = root
db_passwd = 111111
db_name = csc

[db_cs]            # 数据库（不用改）
db_port = 3306
db_user = root
db_passwd = 111111
db_name = configer

[192.168.1.117]    # 第一个ds节点的IP
obsid_limit_small = 5
obsid_limit_big = 5

[192.168.1.118]    # 第二个但是节点的IP，有更多的ds节点可再进行添加模块
obsid_limit_small = 4
obsid_limit_big = 4

[192.168.1.119]  # 第三个ds节点  
obsid_limit_small = 3
obsid_limit_big = 3
```

- 检查消息队列，并启动celery服务,如下图所示

```shell
systemctl status rabbitmq-server


# 进入/application/fsg目录下，启动celery服务
cd /application/fsg
celery -A celery_task.main worker -l info -P eventlet -c 4
```

![](https://raw.githubusercontent.com/wjzcscloud/MarkTextImg/master/2022/03/27-21-14-56-uTools_1648219997484.png)

- 进入/application/fsg/目录，启动fsg服务

```shell
# 启动fsg
cd /application/fsg
python3 fsg.py /mountdir
```

重新打开一个终端，创建文件进行测试，发现如下所示成功

![](https://raw.githubusercontent.com/wjzcscloud/MarkTextImg/master/2022/03/27-21-15-10-uTools_1648220272500.png)

- 关闭以上celery和fsg进程，最后将celery和fsg写成服务脚本，交给systemd进行管理

准备条件： FSG节点创建目录  /service/scripts

1.将celery.sh 脚本和 fsg.sh 脚本放入 /service/scripts 目录下

2.将celery.service脚本和fsg.service 脚本放入 /usr/lib/systemd/system/目录下

3.启动服务，并设置服务开机自启动

    注意：先启动celery服务，再启动fsg服务

```shell
systemctl start celery
systemctl enable celery
systemctl status celery

systemctl start fsg
systemctl enable fsg
systemctl status fsg


df -h
```

执行以上命令后，出现如下图所示的情况，成功

![](https://raw.githubusercontent.com/wjzcscloud/MarkTextImg/master/2022/03/27-21-15-36-uTools_1648274551885.png)

![](https://raw.githubusercontent.com/wjzcscloud/MarkTextImg/master/2022/03/27-21-15-48-uTools_1648274606176.png)

![](https://raw.githubusercontent.com/wjzcscloud/MarkTextImg/master/2022/03/27-21-16-03-uTools_1648274694099.png)

