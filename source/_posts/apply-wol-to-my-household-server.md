---
title: 为家庭服务器设置WOL
toc: true
date: 2019-10-03 01:21:12
updated: 2019-10-06 19:57:13
categories: 折腾
tags: [家庭组网, WOL, shell, iptables, TCP]
thumbnail: /gallery/triStates.png
---

在完成[家庭组网](/2019/09/29/how-do-i-construct-my-household-network/)后，剩下的工作就是各种优化了。其中应重点关注的，当属家庭网络设备运行的经济性，以及使用的方便性了。本文主要介绍WOL（Wake On LAN）在提高设备经济便捷性方面的应用。
<!-- more -->
## 问题的引出
最近，在我重新启用家里的台式机，作为开发团队的`GitLab`服务器时，一个问题很快显现出来。

首先，之所以需要将家里的电脑当作服务器来用，主要有以下几点原因：

* 穷，买不起配置较好的VPS（`GitLab`对配置有要求）
* 家里路由器有公网IP，且已[配好DDNS](/2019/10/01/apply-ddns-to-my-household-network/)
* 台式机闲置已久，系统是`Ubuntu 1604`，可充当服务器
* 偶尔需要登录台式机，跑点测试，玩一下`TensorFlow`啥的（台式机有老独显，可加点速）

但，毕竟是过时硬件 + Linux，整个系统功耗有点高（八九十瓦至少），7*24运行的话，不仅会产生可观电费，机器也会快速积灰、折旧。同时经过观察，这台服务器使用频率非常低，一天半小时顶破天。

于是，要合理的利用，最好可以让服务器在不用的时候自动进入睡眠待机（挂起状态，此时绝大部分硬件不工作，功耗极低），而需要用的时候可以远程唤醒。

## 解决过程
### Ubuntu自动睡眠
先从看似简单的下手。在`Ubuntu`系统中，需要设置在不活动一段时间后，自动进入睡眠状态，大家可能觉得这不是一个问题：在设置里的电源选项中设一下就好了嘛。

可是，经过实际操作检验，这样很可能不会成功，至少在我的这台`Ubuntu 1604`老系统上没用。即使我设置了在不活动10分钟后就挂起，但不管系统处于idle状态多久，它就是不睡。

经过分析，我发觉应该是系统中运行的某些服务阻止了电源选项发挥作用，如`GitLab`后台运行的一堆东西，数据库、`nginx`、`Web`服务器等，其中只要有一个让电脑不能主动进入睡眠状态，系统就不会在规定时间休息了。而一旦停掉GitLab所有服务，系统就可恢复正常。只是，停服务是不可能停的，这次停了某一个，下次再起别的什么服务后若又出问题，就又要重复排查。
#### 强制睡眠
如何解决自动睡眠问题？针对诸如此类系统方面的glitch，个人一贯追求简单直接的处理方法，不愿花太多精力去一一排查，追根究底。借用一位前同事的说法，*我们毕竟是开发，不是运维*，可能绕了一大圈，最后最有效的方法就是最简单直接的那个。

所以，既然不能利用系统自身的自动睡眠，那就手动强制睡眠吧：

``` bash
sudo systemctl suspend
```
通过`sudo`提权后，系统当然会乖乖去睡了。注意，可以睡眠不代表也能正常从睡眠中恢复，这时应测试一下唤醒情况。

PS：若你和我一样用的是N家的独显，在唤醒时如果发现系统当机了，内核panic，有可能是显卡驱动造成的，试着重装一下最新兼容驱动或许能解决。
PPS：N卡驱动可能只在图形界面可以正常睡眠唤醒，如果将启动等级设为命令行界面，即`text mode`，不运行`X server`（少跑几十个进程），唤醒时依然会当机黑屏。这个至今没有妥善解决，后面若找到方法会分享出来（很有可能这是现有驱动本身的依赖特点，根本没有解决方法，除非发生小概率事件：未来新驱动支持`text mode`的睡眠唤醒）。
#### 自动判断idle
系统可以通过命令强制睡眠后，下一步就是如何在合适的时候运行这个命令了。

