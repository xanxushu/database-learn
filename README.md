## 10月9日-10月20日学习安排

# Docker

 ### *主要教程参考：[菜鸟教程](https://www.runoob.com/docker/docker-tutorial.html)*

 ### *Docker三个要点：*

 * 仓库（Repository）：仓库可看成一个代码控制中心，用来保存镜像。
 * 镜像（Image）：Docker 镜像（Image），就相当于是一个 root 文件系统。比如官方镜像 ubuntu:16.04 就包含了完整的一套 Ubuntu16.04 最小系统的 root 文件系统。
 * 容器（Container）：镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的类和实例一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。

 ### *主要命令集*

 #### 1. 镜像

   * `sudo docker pull [imageName]:[Tag]`   下载指定镜像到本地
   * `sudo docker images`  查看所有已下载的镜像
   * `sudo docker rmi [imageName]`  删除指定镜像
   * `sudo docker run -[option] [image] [command]` 将镜像实例化为容器

    例如：sudo docker run -itd centos:7 /bin/bash

   * `sudo docker commit -m="info" -a="user" [containerID] [new imageName]：[Tag]`  将某个容器提交为新镜像

    提交后的镜像可以通过 sudo docker images 来查看

   * `sudo docker tag [imageID] [imageName]:[Tag]` 给某个镜像打上标签

 #### 2. 容器

   * `sudo docker ps -a`  查看现有容器
   * `sudo docker start/stop/restart [containerID/Name]` 启动/停止/重启 一个容器
   * `sudo exec -[option] [containerID/Name] [command]` 进入一个已经创建的容器
   * `sudo docker rm -f [containerID]` 删除某个容器

 #### 3. DockerFile


<br>

# MySQL5.7(CentOS7)

### *MySQL安装（CentOS7中非默认目录，以/app/mysql目录为例）*

 #### 1. 下载MySQL5.7的安装包，[下载地址](https://dev.mysql.com/downloads/mysql/5.7.html)。

 #### 2. 打开安装包的下载目录，将安装包并解压到/app/mysql目录下。

   ```bash
tar -zxvf mysql-5.7.36-linux-glibc2.12-x86_64.tar.gz -C /app
cd /app
mv mysql-5.7.36-linux-glibc2.12 mysql
   ```

 #### 3. 检查系统是否已经安装过mysql或者mariadb，如果有的话，您需要卸载它们，并删除相关的文件和配置。

   ```bash
rpm -qa | grep mysql
yum -y remove mysql-libs.x86_64
whereis mysql
find / -name mysql
rm -rf /var/lib/mysql
rm /etc/my.cnf
rpm -qa | grep mariadb
rpm -e --nodeps mariadb-libs-5.5.60-1.el7-5.x86_64
   ```

 #### 4. 创建mysql用户组和用户，并赋予/app/mysql目录下的所有文件和文件夹相应的权限.

   ```bash
groupadd mysql
useradd -r -g mysql mysql
chown -R mysql:mysql /app/mysql
chmod -R 755 /app/mysql
   ```

 #### 5. 初始化mysql数据库，并记录生成的临时密码。

   ```bash
/app/mysql/bin/mysqld --initialize --user=mysql --datadir=/app/mysql/data --basedir=/app/mysql
#在运行这条命令时会在弹出的[Note]中显示临时密码，稍后登录会用到
   ```

 #### 6. 编写/etc/my.cnf配置文件，并添加相关的配置。

   ```bash
vi /etc/my.cnf #用vim当然更好
#按i进入插入模式
[mysqld]
datadir=/app/mysql/data
port=3306
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
symbolic-links=0
max_connections=400
innodb_file_per_table=1
lower_case_table_names=1 #表名大小写不敏感，敏感为0
   ```

 #### 7. 修改`/app/mysql/support-files/mysql.server`文件，将其中的`/usr/local/mysql`全部替换为`/app/mysql`。

 #### 8. 启动mysql服务，并使用临时密码登录mysql。

   ```bash
/app/mysql/support-files/mysql.server start
/app/mysql/bin/mysql -u root -p #输入临时密码
   ```

 #### 9. 修改root用户的密码，并开放远程连接。

   ```bash
set password for root@localhost = password('your PASSWARD'); #修改密码为你的密码
use mysql;
update user set user.Host='%' where user.User='root'; #开放远程连接
flush privileges;
   ```

 #### 10. 添加软连接，并重启mysql服务。

   ```bash
ln -s /app/mysql/support-files/mysql.server /etc/init.d/mysql
ln -s /app/mysql/bin/mysql /usr/bin/mysql 
service mysql restart
   ```

 #### 11. 开放3306端口，并测试本地客户端是否连接成功。

   ```bash
firewall-cmd --zone=public --add-port=3306/tcp --permanent #开放3306端口
firewall-cmd --reload #配置立即生效
mysql -u root -p #输入你设置的新密码
   ```

