---
title: 为家庭网络设置DDNS
toc: true
date: 2019-10-01 17:19:00
updated: 2019-10-02 22:59:00
categories: 折腾
tags: [家庭组网, DDNS, shell, AppleScript]
thumbnail: /gallery/15699387480111.jpg
---

在家中网络架好以后，下一步就是要方便的随时访问家中的设备了。而即使申请到公网IP，一般家用级IP却不固定，给我们访问造成了麻烦。本博客介绍了解决这个问题的若干途径。
<!-- more -->
*若不想看我前面瞎BB的回忆录，可直接跳到下面最终的[DDNS方法](#DDNS方法)*
## 非DDNS方法（消息通知）
这种方法是我最早在2017年时采用的，因当时国内比较知名的DDNS服务商，如花生壳啥的，已陆续不提供免费DDNS服务，故只能另辟蹊径，通过消息通知的方式来获取家网IP。

想过用微信推送，实现起来也简单，有一堆公众号支持HTTP GET/POST接口往特定微信号推送消息，但这种方式安全性不高，有暴露IP的风险，也不想为这件本应自己解决的事去专门关注公众号。另外有个考量点，我希望在电脑（Mac系统）上自动更新我的家网IP，这样的话，最好能利用到苹果系统本身的自带的App，可更好的与系统联动。

综合以上想法，并查阅资料后，我找到了一种还算顺畅的IP更新流程:
路由器检测到IP变化 -> 将新IP用Email发出 -> 邮件服务器 -> 电脑接收邮件 -> 自动运行脚本更新hosts

### 检测IP变化
#### cron定时任务
如何检测本机外网IP发生变化？如果是在内网某台电脑上做检测，可能需要`curl`某个IP检测服务地址，如：

``` bash
curl myip.ipip.net
```
这样返回的信息中会包含外网IP，可将IP提取出来写入临时文件，并`crontab`一个定时任务（如每分钟检测一次），对比上次结果，若IP有变就发出邮件提醒。

而我们现在是直接在有外网IP的路由器上做检测，就没有这麻烦了，直接`ifconfig`就能知道WAN端口的各种IP配置，然后`sed`截取出需要的IPv4地址：

``` bash
ifconfig pppoe-wan | sed -En 's/.*addr:([0-9.]+).*/\1/p'
```
知道了当前IP，还需要知道其是否有变。当然，也可以如前述那样起cron定时任务进行检测，也不会消耗什么资源。只是，在OpenWrt固件的路由器上，我们可以有一种更合理的方式。

#### hotplug脚本
在OpenWrt系统中，提供了hotplug功能，后台相关daemon进程（`procd`）在监测到某些事件时会执行可自定义的一些shell脚本。利用这个特性，我们可以在`/etc/hotplug.d/iface/`目录下新建一个名为`99-ipreport`的脚本，内容如下：

``` bash
#!/bin/sh
[ "$ACTION" = "ifup" -a "$INTERFACE" = "wan" ] && sh /root/update_ip.sh
```
文件名中的“99-”指定了脚本的执行顺序，越大越靠后。脚本的含义是，当`wan`接口发出`ifup`，即上线的事件消息时，就执行后面的`/root/update_ip.sh`，该脚本负责检测及通知更新后的IP（`update_ip.sh`脚本具体内容见文末[附录](#附录)）。
本方法和前面的cron相比，可避免起定时任务，只在路由器重新拨号时，即IP真正可能发生变化时，执行检测脚本。

### 路由器发邮件
关于路由器如何发邮件，网上已有很多教程，这里简单说明下。

一般OpenWrt固件，都已自带ssmtp邮件客户端，可方便的发邮件（如果固件不带，请尝试安装对应软件包）。我这里使用到的邮箱是163的，仅作参考。163邮箱需要先在设置中开启SMTP服务：
![](/gallery/15699906376197.jpg)

为保障邮箱安全，建议开启客户端授权密码：
![](/gallery/15699907225715.jpg)

邮箱设置完成后，在路由器的ssmtp设置文件`/etc/ssmtp/ssmtp.conf`加入以下配置：

``` ini
root=<your email>
mailhub=smtp.163.com
rewriteDomain=163.com
hostname=openwrt
FromLineOverride=YES
AuthUser=wgdzlh
AuthPass=<your password>
UseTLS=YES
UseSTARTTLS=YES
```
之后就能使用`ssmtp`命令了：

``` bash
echo -e "To: <receiver email>\nFrom: <your email>\nSubject:<title>\n\n<content>" | ssmtp <your email>
```
测试一下能否正常发邮件吧。

### 电脑端自动更新hosts
因没有用到DDNS，我们可以把从邮件获取到的IP写入电脑本地的hosts文件中，以便访问该IP。

对于MacOS而言，系统集成的邮件客户端有个隐藏的强大功能，可以在收到新邮件时，进行规则匹配，通过脚本自动更新系统hosts，从而省掉了手动编辑的麻烦。这里的脚本就是苹果独有的AppleScript。

首先，确定/etc/hosts文件中有自定义的hostname：
![](/gallery/15700201259043.jpg)

然后，在邮件App设置里，添加新规则，如下进行设置：
![](/gallery/15700153020235.jpg)


可以看到，这里对邮件标题进行了规则匹配，只处理标题以特定字符串结尾的邮件。
后面处理步骤应该很清晰了：拷贝到本地特定收件文件夹、设已读、加颜色旗标、运行`AppleScript`。这里的`AppleScript`就是我们自动更新hosts的地方了，内容可参考：

``` applescript
delay 3tell application "Mail"	set theMessages to messages of mailbox "wgdzlhs_ubt" --whose read status is false	if theMessages is not {} then		set aMessage to first item of theMessages		set msgContent to content of aMessage		--log "mail of ip info: " & msgContent & date sent of aMessage		tell me to set wgdzlhs_ubt_ip to do shell script "echo " & quoted form of msgContent & " | grep -Eo '[0-9]{1,3}(\\.[0-9]{1,3}){3}'"		--log "wgdzlhs-ubt ip: " & wgdzlhs_ubt_ip		if length of wgdzlhs_ubt_ip > 0 then			set pattern to "s/^.+(wgdzlhs-ubt)/" & wgdzlhs_ubt_ip & " \\1/"			--log "sed pattern: " & pattern			tell me to do shell script "sed -Ei'_bak' " & quoted form of pattern & " /etc/hosts" with administrator privileges		end if		--move theMessages to mailbox "Trash"		repeat with oneMessage in theMessages			delete oneMessage		end repeat	end if	--log "done."end tell
```
可以看到，AppleScript是一种近乎伪代码的风格，读起来应该比较易懂，这里就不做解释了。值得注意的是脚本中的`sed`命令，在编辑`hosts`时会对原文件进行备份，并需要提升权限（提示用户输入密码）。
另外有个小技巧，在编写AppleScript时，如果需进行测试，其实输出log有点多余（故上面代码中的log命令都被注释掉了），在Script Editor应用中，点击运行后，有一栏能显示出各行命令的实际执行代码及其返回结果，如下图：
![](/gallery/15700195025400.jpg)

`AppleScript`正常运行后，我们的hosts应该就更新好了，测试一下能否ping通家里的路由器：
![](/gallery/15700261554259.jpg)

## DDNS方法
### 缘由
当然，最方便的还是DDNS —— 有固定域名就不用麻烦去更新hosts，且适用所有电脑手机系统。
最近因项目需要，在家里的台式机上部署了GitLab，希望整个团队都能方便访问到，这样就有多个终端、多个用户要访问我的家庭网络，前面的非DDNS方法已不适用了。下面介绍DDNS解决方案。

### 注册服务
鉴于国内没有很好的免费DDNS，我将目光转向了国外。通过搜索，我发现了一家比较靠谱的，地址是：<https://www.dynu.com>。注册后，点击主页右上角齿轮：
![](/gallery/15700277546208.jpg)

进入后台控制中心，就能添加DDNS服务了，设置好自己想要的域名（三级域名免费），不再赘述。

### 路由器自动更新DDNS
有了DDNS服务，同样需要在合适的时候，对记录在DDNS服务商那里的IP进行更新，故前面[hotplug脚本](#hotplug脚本)里的`update_ip.sh`仍可沿用，只是里面运行的命令需要相应调整。

通过查阅该服务商提供的[API文档](https://www.dynu.com/en-US/Support/API)，更新DDNS步骤如下：

1. 生成`API Key`
如下图，在[这里](https://www.dynu.com/en-US/ControlPanel/APICredentials)生成自己账户的`API Key`，想要更安全一点也可以设定验证类型为`OAuth2`，做两步验证（多个临时token而已），这里只介绍`API Key`更新方式：
![](/gallery/15700290733713.jpg)

2. 获取DDNS的用户ID
运行下面`curl`，返回的JSON里就有`id`信息（一串数字），记录下来。这个命令只用运行一次，目的只是为获取DDNS服务的用户识别ID。

 ``` bash
curl https://api.dynu.com/v2/dns -H "accept: application/json" -H "API-Key: <your API Key>"
```

3. 更新DDNS域名对应IP
下面的命令在每次路由器发生拨号行为，并检测出WAN端口IP发生变化后，即需运行：

 ``` bash
curl https://api.dynu.com/v2/dns/<your DDNS id> -H "Content-Type: application/json" -H "API-Key: <your API Key>" -d "{\"name\":\"<your domain name>\",\"ipv4Address\":\"<your new ip>\"}"
```

IP更新后，可用`ping`测试一番，看是否正常。具体所用的shell脚本请见文末[附录](#附录)。通过路由器[hotplug功能](#hotplug脚本)，该脚本可如上面第3步里的那样，按需运行，不用起`cron`。
至此，DDNS的问题已圆满解决。

## 附录
最后，附上题图中的 `update_ip.sh` shell脚本，以供参考。该脚本同时采用了前面两种方式获取家网IP，以防DDNS服务失效：

``` bash
#!/bin/sh

smail() {
  echo -e "To: ${1}\nFrom: <your email>\nSubject:${2}\n\n${3}" | /usr/sbin/ssmtp ${1}
}

ddns() {
  curl https://api.dynu.com/v2/dns/<your DDNS id> -H "Content-Type: application/json" -H "API-Key: <your API Key>" -d "{\"name\":\"<your domain name>\",\"ipv4Address\":\"${1}\"}" &> /dev/null
}

cd /tmp
if [ -f "current_ip.log" ]; then
  old_ip=$(cat current_ip.log)
fi
new_ip=$(ifconfig pppoe-wan | sed -En 's/.*addr:([0-9.]+).*/\1/p')
echo ${new_ip} > current_ip.log
[ "${new_ip}" != "${old_ip}" ] && smail <your email> "$(date +'%Y%m%d %H:%M:%S') my ip updated --from wgdzlhs-ubt" ${new_ip} && ddns ${new_ip}
```

