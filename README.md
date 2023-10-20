## 10月9日-10月20日学习安排

# Docker
 ### *主要教程参考：<a herf=https://www.runoob.com/docker/docker-tutorial.html>菜鸟教程</a>*

 ### *Docker三个要点：*
 * 仓库（Repository）：仓库可看成一个代码控制中心，用来保存镜像。
 * 镜像（Image）：Docker 镜像（Image），就相当于是一个 root 文件系统。比如官方镜像 ubuntu:16.04 就包含了完整的一套 Ubuntu16.04 最小系统的 root 文件系统。
 * 容器（Container）：镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的类和实例一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。

 ### *主要命令集*
 1. 镜像
   * `sudo docker pull [imageName]:[Tag]`   下载指定镜像到本地
   * `sudo docker images`  查看所有已下载的镜像
   * `sudo docker rmi [imageName]`  删除指定镜像
   * `sudo docker run -[option] [image] [command]` 将镜像实例化为容器

    例如：sudo docker run -itd centos:7 /bin/bash
   * `sudo docker commit -m="info" -a="user" [containerID] [new imageName]：[Tag]`  将某个容器提交为新镜像

    提交后的镜像可以通过 sudo docker images 来查看
   * `sudo docker tag [imageID] [imageName]:[Tag]` 给某个镜像打上标签
 2. 容器
   * `sudo docker ps -a`  查看现有容器
   * `sudo docker start/stop/restart [containerID/Name]` 启动/停止/重启 一个容器
   * `sudo exec -[option] [containerID/Name] [command]` 进入一个已经创建的容器
   * `sudo docker rm -f [containerID]` 删除某个容器
 3. DockerFile
   

<br>

# MySQL5.7(CentOS7)

### *MySQL安装（CentOS7中非默认目录，以/app/mysql目录为例）*
 1. 下载MySQL5.7的安装包，[下载地址](https://dev.mysql.com/downloads/mysql/5.7.html)。
 2. 打开安装包的下载目录，将安装包并解压到/app/mysql目录下。
   
   ```bash
   tar -zxvf mysql-5.7.36-linux-glibc2.12-x86_64.tar.gz -C /app
   cd /app
   mv mysql-5.7.36-linux-glibc2.12 mysql
   ```
 3. 检查系统是否已经安装过mysql或者mariadb，如果有的话，您需要卸载它们，并删除相关的文件和配置。
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
 4. 创建mysql用户组和用户，并赋予/app/mysql目录下的所有文件和文件夹相应的权限.
   ```bash
   groupadd mysql
   useradd -r -g mysql mysql
   chown -R mysql:mysql /app/mysql
   chmod -R 755 /app/mysql
   ```
 5. 初始化mysql数据库，并记录生成的临时密码。
   ```bash
   /app/mysql/bin/mysqld --initialize --user=mysql --datadir=/app/mysql/data --basedir=/app/mysql
   #在运行这条命令时会在弹出的[Note]中显示临时密码，稍后登录会用到
   ```
 6. 编写/etc/my.cnf配置文件，并添加相关的配置。
   ```bash
   vi /etc/my.cnf #用vim当然更好
   #按i进入插入模式
   [mysqld]
   datadir=/app/mysql/data
   port = 3306
   sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
   symbolic-links=0
   max_connections=400
   innodb_file_per_table=1
   lower_case_table_names=1 #表名大小写不敏感，敏感为0
   ```
 7. 修改`/app/mysql/support-files/mysql.server`文件，将其中的`/usr/local/mysql`全部替换为`/app/mysql`。
 8. 启动mysql服务，并使用临时密码登录mysql。
   ```bash
   /app/mysql/support-files/mysql.server start
   /app/mysql/bin/mysql -u root -p #输入临时密码
   ```
 9. 修改root用户的密码，并开放远程连接。
   ```bash
   set password for root@localhost = password('your PASSWARD'); #修改密码为你的密码
   use mysql;
   update user set user.Host='%' where user.User='root'; #开放远程连接
   flush privileges;
   ```
 10. 添加软连接，并重启mysql服务。
   ```bash
   ln -s /app/mysql/support-files/mysql.server /etc/init.d/mysql
   ln -s /app/mysql/bin/mysql /usr/bin/mysql 
   service mysql restart
   ```
 11. 开放3306端口，并测试本地客户端是否连接成功。
   ```bash
   firewall-cmd --zone=public --add-port=3306/tcp --permanent #开放3306端口
   firewall-cmd --reload #配置立即生效
   mysql -u root -p #输入你设置的新密码
   ```

### *基于gtid的主从同步配置*

### *主要参考教程：[博客地址](https://cloud.tencent.com/developer/article/1817185)*

1. 前提条件：MySQL版本大于5.6，并且两个MySQL版本相同或相近（5.7与8.0跨度太大，有很多因素会导致同步设置不成功如密码格式等）。了解MySQL主体安装路径。
2. 编辑master（主服务器）配置文件。
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
3. 编辑slave（从服务器）配置文件
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
4. 在master（主服务器）中创建用于主从复制的MySQL账号
  ```bash
  mysql -uroot -p #登录MySQL
  mysql> grant replication slave on *.* to 'user'@'host' identified by 'password'; #创建账号并授予权限，建议主机为X.X.X.%类型
  mysql> flush privileges; #更新权限
  ```
5. 在slave（从服务器）中指定同步对象master
  ```bash
  mysql -uroot -p
  mysql> change master to master_host='X.X.X.X', master_user='user',master_password='password',master_port=3306,master_auto_position=1; #指定master
  mysql> start slave; #开启slave同步
  ```
6. 查看slave状态（在从服务器的MySQL中）
  ```bash
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
7. 如果上述状态信息显示成功并且显示正确的话，可以开始验证
  ```bash
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
8. 如果不幸哪里出现了问题导致失败可以尝试重设master
  ```bash
  #在slave（从机）的MySQL中运行指令
  mysql> stop slave;
  mysql> reset slave;
  #停止服务并且重置slave
  #然后再次从第5步的代码开始执行
  #mysql> change master to
  ```
### *基于xtrabackup工具的备份恢复*
