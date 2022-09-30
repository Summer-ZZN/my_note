# Linux系统操作备忘



## 软件安装



### 一、设置yum源

1、CentOS-6 的操作系统， yum镜像仓库在 2020 年的时候停止对其的维护，导致不能正常使用。

#### 备份

```sh
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup

mv /etc/yum.repos.d/epel.repo /etc/yum.repos.d/epel.repo.backup
```

#### 下载

```sh
wget -O /etc/yum.repos.d/CentOS-Base.repo https://gitee.com/yeungchie/CentOS6_yum/raw/master/CentOS6-Base.repo

wget -O /etc/yum.repos.d/epel.repo https://gitee.com/yeungchie/CentOS6_yum/raw/master/epel6.repo
```

#### 更新

```sh
yum clean all

yum makecache
```

如果还是提示没有下载资源，尝试更新命令

```sh
yum update -y
```



### 二、设置pg_dump工具

在使用pg_dump工具对数据库进行备份的时候，会提示pg_dump版本和数据库版本不一致；

```sh
pg_dump: server version: 10.6; pg_dump version: 9.6.22
```

我们需要升级一下pg_dump；

1、查看pg_dump文件位置

```sh
sudo find / -name pg_dump
```

2、将/usr/bin/pg_dump文件拷贝下来，预防出什么问题，然后将/usr/pgsql-10/bin/pg_dump复制到/usr/bin/目录下

```sh
cd /usr/bin/
sudo mv pg_dump pg_dump.9.2.back
sudo cp /usr/pgsql-11/bin/pg_dump ./
```

再次备份数据库数据，可以成功备份。

```sh
pg_dump --username=root --host=192.168.1.xxx --port=5432 --format=plain --file=[备份文件名.backup] [数据库名]
```



### 三、centos7  开机设置应用自启动

#### 1、方式一

采用 rc.local ;相关启动命令在/etc/rc.d/rc.local 这个文件中添加

```sh
chmod +x /etc/rc.d/rc.local        #设置执行权限
systemctl enable rc-local.service  #开机启动
systemctl start rc-local.service   #启动服务
systemctl status rc-local.service  #查看服务启动状态
```

  注意: tomcat启动依赖于 java环境;可以在其中先添加java环境,例如以下rc.local 脚本示例

```sh
touch /var/lock/subsys/local
# 添加java环境
export JAVA_HOME=/usr/java/jdk1.8.0_131
export JRE_HOME=${JAVA_HOME}/jre
/data/devHome/protection-4.0/apache-tomcat-admin-8.5.32-1130/bin/startup.sh
/data/devHome/protection-4.0/apache-tomcat-patrol-8.5.32-1140/bin/startup.sh
/data/devHome/protection-4.0/apache-tomcat-camera-8.5.32-1150/bin/startup.sh
/data/devHome/protection-4.0/apache-tomcat-envir-8.5.32-1160/bin/startup.sh
/data/devHome/protection-4.0/apache-tomcat-monitor-8.5.32-1190/bin/startup.sh
/data/devHome/protection-4.0/apache-tomcat-video-8.5.32-1170/bin/startup.sh
```

#### 2、方式二

​        自己写脚本,配置各个服务自己开机启动

##### ①、nginx脚本

创建文件 nginx.service ,将下列脚本粘贴到该文本；(根据实际安装目录修改脚本中的路径)

```shell
[Unit]
Description=nginx service
After=network.target 
   
[Service] 
Type=forking 
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s quit
PrivateTmp=true 
   
[Install] 
WantedBy=multi-user.target
```

第一步：进入到/lib/systemd/system/目录

```sh
cd /lib/systemd/system/
```

第二步：上传nginx.service文件至该目录

第三步：加入开机自启动

```shell
systemctl enable nginx

#取消开机自启服务
systemctl disable nginx
```

第四步：服务的启动/停止/刷新配置文件/查看状态

```sh
systemctl start nginx.service　 启动nginx服务
systemctl stop nginx.service　 停止服务
systemctl restart nginx.service　 重新启动服务
systemctl list-units --type=service 查看所有已启动的服务
systemctl status nginx.service 查看服务当前状态
systemctl enable nginx.service 设置开机自启动
systemctl disable nginx.service 停止开机自启动
```

如果遇到错误
Warning: nginx.service changed on disk. Run 'systemctl daemon-reload' to reload units.
直接按照提示执行命令systemctl daemon-reload 即可。
 虚拟机重启后,nginx启动失败时,参考下面链接:
 https://blog.csdn.net/zyhlearnjava/article/details/71932719



##### ②、redis脚本

创建文件 redis ,将下列脚本粘贴到该文本；(根据实际安装目录修改脚本中的路径)