### *基于gtid的主从同步配置*

### *主要参考教程：[博客地址](https://cloud.tencent.com/developer/article/1817185)*

#### 1. 前提条件：MySQL版本大于5.6，并且两个MySQL版本相同或相近（5.7与8.0跨度太大，有很多因素会导致同步设置不成功如密码格式等）。了解MySQL主体安装路径。

#### 2. 编辑master（主服务器）配置文件。

  ```bash
vim /etc/my.cnf #按i进入插入模式
[client]
socket=/app/mysql/mysql.sock
[mysqld]
basedir=/app/mysql
datadir=/app/mysql/data
user=mysql
pid-file=/app/mysql/data/mysqld.pid
log-error=/app/mysql/data/mysql.err
socket=/app/mysql/mysql.sock
port=3306
server-id=1
gtid-mode=ON
enforce-gtid-consistency=ON
server-id=1
binlog_format=row
log-bin=/app/mysql/data/mysql-bin
#编辑到此为止
systemctl restart mysqld
#开放3306端口
firewall-cmd --add-port=3306/tcp --permanent
firewall-cmd --reload
  ```

#### 3. 编辑slave（从服务器）配置文件

  ```bash
vim /etc/my.cnf #按i进入插入模式
[client]
socket=/app/mysql/mysql.sock
[mysqld]
basedir=/app/mysql
datadir=/app/mysql/data
user=mysql
pid-file=/app/mysql/data/mysqld.pid
log-error=/app/mysql/data/mysql.err
socket=/app/mysql/mysql.sock
port=3306
server-id=2
gtid-mode=ON
enforce-gtid-consistency=ON
server-id=2
binlog_format=ROW
log-bin=/app/mysql/data/mysql-bin
log_slave_updates=ON
skip-slave-start=1
#编辑到此为止
systemctl restart mysqld
#开放3306端口
firewall-cmd --add-port=3306/tcp --permanent
firewall-cmd --reload
  ```

#### 4. 在master（主服务器）中创建用于主从复制的MySQL账号

  ```SQL
mysql -uroot -p #登录MySQL
mysql> grant replication slave on *.* to 'user'@'host' identified by 'password'; #创建账号并授予权限，建议主机为X.X.X.%类型
mysql> flush privileges; #更新权限
  ```

#### 5. 在slave（从服务器）中指定同步对象master

  ```SQL
mysql -uroot -p
mysql> change master to master_host='X.X.X.X', master_user='user',master_password='password',master_port=3306,master_auto_position=1; #指定master
mysql> start slave; #开启slave同步
  ```

#### 6. 查看slave状态（在从服务器的MySQL中）

  ```SQL
mysql> show slave status\G;
#在显示的信息中找到关键几条：
Slave_IO_State: Waiting for master to send event #必须是这句
Master_Host: X.X.X.X
Master_User: user
Master_Port: 3306
Slave_IO_Running: Yes #必须是yes
Slave_SQL_Running: Yes #必须是yes
Master_Server_Id: 1
Master_UUID: #一串很长的码，如果为空就是设置没成功
Retrieved_Gtid_Set: #一串很长的码
Executed_Gtid_Set: #一串很长的码
  ```

#### 7. 如果上述状态信息显示成功并且显示正确的话，可以开始验证

  ```SQL
  #在master中新建数据库、表、插入数据
  mysql> create database test1;
  mysql> use test1;
  mysql> create table testtb(id int primary key,name varchar(20);
  mysql> insert into testtb values (101,'test');

  #切换到slave中输入命令
  mysql> show databases; #显示的数据库中应包含刚刚在master中新建的test1
  mysql> use test1;
  mysql> show tables; #显示的结果中应包括刚刚在master创建的testtb
  mysql> select * from testtb; #应能查询到在master中插入的数据
  ```

