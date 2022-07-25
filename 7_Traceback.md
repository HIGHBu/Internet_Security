# Attack Traceback

在检测到攻击之后，最简单的方式是记录source IP，但是有IP spoofing的存在

IP spoofing并不能被ingress filteing全部过滤掉

- ISP只转发那些source IP是ISP的合法区域范围内的packet
- 但是如果不是所有的ISP都布置了ingress filtering，那么基本没有效果



router将自己的id写到经过的packet中

可以使用采样+merge的方法，经过的路由器概率把id写到packet里，最后综合一下就能还原出完整的路径

但是packet中的s、e、d字段都是明文存储的，没有加密，容易被篡改，会干扰victim复现攻击路径



### ICMP Traceback

被大部分路由器支持，功能名字叫iTrace

- 每个路由器对它所转发的包中的一个进行**采样**，然后将包的内容和相邻路由器的信息复制到ICMP回溯消息中
  - 提供的不止是自己的信息，还把邻居结点的信息也加进去

- 路由器使用HMAC和X.509数字证书对回溯消息进行认证
- 路由器向目的地发送ICMP回溯消息

**不足**

- 路由器的开销非常大，且由于ICMP Ping Flood Attack，所以ICMP packets可能会被过滤掉

- 如果攻击流量不够多，或者采样不均衡，那么可能无法恢复出一条攻击路径

### Path Validation  不考

路径认证

通信双方严格指定了一条路径，发送者有时会正常通信，有时会发送攻击

PoC: Proof of Consent（许可）

证明提供商同意沿路径传输流量

PoP: Proof of Provenance（出处）

允许上游节点向下游节点证明他们携带了数据包

 



只通过路由协议，无法找到具体哪台PC发送了什么packet，只能找到网络边界的路由器

### Link Testing

从离victim最近的router开始，逐个找到路由路径的上一个router，直到找到离攻击者最近的router

需要假设攻击是持续的

#### Input Debugging

1. 发现攻击流量后，需要提取出攻击流量的特征
2. 然后路由器将攻击流量的特征发送给上游路由器，由上游路由器对攻击包进行过滤，并确定攻击流量进入的端口
3. 这样当前路由器就可以确定是不是这个端口来的攻击流量了

#### Controlled Flooding

需要协作的主机和精准的拓扑结构

强制主机将链路泛洪到上游路由器

由于受害链路上的缓冲区被所有入站链路共享，导致攻击链路泛洪，导致攻击报文掉落

在上游路由器上递归地应用上述技术，直到到达攻击源



## Post-attack traceback

攻击结束后才进行调查

### Logging-Based Traceback

router存储转发过的packet的信息（logs）

victim递归向最近的router查询是否出现过packet

#### Bloom Filter 知道就行

用多轮哈希，来降低冲突的概率

每个输入做多次哈希，都对哈希表置1

查询的时候，如果不是多个都置1，那么说明没有；但是如果都是1，那还是有概率没有

从header中选取一些不变的量，当作bloom filter的输入，来作为packet的log

