1.安装jdk1.7以上

2.下载zookeeper-3.4.10.tar.gz  [http://mirrors.hust.edu.cn/apache/zookeeper/](http://mirrors.hust.edu.cn/apache/zookeeper/)

3.zk文件夹下面建立servers，data，logs三个文件夹

![1](../1.png)
4.每个文件夹下面简历三个子文件夹分别为s1,s2,s3

5.在/data/s1/下简历myid文件，内容为1，s2,s3下面的myid分别为2,3
![1](../2.png)

5.tar -xzvf zookeeper-3.4.8.tar.gz解压，然后复制zk的conf目录下的zoo_sample_cfg复制一份为到/servers/s1/zoo.cfg，并修改配置文件内容
vi ./zoo.cfg
dataDir=/data/home/zookeeper/data   
dataLogDir=/data/home/zookeeper/log   
clientPort=2181
server.1=host1:2886:3886
server.2=host2:2887:3887
server.3=host3:2888:3888 
host1为ip，如果是在同一台机子上面，配置端口需要递增
clientPort=2182
server.1=host1:2886:3886
server.2=host2:2887:3887
server.3=host3:2888:3888 



**dataDir** zk 的放置一下数据的目录如版本号，id 等等，所以每一个节点都区分一下如，/data/s1;

**clientPort** 接受客户端请求的端口，每个节点都需要不一样。如：2181

**server.X** 这个数字就是对应 data/myid中的数字。你在3个server的myid文件中分别写入了1，2，3，那么每个server中的zoo.cfg都配server.1,server.2,server.3就OK了。因为在同一台机器上，后面连着的2个端口3个server都不要一样，否则端口冲突，其中第一个端口用来集群成员的信息交换，第二个端口是在leader挂掉时专门用来进行选举leader所用。

6.使用不同的配置文件启动三个进程;./zkServer.sh start ../../servers/server1/zoo.cfg 

7.连接zk服务器：/zkCli.sh -server ip:port