理想情况是，系统能自动判断前台或后台没有正在运行的用户程序，也没有远程用户在使用其提供的网络服务，且这种状态持续了可设定的一段时间，就认为系统正处于空闲，可以安全进入睡眠了。只是，前面已提到，设置中的电源选项在很多情况下不起作用。进一步的搜索表明，这里的电源选项仅能判断前台GUI界面中是否有用户活动，即鼠标或键盘事件，而不能探测后台中的活动，故其作用面非常有限。

有网友分享过自己的经验，也有一些开源脚本在特定情形可利用，但经过分析，判断系统idle状态其实并不复杂，可以用shell脚本加定时任务自行解决，不需要借助其他软件，下面贴出代码：

``` bash
#!/bin/bash

user_proc_num=$(ps -fU <your username> | grep -vE "systemd|sd-pam|ps -fU|wc -l" | wc -l)
if [ ${user_proc_num} -gt 1 ]; then
  active="true"
else
  net_con_num=$(netstat -ant | grep -E "<service socket 1>|<service socket 2>|..." | grep -v "LISTEN" | wc -l)
  if [ ${net_con_num} -gt 0 ]; then
    active="true"
  fi
fi

out_inac="/tmp/inac_times.log"
if [ "${active}" = "true" ]; then
  echo 0 > ${out_inac}
else
  inac_times=$(cat ${out_inac} 2>/dev/null || echo 0)
  if [ ${inac_times} -ge 10 ]; then
    echo 0 > ${out_inac}
    systemctl suspend
  else
    echo $(( ${inac_times} + 1 )) > ${out_inac}
  fi
fi
```
以上代码保存为`auto_suspend.sh`（注意修改其中的`service socket`为需要判断空闲状态的TCP端口），加可执行权限，在`cron`中起定时任务（`root`用户的`crontab`，因为`systemctl suspend`这句须提权）：

``` bash
$ sudo crontab -l
*/1 * * * * /some_path/auto_suspend.sh
```
即每分钟运行一次，判断空闲状态。结合脚本中第18行里的`10`，可看到这里设置的是，若连续十分钟内，`10`次都未探测到系统有活动的进程或TCP端口，则认为系统处于空闲，可强制进入睡眠。

### 路由器唤醒Ubuntu
#### 手动唤醒
在设置自动唤醒前，可先测试一下手动唤醒。测试之前，需打开Ubuntu网卡的`WOL`功能：

``` bash
$ sudo ethtool -s eno1 wol g
```
*注：其中`eno1`改为需设置`WOL`的网卡名。*
然后看下是否配置成功：

``` bash
$ sudo ethtool eno1
```
输出有`Wake-on: g`，就说明已成功设置`WOL`。
在Ubuntu上`systemctl suspend`后，SSH登入路由器，运行以下命令：

``` bash
$ etherwake -b -i <LAN interface name> <ubuntu MAC address>
```
测试是否可以成功唤醒Ubuntu。
*注：除了这种`etherwake`命令方式，还有其他各种WOL工具可以使用。若从外网唤醒，需要在路由器上配置好端口转发，但此种转发方式的弊端是，只能在Ubuntu直接连在路由器某LAN网口上，且配置好静态`ARP`绑定的条件下才行，否则会因`ARP`老化而无法成功转发magic唤醒包（例如像[这里](/2019/09/29/how-do-i-construct-my-household-network/)的网络拓扑，路由器与Ubuntu Desktop PC之间有其他网桥设备）。所以，最好直接从路由器发出广播型的magic包，不管内网的二层拓扑怎样，Ubuntu都能正常收到而被唤醒。故上面`etherwake`命令里有`-b`参数，即发出`broadcast`型magic包。*

#### 自动唤醒
手动唤醒测试通过后，下一步就是将其自动化。如何在需要连接Ubuntu时，自动将其唤醒？简单的端口转发当然无法做到这个，我们需要的是一种路由器可以察觉到的连接信号。经过搜索，我找到了一种很好的解决流程：

1. 某客户端在外网发出连接Ubuntu的TCP第一次握手（`SYN=1 ACK=0`）；
2. 路由器`iptables`转发时，识别出该数据包，发出特定信号；
3. 路由器上某进程接收到该信号，运行`etherwake`命令，发出二层广播magic包唤醒Ubuntu。