#### 8. 如果不幸哪里出现了问题导致失败可以尝试重设master

  ```SQL
#在slave（从机）的MySQL中运行指令
mysql> stop slave;
mysql> reset slave;
#停止服务并且重置slave
#然后再次从第5步的代码开始执行
#mysql> change master to
  ```

### *基于xtrabackup工具的备份恢复*

### *主要参考教程：[博客1](https://blog.csdn.net/carefree2005/article/details/110170360)、[博客2](https://www.cnblogs.com/liantang-blog/p/16649240.html)*

#### *[xtrabackup下载地址（2.4版本centos7系统）](https://downloads.percona.com/downloads/Percona-XtraBackup-2.4/Percona-XtraBackup-2.4.28/binary/redhat/7/x86_64/percona-xtrabackup-24-2.4.28-1.el7.x86_64.rpm)*

#### 1. 下载软件包，wget或者直接从上面链接下载

  ```bash
wget https://downloads.percona.com/downloads/Percona-XtraBackup-2.4/Percona-XtraBackup-2.4.28/binary/redhat/7/x86_64/percona-xtrabackup-24-2.4.28-1.el7.x86_64.rpm
yum install -y percona-xtrabackup-24-2.4.28-1.el7.x86_64.rpm
  ```

#### 2. MySQL 准备，创建备份账户并授予权限

  ```SQL
mysql> create user backup@‘localhost’ identified by ‘password’;
mysql> grant reload,lock tables,replication client,CREATE TABLESPACE,PROCESS,SUPER,CREATE,INSERT,SELECT on . to ‘backup’@‘localhost’;
  ```

#### 3. 设置备份文件的目录（可以省略，直接在命令中指定）

  ```bash
  #方法一在MySQL配置文件中编辑
  vi /etc/my.cnf 
  [xtrabackup]
  target_dir = /app/backup/

  #方法二在命令中指定，见下面步骤
  ```

#### 4. 全量备份

  ```bash
  # 参数说明, innobackupex是xtrabackup的符号链接
  innobackupex --defaults-file=[mysql配置文件路径] \
  --datadir=[mysql数据文件夹路径] \
  --host=[mysql连接地址] \
  --port=[mysql端口] \
  --user=[mysql备份账号名称] \
  --password=[mysql备份账号密码] \
  --socket=[用于连接mysql的socket文件] \
  [指定备份存放路径]

  # 操作例子如下, 指定全量备份路径在 /app/backup/full
  innobackupex --defaults-file=/etc/my.cnf \
  --datadir='/app/mysql/data' \
  --host='X.X.X.X' \
  --port=3306 \
  --user='user' \
  --password='password' \
  --socket='/app/mysql/mysql.sock' \
  /app/backup/full

  # 备份好会出现 completed OK! 的字样, 按照例子出现的全量备份文件夹路径如下
  /backup/full/YEAR-MONTH-DATE_HOUR-MIN-SEC
  ```

#### 5. 增量备份（基于全量备份）

  ```bash
  # 参数说明
  innobackupex --defaults-file=[mysql配置文件路径] \
  --datadir=[mysql数据文件夹路径] \
  --host=[mysql连接地址] \
  --port=[mysql端口] \
  --user=[mysql备份账号名称] \
  --password=[mysql备份账号密码] \
  --socket=[用于连接mysql的socket文件] \
  --incremental-basedir=[指定前一次备份的目录] \
  --incremental=[指定增量备份存放的文件路径]

  # 操作例子如下
  innobackupex --defaults-file=/etc/my.cnf \
  --datadir='/app/mysql/data' \
  --host='X.X.X.X' \
  --port=3306 \
  --user='user' \
  --password='password' \
  --socket='/app/mysql/mysql.sock' \
  --incremental-basedir=/app/backup/full/YEAR-MONTH-DATE_HOUR-MIN-SEC \
  --incremental /app/backup/inc 

  # 备份好会出现 completed OK! 的字样, 按照例子出现的第一次增量备份文件夹路径如下
  /app/backup/inc/YEAR-MONTH-DATE_HOUR-MIN-SEC
  

  #增量备份也可以基于增量备份
  innobackupex --defaults-file=/etc/my.cnf \
  --datadir='/app/mysql/data' \
  --host='X.X.X.X' \
  --port=3306 \
  --user='user' \
  --password='password' \
  --socket='/app/mysql/mysql.sock' \
  --incremental-basedir=/app/backup/inc/YEAR-MONTH-DATE_HOUR-MIN-SEC \
  --incremental /app/backup/inc 

  # 备份好会出现 completed OK! 的字样, 按照例子出现的第一次增量备份文件夹路径如下
  /app/backup/inc/YEAR-MONTH-DATE_HOUR-MIN-SEC
  ```

