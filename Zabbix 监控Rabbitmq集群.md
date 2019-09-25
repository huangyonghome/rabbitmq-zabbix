### Zabbix 监控Rabbitmq集群



github上有一个非常不错的rabbitmq监控模板:[rabbitmq-zabbix](<https://github.com/jasonmcintosh/rabbitmq-zabbix>)

本文档不赘述zabbix的配置细节,仅仅记录该rabbitmq监控模板的使用

一. 将github的模板文件克隆到服务器上:

```
oot@hsq-mq-node1:~/rabbitmq-zabbix# ll
total 132
drwxr-xr-x  6 root root  4096 May 30 18:36 ./
drwx------ 11 root root  4096 Jun 18 09:27 ../
drwxr-xr-x  8 root root  4096 May 30 11:05 .git/
-rw-r--r--  1 root root    40 May 30 11:05 .gitignore
-rw-r--r--  1 root root 10273 May 30 11:05 LICENSE
-rw-r--r--  1 root root 76316 May 30 11:05 rabbitmq.template.xml
-rw-r--r--  1 root root  3274 May 30 11:05 README.ja.md
-rw-r--r--  1 root root  5598 May 30 11:05 README.md
drwxr-xr-x  3 root root  4096 May 30 11:05 scripts/
drwxr-xr-x  2 root root  4096 May 30 11:05 tests/
-rw-r--r--  1 root root   500 May 30 11:05 .travis.yml
drwxr-xr-x  2 root root  4096 May 30 11:05 zabbix_agentd.d/
```

其中:

scripts目录下是监控脚本文件

zabbix_agentd.d目录下是zabbix监控配置文件

rabbitmq.template.xml是zabbix监控模板.



二.将scripts目录下所有文件拷贝到/etc/zabbix/scripts目录下:

```
work@hsq-mq-node1:/etc/zabbix/scripts$ ls rabbitmq/ -al
total 60
drwxr-xr-x 2 zabbix zabbix  4096 Jun 17 10:20 .
drwxr-xr-x 3 zabbix zabbix  4096 May 31 17:36 ..
-rwxr-xr-x 1 root   root   14149 May 31 17:56 api.py
-rwxr-xr-x 1 root   root   13957 May 31 11:54 api.py.bak
-rwxr-xr-x 1 zabbix zabbix   424 May 30 11:43 list_rabbit_nodes.sh
-rwxr-xr-x 1 zabbix zabbix   426 May 30 11:43 list_rabbit_queues.sh
-rwxr-xr-x 1 zabbix zabbix   430 May 30 11:43 list_rabbit_shovels.sh
-rwxr-xr-x 1 zabbix zabbix   170 Jun 17 10:20 .rab.auth
-rwxr-xr-x 1 zabbix zabbix   782 May 31 14:23 rabbitmq-status.sh
```

文件分析:

* api.py                                --python脚本，利用rabbitmq的web api，获取监控的相关数据
* list_rabbit_nodes.sh       --shell脚本，将参数传给api，获取节点数据
* list_rabbit_queues.sh     --shell脚本，将参数传给api，获取队列数据
* rabbitmq-status.sh        --shell脚本，将参数传给api，获取状态数据
* .rab.auth                          ---参数设置，设置登陆rabbitmq的相关参数



三.将zabbix_agentd.d/zabbix-rabbitmq.conf文件拷贝到/etc/zabbix/zabbix_agentd.conf.d目录下

```
root@hsq-mq-node1:~/rabbitmq-zabbix# ll /etc/zabbix/zabbix_agentd.conf.d
total 16
drwxr-xr-x 2 root   root   4096 May 31 16:34 ./
drwxr-xr-x 4 root   root   4096 Jun 17 10:44 ../
-rw-r--r-- 1 root   root    356 May 30 11:49 zabbix-rabbitmq.conf
-rw-r--r-- 1 zabbix zabbix  140 May 30 11:00 userparameter_disk.conf
```



注意: 关于scripts目录下的脚本文件和zabbix-rabbitmq.conf配置文件的存放路径,要参考zabbix-agentd.conf的zabbix客户端配置文件中定义的路径.例如以下是我服务器上的配置

```
root@hsq-mq-node1:~/rabbitmq-zabbix# sed '/^#/d' /etc/zabbix/zabbix_agentd.conf | sed '/^$/d'
PidFile=/var/run/zabbix/zabbix_agentd.pid
LogFile=/var/log/zabbix-agent/zabbix_agentd.log
LogFileSize=0
Server=172.16.10.214
ServerActive=172.16.10.214:10051
Hostname=hsq-mq-node1
Include=/etc/zabbix/zabbix_agentd.conf.d/*.conf   #配置文件路径.
Timeout=30
```



四.因为rabbitmq服务器上有太多的默认和随机queue队列.所以需要对脚本的监控队列进行过滤.不然队列太多,会撑爆监控服务器.请参考我在github上提的issue: [how to filter default queue?](<https://github.com/jasonmcintosh/rabbitmq-zabbix/issues/104>):

4.1 修改scripts目录下的.rab.auth隐藏配置文件:

```
root@hsq-mq-node1:/etc/zabbix/scripts/rabbitmq# cat .rab.auth
USERNAME=zabbix   #zabbix用户名
PASSWORD=zabbix   #zabbix密码
CONF=/etc/zabbix/zabbix_agentd.conf #zabbix配置文件
LOGLEVEL=DEBUG 
LOGFILE=/var/log/zabbix-agent/rabbitmq_zabbix.log 
PORT=15672  
FILTER='[{"name":"hsq"}]'  #过滤的队列.这个字段表示只监控队列名以hsq开头的队列
```

> 需要提前在rabbitmq集群中创建zabbix用户密码,并且给与monitor监控权限



4.2. 由于原始的监控模板只支持过滤特定的队列名,而不支持队列名通配符匹配.参考上面的Issue..所以需要对代码文件进行修改.

修改api.py文件:

```
#第52行. 注释 check = [(x, y) for x, y in queue.items() if x in _filter]
```

```
#第52行. 将shared_items = set(_filter.items()).intersection(check) 更改为:

shared_items = [(x, y) for x, y in queue.items() if x == u'name' and y.startswith(_filter['name'])]
```

参考如下:

```
shared_items = [(x, y) for x, y in queue.items() if x == u'name' and y.startswith(_filter['name'])]
#check = [(x, y) for x, y in queue.items() if x in _filter]
#shared_items = set(_filter.items()).intersection(check)
```

另外下列位置也需要进行同样的修改:

```
line 77
line 119
line 151
```

修改完后保存文件

五.启动zabbix_agentd服务

```
systemctl start zabbix-agent
```

六,配置zabbix监控主机,导入监控模板.搞定