##### 识别TCP第一次握手
利用`iptables`监听跳转，可识别特定TCP第一次握手包，发出相应信号。这里的*信号*是写入特殊字符串开头的系统日志，其他进程可以通过实时读取日志来识别这个*信号*。之所以采取这种迂回方式，主要原因是`iptables`无法直接启动其他命令或脚本。

根据OpenWrt官网提供的`iptables`设置样例，我们的监听跳转命令如下：

``` bash
# create a new chain for logging forwarded packets
iptables -N forwarding_log_chain

# append to openwrt forwarding_rule chain (which generally has nothing in it)
iptables -A forwarding_rule -j forwarding_log_chain

# add log rules all HTTP/S SYN (can use --syn instead of --tcp-flags) and FIN-ACK events
iptables -A forwarding_log_chain -p tcp -d <LAN ip of the server> --tcp-flags ALL SYN -j LOG --log-prefix "MY_PC_SYN:"
```
这些命令可以写入OpenWrt的防火墙自定义规则中，在防火墙每次启动时自动运行。

##### 发出广播唤醒包
在路由器上使用`nohup`另起一个后台进程，实时读取系统日志，发现有连接Ubuntu的TCP第一次握手信号，就发magic唤醒包。当然，通过优化发包逻辑，可以适当限制一下发包频率，避免给LAN传太多二层广播（虽说这里的广播包很小，占用带宽极小，但有优化总是好的）。

``` bash
nohup logread -f | awk '/MY_PC_SYN/ {system("/root/auto_wol.sh")}' & 
```
这个`nohup`任务可以写在`/etc/rc.local`里，随系统启动。其中的`/root/auto_wol.sh`脚本内容如下：

``` bash
#!/bin/sh

outfile="/tmp/autowake.tmp"
last_time=$(date +%s -r ${outfile} 2>/dev/null || echo 0)
interval=$(( $(date +%s) - ${last_time} ))
epoch=$(cat ${outfile} 2>/dev/null || echo 0)

if [ ${epoch} -eq 0 ]; then
  if [ ${interval} -le 6 ]; then
    etherwake -b -i <LAN interface name> <ubuntu MAC address>
    echo 2 > ${outfile}
  else
    echo 1 > ${outfile}
  fi
else
  if [ ${interval} -le 6 ]; then
    if [ ${epoch} -eq 1 ]; then
      etherwake -b -i <LAN interface name> <ubuntu MAC address>
      echo 2 > ${outfile}
    fi
  elif [ ${epoch} -eq 2 ]; then
    echo 1 > ${outfile}
  else
    echo 0 > ${outfile}
  fi
fi
```
可以看到，第16行设置了6秒内，最多只会发出一个广播唤醒包。

同时，我们的脚本还可以防IP端口扫描器的扫描骚扰，避免无谓的唤醒（通过检查系统日志，发现几个来自荷兰的IP隔几个小时就对我的路由器各端口进行扫描，疑似想通过漏洞侵入系统）。

这里具体的实现原理很简单，IP扫描器为了提高效率，每个`IP:SOCKET`套接字一般只会发出一个TCP握手包，没响应就扫下一个，而正常的TCP访问更*执着*，第一个握手包没响应，一两秒内会发出另一个。利用两者区别，可以故意设计为只响应连续的握手包，而对孤立的包置之不理。所以，脚本的状态就有了`0、1、2`这三个，三者间的转化关系如下：
<div align="center">

![唤醒脚本三重态](/gallery/triStates.png)
</div>

#### 几点思考
1. 这样的TCP连接惰性处理，在能抵抗一次IP套接字扫描的同时，也多少增加了服务器的响应时延。经过外网访问测试，服务器从睡眠中自动唤醒，没有惰性处理和有惰性处理，相隔1-2s，在总时延10s左右的情况下，可忽略不计。

2. 引入WOL后，所有的网络服务大多情况下均会附加一个较长的10s初始时延，对用户体验有点影响，但在节省大量资源的利益面前，这点影响同样可忽略不计了。

至此，WOL问题已得到解决。

