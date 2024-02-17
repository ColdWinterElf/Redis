##### Redis

    通过两层拦截器拦截请求第一层拦截器拦截所有请求 判断user是否存在 存在更新token 不存在直接放行，第二层拦截需要登陆的请求做校验
判断userholder.getuser是否为null 如果为null放行 并更新token时间

魔法值：业务代码中直接出现的常量。这些常量至少要放到全局变量中，或者枚举类中
redis-cli连接redis
单节点redis问题
1.	数据丢失：redis是内存存储，重启会丢失数据 
解决办法：实现redis持久化
2.	并发能力问题：对更高的数据量如618就不够用
解决办法：搭建主从集群，实现读写分离
3.	故障恢复问题：如果redis宕机，服务就不可用了，需要自动恢复故障手段
解决办法：利用redis哨兵，实现健康检测和自动恢复
4.	存储数量问题：基于内存存储，难以满足海量数据要求
解决办法：搭建分片集群，利用插槽机制实现动态扩容
RDB被称作数据快照，将内存中的所有数据记录到磁盘上，当redis故障重启后，从磁盘读取快照文件，恢复数据。
Save保存快照文件s
bgsave后台异步执行 开始时会fork主进程得到子进程，紫禁城共享	主进程的内存数据。完成fork后读取内存数据并写入RDB文件。fork采用的是copy-on-write技术
当主进程进行读操作，访问共享内存
当主进程进行写操作，则拷贝一份数据，执行写操作。
Fork主进程得到子进程，共享内存空间
子进程读取内存数据并写入新的RDB文件
用新RDB文件替换旧的RDB文件 默认在服务停止时执行，save 60 1000含义为：60s内有1000次修改出发RDB
AOF文件不断追加写操作命令一般设置为ｅｙｅｒｙｓｅｃ 命令记录会重复 可以通过bgrewriteaof执行重写，可以设置重写触发条件。
 ![image](https://github.com/ColdWinterElf/Redis/assets/77095414/1b27b20e-f355-497a-aa0b-69d2d96dfa8c)

 ![image](https://github.com/ColdWinterElf/Redis/assets/77095414/9b32f3f3-a9ab-405d-b8f4-b9165c55b4ae)

 ![image](https://github.com/ColdWinterElf/Redis/assets/77095414/a42f323a-2af7-4820-8f93-4115cdda2773)
一般来讲读操作也就是查询是多于写操作，而单节点redis并发能力有限，所以需要搭建主从集群，实现读写分离。从节点只允许读操作，主节点只允许写操作以此实现读写分离。

命令格式 （slaveof/replicaof） 从ip 主ip 
 ![image](https://github.com/ColdWinterElf/Redis/assets/77095414/89dd69d1-5b05-4cce-aed9-64e8c90dfd5f)

主从第一次要做全量同步
1 从节点执行replicaof命令，建立连接，并向主节点请求数据同步。
2 主节点判断是否是第一次同步。
3 是第一次，返回master的数据版本信息。
4 从节点保存版本信息
5 主节点执行bgsave，生成RDB并向从节点发从RDB，主节点记录在RDB生成期间的所有命令repl_baklog。
6 从节点清空本地数据，加载RDB文件
7 发送repl_baklog中的命令，从节点执行接收到命令
主从数据同步原理
 ![image](https://github.com/ColdWinterElf/Redis/assets/77095414/8e739745-7018-4561-86e0-c732ff9f6d1e)

全量同步流程
1 从节点请求增量同步
2 master节点判断replid，发现不一致，拒绝增量同步
3 master将完成内存数据生成RDB，发动RDB到slave
4 slave清空本地数据，加载master的RDB
5 master将RDB期间的命令记录在repl_baklog,并持续将log中的命令发送给slave
如果slave节点重启后同步，则执行增量同步
1 从节点psync replid offset
2 主节点判断replid是否一致
3 不是第一次 回复continue
4 去 repl_baklog获取offset后的命令，并发送给从节点offset后的命令，从节点执行命令
 ![image](https://github.com/ColdWinterElf/Redis/assets/77095414/5258f3ea-801f-4287-b204-379da9c4bb8e)

优化方向：
1在master中配置repl-diskless-sync yes启用无磁盘复制避免全量同步时的磁盘IO
2 redis单节点内存占用不要太大，防止生成RDB文件时产生大量的磁盘IO
3适当提高repl_baklog的大小，发现slave宕机时尽快实现故障修复，避免全量同步
4限制一个master上的slave节点数量，如果slave节点太多，可以采用主-从-从链式结构，减少master压力
 ![image](https://github.com/ColdWinterElf/Redis/assets/77095414/750d78e8-4549-4869-8be3-b73651d14fff)

redis提供了哨兵机制Sentinel来实现主从集群的故障恢复。
1.	监控：sentinel会不断检查master和slave节点是否按预期工作
2.	自动故障恢复：如果master故障，sentinel会将一个slave提升为master。当故障实例恢复后也以新的master为主
4.	sentinel充当redis客户端的服务发现来源，当集群发生故障转移时，会将	最新信息推送给redis客户端
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/2ca7978f-cbaa-46fd-ad61-500c54fe3ff8)
sentinel基于心跳机制监测服务状态，没隔1秒向集群的每个实例发送ping命令：
主观下线：如果某sentinel节点发现某实例未在规定时间相应，则认为该实例主观下线
客观下线：如超过指定数量（quorum）的sentinel都认为该实例主观下线，则该实例客观下线。quorum值最好超过sentinel实例数量的一半
选举新的master
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/10547289-1548-45c0-8f15-543bbfc80df4)
如何实现故障转移
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/2f22ffbe-5e75-47e7-8717-43239f773401)
关于哨兵的问题
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/63f73b61-2713-4ba3-b819-460475900a4e)
redistemplate的哨兵模式
 ![image](https://github.com/ColdWinterElf/Redis/assets/77095414/67e493a1-6701-465d-bb9e-8e9625022039)
分片集群结构
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/7729c1a1-7948-4ea6-be7a-04e527bec132)
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/14b7a536-1386-4ce6-9eea-ed98896affb9)
redis分片集群集群故障转移
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/cd922c0b-1d2a-4be4-b696-d38d3aa06049)
数据迁移由于（机器老旧或升级或其他）
传统缓存存在的问题
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/0d75d05a-56e6-48cf-907f-74bdffd0f80c)
采用多级缓存方案降低服务器压力
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/08262893-8063-471a-93fb-73a9864c6443)
本地进程缓存
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/cfb3d74f-855e-4532-a050-9dec970133d8) 
caffeine示例
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/8c3105dd-089d-4e41-8916-fc439c77f1f0)
冷启动与缓存预热
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/c0aa3449-2ceb-431c-83f1-08f984532bf4)
缓存预热
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/376f9826-5573-4ce0-9609-14d7c94b6f5c)
canal
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/9d9fcdee-e63e-417b-ba1b-0f0498d84f07)
缓存同步策略
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/f13882b0-076e-44de-b130-a78af49a35d9)
key结构推荐
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/9d29889f-fb81-44e1-b238-aa6ce153820d)
Bigkey危害
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/8904ac74-242c-443f-b1cb-f5625ec55c97)
如何发现BigKey
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/0cf437ed-415e-40b2-acab-20cfa8ef27d3)
删除BigKey
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/e831e867-971a-4762-91ef-264f2125700d)
对象类型如何设置数据类型避免BigKey的出现
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/fa66b78c-9e67-4de5-b592-79bef780de29)
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/1ae63143-c2c4-43bc-872d-12c1a8667cd5)
 KEY VALUE的最佳实践
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/45f9c001-3d58-44c1-8c00-853510ff9611)
批处理命令操作方案
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/e5ef9858-3edf-483a-b504-58fd57d89868)
集群下的批处理方案
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/e418e28f-9c66-4991-a609-8d0e9fa48c51)
持久化配置推荐
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/7f350748-0017-4ccf-bc93-fda57a95bf40)
安全防护措施
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/bfb0c261-05e8-457b-bf11-526ec779b5c6)
reids内存配置
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/83f5a328-9a5a-48fe-b0b3-3ce03d05198d)
内存缓冲区配置
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/460ab6cb-82cd-4f6f-a4b7-6ecc9b5073f9)
集群使用不当存在的问题
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/f63b1abe-0e0f-4c98-bcce-062d2d266428)
集群完整性问题
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/82b4de6f-c523-47a2-9c85-c4f0e73d9f0e)
集群带宽问题
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/9c24efba-4b30-4118-835d-0b451529a613)
选择集群还是主从
![image](https://github.com/ColdWinterElf/Redis/assets/77095414/719136a6-b00b-42a0-a084-ef5a5466d533)
















































































































































































































