<br>

# Zookeeper 的安装与连接（CentOS7）

### *参考教程：[菜鸟教程](https://www.runoob.com/w3cnote/zookeeper-tutorial.html)、[博客地址](https://blog.csdn.net/A_Little_Fish_/article/details/117384614#:~:text=%E4%B8%80%E3%80%81%E8%AF%BE%E5%89%8D%E5%87%86%E5%A4%87%E5%87%86%E5%A4%87%E4%B8%80%E5%8F%B0%E5%86%85%E5%AD%98%E6%9C%80%E5%B0%918G%EF%BC%88%E5%BB%BA%E8%AE%AE16G%EF%BC%89%E3%80%81cpu,i7%204%E6%A0%B8%E7%9A%84%E7%94%B5%E8%84%91%E6%90%AD%E5%BB%BA%E5%A5%BD%E4%B8%89%E5%8F%B0%E8%99%9A%E6%8B%9F%E6%9C%BA%EF%BC%8C%E5%88%86%E5%88%AB%E6%98%AFnode01%2Cnode02%2Cnode03%E4%BA%8C%E3%80%81%E8%AF%BE%E5%A0%82%E4%B8%BB%E9%A2%98%E6%90%AD%E5%BB%BA3%E8%8A%82%E7%82%B9zookeeper%E9%9B%86%E7%BE%A4%E4%B8%89%E3%80%81%E8%AF%BE%E5%A0%82%E7%9B%AE%E6%A0%87%E5%AE%8C%E6%88%90zookeeper%E9%9B%86%E7%BE%A4%E5%AE%89%E8%A3%85%E5%9B%9B%E3%80%81%E5%AE%89%E8%A3%85%E6%AD%A5%E9%AA%A4%E4%B8%8B%E8%BD%BDzookeerper%E5%8E%8B%E7%BC%A9%E5%8C%85%EF%BC%8C%E5%B9%B6%E4%B8%94%E4%B8%8A%E4%BC%A0%E5%88%B0%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%AC%AC%E4%B8%80%E5%8F%B0%E6%9C%BA%E5%99%A8%E8%8A%82%E7%82%B9%E3%80%82)*

#### *[Zookeeper下载地址](https://zookeeper.apache.org/releases.html),选择适合的版本下载。*

### ***前提：Zookeeper依赖于Java，必须保证系统中存在Java的环境变量，可以使用 `java -version` 来查看***

#### 1. 准备3个设备进行搭建，每个设备都执行下面的步骤。下载安装包，使用wget或者直接从上面的下载网址下载即可，但是需要注意一点，安装包要下载带***bin***字样的版本，才能够正确使用。

  ```bash
wget https://dlcdn.apache.org/zookeeper/zookeeper-3.8.3/apache-zookeeper-3.8.3-bin.tar.gz
tar -zxvf apache-zookeeper-3.8.3-bin.tar.gz -C /app #将软件解压到指定目录
cd /app
mv apache-zookeeper-3.8.3-bin zookeeper #重命名
  ```

#### 2. 整理文件

  ```bash
#刚解压完的Zookeeper不能直接运行需要整理一些文件
cd /app/zookeeper
mkdir data #新建data文件夹备用
cd conf
mv zoo_sample.cfg zoo.cfg #将配置文件重命名，这一步不可省略，只有这个名字才能被读取
  ```