```shell
#!/bin/sh
#
# Simple Redis init.d script conceived to work on Linux systems
# as it does use of the /proc filesystem.

### BEGIN INIT INFO
# Provides:     redis_6379
# Default-Start:        2 3 4 5
# Default-Stop:         0 1 6
# Short-Description:    Redis data structure server
# Description:          Redis data structure server. See https://redis.io
### END INIT INFO

REDISPORT=6379
EXEC=/usr/local/bin/redis-server
CLIEXEC=/usr/local/bin/redis-cli

PIDFILE=/var/run/redis_${REDISPORT}.pid
CONF="/etc/redis/${REDISPORT}.conf"

case "$1" in
    start)
        if [ -f $PIDFILE ]
        then
                echo "$PIDFILE exists, process is already running or crashed"
        else
                echo "Starting Redis server..."
                $EXEC $CONF
        fi
        ;;
    stop)
        if [ ! -f $PIDFILE ]
        then
                echo "$PIDFILE does not exist, process is not running"
        else
                PID=$(cat $PIDFILE)
                echo "Stopping ..."
                $CLIEXEC -p $REDISPORT shutdown
                while [ -x /proc/${PID} ]
                do
                    echo "Waiting for Redis to shutdown ..."
                    sleep 1
                done
                echo "Redis stopped"
        fi
        ;;
    *)
        echo "Please use start or stop as first argument"
        ;;
esac

```

第一步：进入 /etc/init.d目录，将编辑好的 redis脚本上传至该目录下

第二步：设置权限

```sh
chmod 755 redis
```

第三步：启动测试

```sh
/etc/init.d/redis start
```

启动成功提示：
Starting Redis server...
Redis is running

第四步：设置开机自启动

```sh
chkconfig redis on
```



### 四、JDK的安装

1、查询自带的JDK版本

```sh
rpm -qa | grep jdk
rpm -qa | grep gcj
```

2、卸载系统自带的版本

```sh
# 卸载
yum -y remove copy-jdk-configs-2.2-5.el7_4.noarch
# 或者单个删除
rpm -e --nodeps
```

3、将安装包上传至服务器 （官方下载离线安装包，目前自用版本：jdk1.8.0_131)

4、解压

```sh
tar -zxvf jdk-8u131-linux-x64.tar.gz
```

我们要将解压后的【jdk1.8.0_131】里面的所有数据移动到我们需要安装的文件夹当中，

我们打算将jdk安装在usr/java当中，我们在usr目录下新建一个java文件夹

5、创建文件夹

```sh
mkdir /usr/java
```

6、将【jdk1.8.0_131】里的数据拷贝至java目录下 

```sh
mv /root/jdk1.8.0_131 /usr/java
```

7、修改环境变量

```sh
vim /etc/profile
```

在最下方粘贴：

```sh
export JAVA_HOME=/usr/java/jdk1.8.0_131 
export JRE_HOME=${JAVA_HOME}/jre 
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib:$CLASSPATH export JAVA_PATH=${JAVA_HOME}/bin:${JRE_HOME}/bin 
export PATH=$PATH:${JAVA_PATH}
```

8、通过命令使 profile 文件生效

```sh
source /etc/profile
```

9、测试是否安装成功

```
①、使用javac命令，不会出现command not found错误
②、使用java -version，出现版本为java version "1.8.0_131" 
③、echo $PATH，看看自己刚刚设置的的环境变量配置是否都正确
```



### 五、nginx的安装

1、下载nginx包

```sh
wget http://nginx.org/download/nginx-1.12.2.tar.gz
```

2、解压

```sh
tar -zxvf nginx-1.12.2.tar.gz
```

​	然后移动到目录

```sh
cd nginx-1.12.2
```

3、执行如下命令

```sh
./configure \
--prefix=/usr/local/nginx \
--pid-path=/var/run/nginx/nginx.pid \
--lock-path=/var/lock/nginx.lock \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--with-http_gzip_static_module \
--http-client-body-temp-path=/var/temp/nginx/client \
--http-proxy-temp-path=/var/temp/nginx/proxy \
--http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
--http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
--with-http_stub_status_module \
--with-http_ssl_module \
--http-scgi-temp-path=/var/temp/nginx/scgi

```

4、如果执行 make 命令如报错 

```
make: *** 没有规则可以创建“default”需要的目标“build”。 停止。
```

​	则安装Nginx相关依赖包：

```sh
yum -y install gcc openssl openssl-devel pcre-devel zlib zlib-devel
```

​	然后在执行如下命令：(注意登录账户权限，否则可能命令无法创建文件夹导致执行失败)

