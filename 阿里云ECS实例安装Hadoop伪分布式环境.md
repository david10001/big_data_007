## 阿里云ECS实例安装Hadoop伪分布式环境  
操作步骤：
*  阿里云购买机器，可参照博客
*  安装jdk
*  安装ssh
*  安装hadoop
*  启动dfs和yarn


* ### 初始化

#### 添加hadoop账号并给用户赋sudo权限
本实例主机名为hadoop,添加用户hadoop，并赋sudo权限
>sudo是Linux系统管理指令，是允许系统管理员让普通用户执行一些或者全部root命令的一个工具。Linux系统下，为了安全，一般来说我们操作都是在普通用户下操作，但是有时候普通用户需要使用root权限，比如在安装软件的时候。这个时候如果我们切回root用户下效率就会比较低，所以用sudo命令就会很方便。

root权限下操作  
```shell
[root@hadoop ~]# useradd hadoop
[root@hadoop ~]# vim /etc/sudoers

root    ALL=(ALL)       ALL
hadoop    ALL=(ALL)     NOPASSWD:ALL
```
:qw!强制保存退出   
chown -R hadoop:hadoop /home/hadoop/*

#### 配置hosts
切换到hadoop账号下 su - hadoop  
注意：该命令在执行的时候，会同时执行source ~/.bash_profile，而su hadoop不会
```shell
[hadoop@hadoop ~]$ vi  /etc/hosts

127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.18.24.162 hadoop
```
:wq保存退出  
>注意：  
1）hosts里面配置的是内网ip，即私有ip，配置过程中所有的ip都是私有ip。  
2）前两行不要删除，不然后续可能出错。

* ### 配置ssh

#### 为什么要配置ssh
如果不配置，每次在启动dfs等进程的时候就需要输入密码，并且如果hadoop用户没有单独设置密码的话，输入什么都是错的，如同下面这样
```ssh
[hadoop@hadoop ~]$ ssh localhost
The authenticity of host 'localhost (127.0.0.1)' can't be established.
ECDSA key fingerprint is 5a:47:ee:a8:88:9d:be:64:d7:25:01:22:7c:b2:b1:82.
Are you sure you want to continue connecting (yes/no)?yes
hadoop@localhost's password:
Permission denied, please try again.
hadoop@localhost's password:
Permission denied, please try again.
```
#### 检测并安装ssh
```shell
[root@hadoop ~]# which ssh
/usr/bin/ssh
[root@hadoop ~]# yum install ssh
```
#### 生成公钥和私钥
```shell
ssh-keygen -t rsa
```  
连续三次Enter，公钥和私钥保存在 ~/.ssh中，ll -a 命令可以查看隐藏的文件夹
#### 复制公钥
```shell
hadoop@sf ~]$ cd .ssh
[hadoop@sf .ssh]$ ll
-rw-------. 1 hadoop hadoop 1675 5月 31 22:38 id_rsa
-rw-r--r--. 1 hadoop hadoop 391 5月 31 22:38 id_rsa.pub
[hadoop@hadoop .ssh]$ pwd
/home/hadoop/.ssh
[hadoop@hadoop .ssh]$ cp id_rsa.pub authorized_keys
```
#### 检测ssh配置是否生效
```shell
[hadoop@hadoop .ssh]$ ssh localhost date
Mon Sep  2 20:57:12 CST 2019
[hadoop@hadoop app]$
```
* ### 安装jdk

#### 新建目录
```ssh
[hadoop@hadoop ~]$ pwd
/home/hadoop
[hadoop@hadoop ~]$ mkdir app data software tmp script
[hadoop@hadoop ~]$ ls
app  data  script  software  tmp
```
#### 官网下载jdk
这里使用版本是 jdk-8u45-linux-x64.gz  
下载过程，略
#### 文件传输工具
常用的有fileZila xftp等，这里使用rz命令工具  
检测rz是否存在 which rz  
yum在线安装,yum install -y lrzsz，hadoop没有权限,没有权限可以加sudo命令执行，即sudo yum install -y lrzsz，或者exit命令退回root权限下安装
#### 解压缩
tar -zxvf jdk-8u131-linux-x64.tar.gz
mv jdk1.8.0_131 /home/hadoop/app/  
>注意坑：如果采用该种方式解压到某个目录下，如下
tar -zxvf /home/hadoop/software/jdk-8u45-linux-x64.gz -C  /home/hadoop/app
由于解压使用的是临时用户组，所以要更改用户和用户组 ，需要重新对java目录赋600权限，chown -R hadoop:hadoop /usr/java/*

#### 配置jdk环境变量
ls -n  jdk1.8.0_45 jdk
建立软连接，如果jdk版本号变动，只需要更软链接对应的版本路径名，而相关配置中的内容不需要变动。
```ssh
[hadoop@hadoop app]$ ln -s jdk1.8.0_45/ jdk
[hadoop@hadoop app]$ vim ~/.bash_profile

export JAVA_HOME=/home/hadoop/app/jdk1.8.0_45  
export PATH=$JAVA_HOME/bin:$PATH
```
>这两行删掉。  
PATH=$PATH:$HOME/.local/bin:$HOME/bin  
export PATH
:wq保存  
注意：这里使用的是用户的环境变量配置，在机器重启的时候会失效，执行su - hadoop切换用户的时候默认执行了 source生效命令  
#### source生效并检查

```shell
[hadoop@hadoop app]$ source ~/.bash_profile
[hadoop@hadoop app]$ java -version
java version "1.8.0_45"
Java(TM) SE Runtime Environment (build 1.8.0_45-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.45-b02, mixed mode)
```
* ### 安装hadoop

#### 版本介绍
这里使用的是编译后的cdh版本，hadoop-2.6.0-cdh5.15.1，该版本和cloudera官网下载的cdh版本的区别在于，编译后的版本支持文件压缩格式
安装成功后可以执行hadoop checknative查看

#### 安装hadoop和配置环境变量
```shell
[hadoop@hadoop software]$ tar -zxvf hadoop-2.6.0-cdh5.15.1.tar.gz
[hadoop@hadoop software]$ mv hadoop-2.6.0-cdh5.15.1 ../app
[hadoop@hadoop software]$ ln -s /home/hadoop/app/hadoop-2.6.0-cdh5.15.1 hadoop
[hadoop@hadoop ~]$ vim ~/.bash_profile
export JAVA_HOME=/home/hadoop/app/jdk
export HADOOP_HOME=/home/hadoop/app/hadoop
export PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
[hadoop@hadoop ~]$ source ~/.bash_profile
```
#### 配置文件介绍
安装hadoop环境需要配置6个主要文件,都在该目录下


|序号|文件名|配置对象|主要内容|
|-|-|-|-|
|1|	hadoop-env.sh|	hadoop运行环境|用来定义hadoop运行环境相关的配置信息
|2|	core-site.xml|	集群全局参数|	用于定义系统级别的参数，如HDFS URL 、Hadoop的临时目录等
|3|	hdfs-site.xml	|HDFS存储|	如名称节点和数据节点的存放位置、文件副本的个数、文件的读取权限等
|4|	mapred-site.xml|	MapReduce参数|	包括JobHistory Server 和应用程序参数两部分，如reduce任务的默认个数、任务所能够使用内存的默认上下限等
|5|	yarn-site.xml|	集群资源管理系统参数|	配置ResourceManager ，nodeManager的通信端口，web监控端口等
|6|slaves|hostname|主从节点机器的hostname|

#### 配置hadoop-env.sh文件
这里由于liunx一些本身的原因，里面的JAVA_HOME必须修改，把java路径写死，不然在后续启动dfs的时候会报错找不到JAVA_HOME路径的错误
```shell
vim hadoop-env.sh

#export JAVA_HOME=${JAVA_HOME}
export JAVA_HOME=/home/hadoop/app/jdk1.8.0_45
```
#### 配置core-site.xml文件
修改dfs默认访问的路径端口和namenode datanode的前缀存储路径
```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop:8020</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/home/hadoop/tmp/hadoop</value>
    </property>
</configuration>
```
#### 配置hdfs-site.xml文件
vi hdfs-site.xml
配置副本数、第二节点的访问端口，注意这里没有修改hdfs的web的50070端口
```xml
<configuration>
  <property>
        <name>dfs.replication</name>
        <value>1</value>
        <discription>hdfs的副本数，伪分布式设置为1</discription>
  </property>
</configuration>
```
这里只需要设置副本数就可以了，其他的如nn snn等如果端口号冲突了也可以修改,如下都是官方默认的
```xml
<property>
     <name>dfs.namenode.http-address</name>
     <value>hadoop:50070</value>
     <discription>hdfs nn web的ip和端口，默认50070</discription>
</property>
<property>
     <name>dfs.namenode.secondary.http-address</name>
     <value>hadoop:50090</value>
     <discription>hdfs snn web的ip和端口，默认50090</discription>
</property>
<property>
     <name>dfs.namenode.secondary.https-address</name>
     <value>hadoop:50091</value>
 </property>
```
#### 配置mapred-site.xml文件
cp mapred-site.xml.template mapred-site.xml  
vim mapred-site.xml    
```xml
<configuration>
  <property>
  	<name>mapreduce.framework.name</name>
  	<value>yarn</value>
  </property>
</configuration>
```
#### 配置yarn-site.xml文件
>hadoop-2.0.3-alpha 的配置中，yarn.nodemanager.aux-services项的默认值是“mapreduce.shuffle”，但如果在hadoop-2.2 中继续使用这个值就会报错，需要设置为mapreduce_shuffle

vim yarn-site.xml
```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```
>如果yarn的8088端口被占用，可以再yarn-site.xml中修改
```xml
<property>
    <name>yarn.resourcemanager.webapp.address</name>
    <value>hadoop:8089</value>
</property>
```

#### 配置slaves文件
```shell
echo hadoop > slaves
```
* ### 启动hdfs和yarn

#### 格式化namenode
```shell
[hadoop@hadoop ~]$ hdfs namenode -format
```  
打印日志中出现success即表示格式化成功。  
/home/hadoop/tmp/hadoop/dfs/name has been successfully formatted
#### 启动dfs
hadoop中的一些sh文件的启动都在sbin中，因此在~/.bash_profile中也需要配置sbin环境变量，这样在任何地方可以直接启动dfs和yarn，然后jpd查看启动是否成功
```shell
[hadoop@hadoop ~]$ start-dfs.sh
[hadoop@hadoop ~]$ jps
19264 Jps
19155 SecondaryNameNode
18884 NameNode
19004 DataNode
[hadoop@hadoop ~]$ start-yarn.sh
[hadoop@hadoop ~]$ jps
19155 SecondaryNameNode
18884 NameNode
19701 Jps
19402 NodeManager
19004 DataNode
19308 ResourceManager
```
其中NodeManager和ResourceManager是yarn的进程，其他的是hdfs的进程。
### 页面访问
设置阿里云安全组，限定只能是家里网络IP和端口避免被挖矿
查看网络ip
百度输入ip即可，183.17.232.77，变动的
hdfs的web端口没有修改默认即为50070  
yarn的web端口没有修改默认为8088  
都需要添加安全组规则  
8088/8088 网络ip  
50070/50070 网络ip
8020/8020 0.0.0.0/0 ???

查看hdfs的端口 cat ~/app/hadoop/etc/hadoop/hdfs-site.xml,没有设置即为默认  
实例公网ip访问hdfs和yarn的web界面
http://120.24.194.145:50070
查看yarn的端口 cat ~/app/hadoop/etc/hadoop/yarn-site.xml,没有设置即为默认
http://120.24.194.145:8089

J哥操作记录
修改几个配置文件
root账号下赋权限
chown -R hadoop:hadoop /home/hadoop/*
>chown将指定文件的拥有者改为指定的用户或组，用户可以是用户名或者用户ID；组可以是组名或者组ID；文件是以空格分开的要改变权限的文件列表，支持通配符。系统管理员经常使用chown命令，在将文件拷贝到另一个用户的名录下之后，让用户拥有使用该文件的权限。  

su - hadoop

查看hadoop进程 /home/hadoop/tmp路径下 root权限
ps -ef|grep hadoop\
kill -9 $(pgrep -f hadoop)

删除配置的dfs的namenode和datanode，因为在core-site.xml里面已经配置了tmp路径了，
namenode和datanode初始化的时候会在这个路径下的，官方配置参数文档可以查看到
查看hostname cat /etc/hostname
检查ssh,hadoop账号，ssh localhost date
hdfs namenode -format
sbin/start-dfs.sh
sbin/start-yarn.sh
jps
4897 Jps
4370 SecondaryNameNode
4099 NameNode
4219 DataNode
4605 NodeManager
4511 ResourceManager  
netstat -nlp |grep 4511
hadoop  hadoop checknative