#### 3. 修改和添加配置

  ```bash
vi zoo.cfg #当然用vim更好
  ```

  ![alt zoo.cfg1](/source/zoo.cfg1.png)

  ```bash
#找到如上图所示的dataDir并修改为刚刚新建的data目录的地址
dataDir=/app/zookeeper/data
  ```

  ![alt zoo.cfg2](/source/zoo.cfg2.png)

  ```bash
#再将上图中参数前的#去掉，如下所示
autopurge.snapRetainCount=3
autopurge.purgeInterval=1

#然后在文件末尾加上如下代码
server.1=X.X.X.X:2888:3888
server.2=X.X.X.X:2888:3888
server.3=X.X.X.X:2888:3888
#其中X代表三个节点的IP地址
  ```

  ```bash
  #在data文件夹中新建文件myid
  cd /app/zookeeper/data

  #下面的步骤三台设备分别执行，即第一台机myid中写入1，第二台机myid中写入2……以此类推
  #第一台
  echo 1 > myid
  #第二台
  echo 2 > myid
  #第三台
  echo 3 > myid
  #可以使用cat查看是否写入成功
  cat myid
  ```

#### 4. 开启服务并验证

  ```bash
cd /app/zookeeper/bin
#三台机全部开启
./zkServer.sh start #开启成功的话最后会有STARTED字样提示
#验证并查看状态
./zkServer.sh status #成功最后一行会提示Mode:leader/follower
  ```

#### 5. 使用结束后关闭集群

  ```bash
./zkServer.sh stop #三台机全部执行，成功会有STOPPED字样提示
  ```

<span style="color:red;background-color:yellow">
！！！注意：当集群中存在虚拟机时关闭顺序应为：
</span>

<span style="color:red;background-color:yellow;">1. 使用上面命令关闭三台机集群服务
</span>

<span style="color:red;background-color:yellow">2. 关闭虚拟机
</span>

<span style="color:red;background-color:yellow">3. 关闭主机
</span>

<br>
<br>

# PostgreSQL基本连接

#### [PostgreSQL下载链接](https://www.postgresql.org/download/)，选择适合的版本进行下载。

#### 1. 下载和安装PostgreSQL，本例基于CentOS7，到上面的下载界面选择Linux系统，CentOS。

![alt PSQLDL](/source/PSQLDL.png)

#### 2. 在下一个页面中选择适合你的版本、机型、架构，就会弹出下载指令

![Alt Download](/source/download.png)

#### 3. 直接在终端执行提示的指令，即可下载并开启服务。

  ```bash
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo yum install -y postgresql16-server
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl enable postgresql-16
sudo systemctl start postgresql-16
  ```

#### 4. 登录并修改密码。

  ```bash
su postgres #会进入一个新的命令行
psql #进入应用
  ```

  ```SQL
# 设置密码
alter user postgres with password 'password';
  ```

#### 5. 回到原终端命令行，修改配置文件。

  ```bash
vim /var/lib/pgsql/16/data/postgresql.conf
#修改参数:
listen_addresses = '*'
# 编辑配置
vim /var/lib/pgsql/16/data/pg_hba.conf
# 添加内容
host    all      all    0.0.0.0/0      md5
#重启服务
sudo systemctl restart postgresql-16
  ```

#### 6. 在原终端中直接登录

  ```bash
#开启5432端口
firewall-cmd --add-port=5432/tcp --permanent --zone=public
#重启防火墙
firewall-cmd --reload
#格式
psql -h 主机IP -p 端口  -U 用户名 -W -d 数据库
#示例
psql -h 127.0.0.1 -p 5432  -U postgres -d postgres
#如果提示无此数据库则需要使用上面的方法登录并且新建一个你想登录的数据库，5432为psql的默认端口
  ```

#### 7. 使用DBeaver连接数据库