```sh
./configure --prefix=/usr/local/nginx --user=nginx --group=nginx --with-http_ssl_module
...
Configuration summary
+ using system PCRE library
+ using system OpenSSL library
+ using system zlib library

nginx path prefix: "/usr/local/nginx"
nginx binary file: "/usr/local/nginx/sbin/nginx"
nginx modules path: "/usr/local/nginx/modules"
nginx configuration prefix: "/usr/local/nginx/conf"
nginx configuration file: "/usr/local/nginx/conf/nginx.conf"
nginx pid file: "/usr/local/nginx/logs/nginx.pid"
nginx error log file: "/usr/local/nginx/logs/error.log"
nginx http access log file: "/usr/local/nginx/logs/access.log"
nginx http client request body temporary files: "client_body_temp"
nginx http proxy temporary files: "proxy_temp"
nginx http fastcgi temporary files: "fastcgi_temp"
nginx http uwsgi temporary files: "uwsgi_temp"
nginx http scgi temporary files: "scgi_temp"
```

5、编译安装

```sh
make && make install
```

6、移动至安装目录，并校验nginx是否可以正常；

```sh
cd /usr/local/nginx/sbin

# 校验nginx
 ./nginx -t 
```

​	如果报错：nginx: [error] invalid PID number "" in "/usr/local/nginx/logs/nginx.pid"
​	则执行如下命令：

```sh
# 返回至上一级目录
cd /usr/local/nginx
# 执行命令
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```

7、启动nginx 

```sh
# 移动至 sbin 目录
cd /usr/local/nginx/sbin 
# 启动nginx
./nginx -t 
./nginx -s reload
```



### 六、redis安装

1、下载

```sh
wget http://download.redis.io/releases/redis-5.0.7.tar.gz
```

2、解压

```sh
tar -zvxf redis-5.0.7.tar.gz
```

3、将redis移动到 /usr/local/目录下并更改文件名为 redis

```sh
mv /root/redis-5.0.7 /usr/local/redis
```

4、移动到/usr/local/redis目录，输入命令make执行编译命令

```sh
# 移动至目录
cd /usr/local/redis
# 编译
make
```

5、安装

```sh
make PREFIX=/usr/local/redis install
```

​	修改后台运行的配置

```sh
vim /usr/local/redis/redis.conf

将daemonize no改成yes
```

6、启动：在目录/usr/local/redis 输入下面命令启动redis

```sh
# 移动至目录
cd /usr/local/redis
# 启动redis
./bin/redis-server& ./redis.conf
```

7、查看进程确认安装正确

```sh
# 方式一
ps -aux | grep redis 
# 方式二
netstat -lanp | grep 6379
```



### 七、安装postgreSql-10

postgreSql 官方已经不再维护 centOS-6 操作系统

postgreSql-9.* 版本官方已经不再维护

​	

下面是 centOS-7 操作系统上进行安装 postgreSql-10

1、卸载之前的版本

```sh
yum remove postgresql*
```

2、下载安装包

```sh
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

3、安装 PostgreSQL

```sh
sudo yum install -y postgresql10-server
```

4、自定义数据存储路径(可跳过)

```sh
# 这时不急着初始化数据库，我们自定义systemd服务。
systemctl edit postgresql-10.service
 
# 写入以下内容：（看服务器实际情况，把数据存放在自定义目录，我把它先放在 /home/pgdata/）
# 更改为你自己的目录
[Service]
 Environment=PGDATA=/home/pgdata/10/data
  
#自定义配置将在：
/etc/systemd/system/postgresql-10.service.d/override.conf
#检查其内容
cat /etc/systemd/system/postgresql-10.service.d/override.conf
  
[Service]
 Environment=PGDATA=/home/pgdata/10/data
   
# 重新加载系统
systemctl daemon-reload
```

5、初始化数据库:(如果不需要自定义存储目录可以直接初始化)

```sh
sudo /usr/pgsql-10/bin/postgresql-10-setup initdb
```

6、开启开机自启服务

```sh
sudo systemctl enable postgresql-10
```

7、启动数据库服务

```sh
sudo systemctl start postgresql-10
```

8、创建 postgres 用户

```sh
# 创建用户
useradd postgres

passwd 123456

# 如果提示密码过于简单不支持
	# 切换至postgres用户
	su - postgres
	# 登录数据库
	psql -U postgres
	# 执行修改postgres用户密码
	ALTER USER postgres with encrypted password 'HelloWord@';
	
# 授予新用户权限
chown postgres:root -R /usr/pgsql-10/
```

9、如果需要远程访问数据库，需要修改配置文件

```sh
vim /var/lib/pgsql/10/data/pg_hba.conf

vim /var/lib/pgsql/10/data/postgresql.conf

# 如果修改了自定义数据存储路径的话:(路径根据自己的自定义路径而定  /data/pgdata/10/data/ 是 数据存储路径)
# 例如： vim /data/pgdata/10/data/pg_hba.conf
```



### 八、安装 postGIS-2.3 扩展库

```sh
# 移动至目录
cd /etc/yum.repos.d/
# 查询是否存在 epel-7.repo 如果没有则下载
wget https://mirrors.aliyun.com/repo/epel-7.repo
# 查看可以安装的包
yum search postgis
# 下载 postgis
yum install pgagent_11 postgis23_10 -y
# 登录数据库并创建扩展库
su - postgres
# 切换数据库
\c [数据库名称]
# 执行sql
create extension postgis;
```



------



## 系统操作

### 一、防火墙设置

1. CentOS 7

   因为CentOS7 里面是用 firewalld 来管理防火墙的
   命令语法：firewall-cmd [--zone=zone] 动作 [--permanent] 
   注：如果不指定--zone选项，则为当前所在的默认区域，--permanent选项为是否将改动写入到区域配置文件中

   例:

   ```sh
   添加80端口为允许：
   （--permanent 没有此参数重启后失效）
   #firewall-cmd --zone=public --add-port=443/tcp --permanent
   添加范围例外端口 如 5000-10000：
   #firewall-cmd --zone=public --add-port=1088-1120/tcp --permanent 
   添加完成后立刻生效：
   重新载入
   #firewall-cmd --reload
   查看
   #firewall-cmd --zone=public --query-port=443/tcp
   删除
   #firewall-cmd --zone=public --remove-port=80/tcp --permanent
   ```

    

2. CentOS 6

```sh
1. 开放端口命令： 
#/sbin/iptables -I INPUT -p tcp --dport 1209 -j ACCEPT
2.保存：
#/etc/rc.d/init.d/iptables save
3.重启服务：
#/etc/init.d/iptables restart
4.查看端口是否开放：
#/sbin/iptables -L -n
```



### 二、文件权限

```sh
# chmod u+x *.sh
```



### 三、查看服务器详情

```sh
1、查看系统运行情况
# top

2、查看系统内存
# free -h

3、查看硬盘使用情况
# df -h

4、查看文件夹大小
# ls-lh

5、查看CPU个数
# cat /proc/cpuinfo | grep "physical id" | uniq | wc -l

6、查看CPU核数
# cat /proc/cpuinfo | grep "cpu cores" | uniq

7、 查看内存总数
# cat /proc/meminfo | grep MemTotal
MemTotal: 32941268 kB //内存32G

8、 查看硬盘大小
# fdisk -l | grep Disk
Disk /dev/cciss/c0d0: 146.7 GB, 146778685440 bytes
总结：硬盘大小146.7G，即厂商标称的160G

9、 显示正在使用的内核版本 
# uname -r

10、显示哪些swap被使用
# cat /proc/swaps

11、显示系统日期
# date 

12、显示工作路径 
# pwd

13、显示文件和目录的详细资料 
# ls -l

14、显示隐藏文件 
# ls -a 
```



### 四、文件的搜索

```sh
1、从 '/' 开始进入根文件系统搜索文件和目录
# find / -name file1
2、在目录 '/ home/user1' 中搜索带有'.bin' 结尾的文件 
# find /home/user1 -name \*.bin 
```



### 五、挂载一个文件系统

```sh
1、挂载一个叫做hda2的盘 - 确定目录 '/ mnt/hda2' 已经存在 
# mount /dev/hda2 /mnt/hda2
2、卸载一个叫做hda2的盘 - 先从挂载点 '/ mnt/hda2' 退出 
# umount /dev/hda2
3、当设备繁忙时强制卸载 
# fuser -km /mnt/hda2
4、运行卸载操作而不写入 /etc/mtab 文件- 当文件为只读或当磁盘写满时非常有用 
# umount -n /mnt/hda2
5、挂载一个软盘 
# mount /dev/fd0 /mnt/floppy
6、挂载一个cdrom或dvdrom 
# mount /dev/cdrom /mnt/cdrom
7、挂载一个cdrw或dvdrom 
# mount /dev/hdc /mnt/cdrecorder
8、挂载一个cdrw或dvdrom 
# mount /dev/hdb /mnt/cdrecorder
9、挂载一个文件或ISO镜像文件 
# mount -o loop file.iso /mnt/cdrom
10、挂载一个Windows FAT32文件系统 
# mount -t vfat /dev/hda5 /mnt/hda5
11、挂载一个usb 捷盘或闪存设备 
# mount /dev/sda1 /mnt/usbdisk
12、挂载一个windows网络共享 
# mount -t smbfs -o username=user,password=pass //WinClient/share /mnt/share
```