* DBeaver[下载地址](https://dbeaver.io/download/)，选择合适的版本进行下载安装。
* 打开软件，在菜单栏中选中数据库/新建数据库链接，在弹出的窗口中找到PostgreSQL，点击下一步。
* 在连接设置中填写主机、端口、数据库、用户名、密码，其余均默认即可。
* 点击测试连接，如果你没有下载该数据库的驱动的话，可能会提示你下载驱动，<span style="color:red">注意，下载驱动时要选择最新版驱动，否则有可能报错。有的时候默认选择的并不是最新版，需要你手动调整，点击驱动版本号即可选择版本。</span>
* 如果测试连接成功，点击完成即可；如果失败，则根据报错提示进行调整。成功实例如下图所示：
  ![alt dbea](/source/dbea.png)

# keepalived绑定vip

### 1. keepalived下载与编译。

* [keepalived下载地址](https://www.keepalived.org/download.html) 选择合适的版本进行下载

* 进入下载好的文件所在位置

  ```bash
  tar -zxvf keepalived-2.1.0.tar.gz
  cd keepalived-2.1.0
  ./configure --prefix=/usr/local/keepalived
  #prefix后地址可自定义，但最好还是这个路径，不太会出错
  #如果上面的命令运行后报错提示有关opensll的错误，可以使用下面的命令进行下载
  yum install openssl openssl-devel
  #安装成功后继续执行上面的./configure 命令
  make && make install
  ```

### 2. 修改与配置

* 编译成功后将会有文件出现在`/usr/local/keepalived`目录中

  ```bash
  cd /usr/local/keepalived
  ls
  #应有etc目录，有的话进行下一步
  mkdir /etc/keepalived
  cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf
  ```

* 接下来编辑配置文件`keepalived.conf`

  ```bash
  vi /etc/keepalived/keepalived.conf #vim更好
  #在默认的模版中我们需要修改几处即可
  ! Configuration File for keepalived
  
  global_defs {
    notification_email {
      acassen@firewall.loc
      failover@firewall.loc
      sysadmin@firewall.loc
    }
    notification_email_from Alexandre.Cassen@firewall.loc
    smtp_server 192.168.200.1
    smtp_connect_timeout 30
    router_id LVS_DEVEL
    vrrp_skip_check_adv_addr
  # vrrp_strict  #这一行如果需要vip飘逸的话需要注释掉
    vrrp_garp_interval 0
    vrrp_gna_interval 0
  }
  
  vrrp_instance VI_1 {
    state MASTER      #MASTER/BACKUP 两个不同模式主/备
    interface ens33   #此处应为网卡名，以centos7为例即为ens33
    unicast_src_ip 192.168.68.130 #本机实际IP地址
    unicast_peer {
        192.168.68.129  #需要飘逸的另一个主机实际IP地址
    }
    virtual_router_id 51 #两台机需要一致
    priority 100    #在MASTER的值要大于BACKUP，例如M100，B50，代表优先级
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.200.16   #vip地址
    }
  }
  
  virtual_server 192.168.200.16 80 { #vip地址/端口
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP
  
    real_server 192.168.68.130 80 { #实际IP地址1/端口
        weight 1  #权重
        SSL_GET {  #以下是默认的我没改过
            url {
              path /
              digest ff20ad2481f97b1754ef3e12ecd3a9cc
            }
            url {
              path /mrtg/
              digest 9b3a0c85a887a256d6939da88aabd8cd
            }
            connect_timeout 3
            retry 3
            delay_before_retry 3
        }
    }
  
       real_server 192.168.68.129 80 { #实际IP地址2/端口
        weight 1
        SSL_GET {
            url {
              path /
              digest ff20ad2481f97b1754ef3e12ecd3a9cc
            }
            url {
              path /mrtg/
              digest 9b3a0c85a887a256d6939da88aabd8cd
            }
            connect_timeout 3
            retry 3
            delay_before_retry 3
        }
    }
  }
  ```

### 3. 使用与检验

* 启动服务

  ```bash
  #开启服务
  systemctl start keepalived.service
  #查看状态
  systemctl status keepalived.service -l
  #如果启动失败可以查看日志看看哪里出错了
  journalctl -u keepalived
  #如果上面两步都没问题则可以查看vip有没有出现
  ip addr
  #如果vip出现了则说明配置成功
  ```

* vip飘逸的检测

  ```bash
  #在MASTER启动服务后，在BACKUP也启动服务
  #在BACKUP机中运行下面的命令
  systemctl start keepalived.service
  systemctl status keepalived.service
  #确认服务启动后查看IP地址
  ip addr
  #此时，列出的信息中不应出现vip，如果有则说明主备都出现vip
  #如果无vip，进行下面的检查
  #回到MASTER机，运行下面的命令
  systemctl stop keepalived.service
  systemctl status keepalived.service
  #确认服务已经停止
  ip addr
  #确认vip也已经消失
  #再次回到BACKUP机，运行下面命令
  ip addr
  #此时，vip应该成功出现，如果没有可以重启服务试试
  systemctl restart keepalived.service
  ```

* 可以多重复几次上面步骤来确定vip飘逸的丝滑，至此keepalived配置vip结束。
