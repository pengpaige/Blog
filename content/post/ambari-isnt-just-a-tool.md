---
categories:
- 技术文章
date: 2016-03-24T11:20:54+08:00
keywords:
- Ambari
title: Ambari安环境搭建和精神分裂
url: 2016/03/24/ambari/

---
因为实习需要用到这个工具，所以打算现在自己的虚拟机装一个试试，结果一下子遇到了巨多的问题，前后搞了三四天，几乎都要崩溃了，还好最后靠着万能的 Stack Overflow 还是把问题一一解决了，痛苦的细节暂且不说了吧，只记录一下安装的过程和用到的工具。
### 搭建Ambari环境的前期准备
首先得有几台干净的Linux主机或者电脑配置够得话多开几个虚拟机也可以，这里是在主机上开了三个 CentOS6.5 系统的虚拟机，他们分别是 server.shineline.com agent1.shineline.com agent2.shineline.com .前期准备的这几步，如果你也是用虚拟机来搭建环境的话你完全可以先在一台虚拟机上（比如 server ）配置好之后直接把虚拟机文件复制，另外几个副本稍微修改下就可以作为 agent 来使用了。

#### 开启 SSH 服务
后面需要配置 SSH 免密登录，这里必须要先开启 SSH 服务。
1. 先输入`service sshd status`看下 SSH 服务的状态，如果显示`openssh-daemon (pid  xxxx) is running...`说明 SSH 已经启动，可以跳过这一步了。不过为了保险起见，最好先看下你的 SSH 服务是不是默认开机自启动：输入`ntsysv`命令，找到 sshd 的服务如果前面有 * 号就是默认自启（使用空格选中或取消）。
2. 如果 SSH 没启动，那么你需要像上面那样先把 SSH 设置成开机自启之后运行 `service sshd Start`将SSH服务开启。

#### 同步系统时间
集群的时间不一样的话可能会出问题，所以最好先同步一下网络时间。输入`date -R`命令显示当前时间；输入`ntpdate time.windows.com && hwclock -w`命令进行同步，同步之后可以再输入`date -R`命令看下是不是已经同步成功了。

#### 使主机 SELinux 失效
执行命令 `sudo vi /etc/selinux/config` ，在打开的文件里将 SELINUX=xxxx 这样改为`SELINUX=disabled`.

#### 关闭防火墙
执行命令 `sudo chkconfig iptables off`.

#### 设置 umask 值
执行 `umask` 命令，如果显示不是 0022 ，那么打开 ~/.bashrc 文件，在最后添加 `umask 022` .

### 搭建 Ambari HDP 和 HDP-UTILS 的本地 yum 源
下面都是在 server 上的操作。
因为 Ambari 会自动下载搭建 Hadoop 环境的组件，但是使用公共的 yum 源速度会非常非常慢，我这里反正连下载 Ambari 都成问题，况且真正的集群也不应该依赖公共的源，学会本地搭建 yum 源是早晚的事儿，所以还是自己撸一个本地 yum 源吧。

#### 搭建 Apache 服务器
这里是使用 Apache 作为文件服务器，安装执行 `sudo yum install httpd`安装 Apache 服务器。
服务器的工作根目录在 `/var/www/html`，配置文件在这里 `/etc/httpd/conf/httpd.conf`.
安装成功之后执行 `service httpd start` 启动 Apache，之后最好把它设置成开机自启动。在 ntsysv 里面Apache 服务名字叫 httpd .
#### 下载 rpm 软件包
[Ambari 的下载地址](http://public-repo-1.hortonworks.com/ambari/centos6/ambari-2.1.0-centos6.tar.gz)

[HDP 的下载地址](http://public-repo-1.hortonworks.com/HDP/centos6/HDP-2.2.0.0-centos6-rpm.tar.gz)

[HDP-UTILS 的下载地址](http://public-repo-1.hortonworks.com/HDP-UTILS-1.1.0.20/repos/centos6/HDP-UTILS-1.1.0.20-centos6.tar.gz)

如果直接浏览器下载速度太慢的话可以用迅雷什么的，找个账号加个速会快很多。至于账号怎么找，google 搜下一下你就知道了。另外如果不想用我上面的版本，可以替换其他的版本，地址自己摸索着改下就好。
在上面的 Apache 服务器根目录新建一个文件夹 centos-6(名字随意)，将上面的压缩包下载好之后放进 centos-6 目录中。现在你在浏览器输入 server 的IP地址加上 /centos-6 应该可以看到 centos-6 目录下面的内容了。
#### 创建本地 yum 源
先看下你的主机上面有没有 createrepo 工具包，执行 `which createrepo`命令，显示目录说明已经安装了，如果没有显示目录执行`yum install createrepo` 命令安装 createrepo 工具。
安装好之后输入命令 `createrepo /var/www/html/centos-6/ambari-2.1.0/centos6` 创建本地 yum 源的 repo .一般执行上面这条命令的位置都是在 `RPM-GPG-KEY` 文件夹的父目录的位置，执行完命令去目录里看一下，生成了 repodata 文件夹就代表执行成功了。
yum 源的配置文件在 /etc/yum.repos.d 目录，可以先打开看一下，里面 .repo 文件都是配置文件。上一步是生成的一个本地的 yum 源的库，但是主机现在还不知道有这么一个源，要在配置文件目录里添加相应的配置文件才能让主机识别到新添加的源。
新建一个 ambari.repo 文件用来匹配 Ambari 的源，文件内容这样写：
```shell
[ambari-2.1.0]
name=Ambari 2.1.0
baseurl=http://server.shineline.com/centos-6/ambari-2.1.0/centos6
gpgcheck=1
gpgkey=http://server.shineline.com/centos-6/ambari-2.1.0/centos6/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1
```
中括号里面显示当前 repo 里的版本，另外上面比较重要的几个参数：
`baseurl` 是主机的域名再加上刚才创建的repo的目录，这个地址一般写到 RPM-GPG-KEY 目录的父目录级别。
`gpgkey` 要写到 RPM-GPG-KEY-Jenkins 文件的具体位置。
`enable` 等于 1 表示启用这个 repo 。
`gpgcheck` 等于 1 表示启用 GPG 验证。

以上就是搭建 yum 源的全部内容，另外还要搭建 HDP 和 HDP-UTILS-1.1.0.20 的两个源，具体的操作步骤仿照上面来做就可以，最后把 .repo 文件的内容改成对应的内容就可以了。.repo 文件是可以在一个文件里写多个软件包的信息，但是最好每个软件包单独写在一个 .repo 文件里。
所有的 .repo 文件都写好之后执行 `yum clean all`和 `yum repolist`两个命令更新本机的 yum 源记录。
### 真正开始搭建 Ambari 环境

首先，把三台机器的 hosts 文件配置好。在每个主机的 /etc/hosts 文件的后面追加：
```
192.168.248.135         server.shineline.com
192.168.248.137         agent1.shineline.com
192.168.248.138         agent2.shineline.com
```
这样以后 SSH 登录之类的操作就可以直接使用域名（可以随意定义）代替 IP 地址。

#### 配置 server 免密登录所有的 agent
所有的 SSH 登录最好都是使用 root 用户来进行，以免有些权限不支持。

 1. 在 server 的 /root/.ssh 目录下执行`ssh-keygen`命令生成密钥。
 2. 执行 `ssh-copy-id root@agent1.shineline.com` 和 `ssh-copy-id root@agent2.shineline.com` 将公钥拷贝到两个 agent 的 /root.ssh/authorized_keys文件中，就能实现免密登录了，具体原理自己搜下吧。

完成之后尝试一下 ssh 登录看看设置成功没有。
在 server 端执行命令`ssh agent1.shineline.com`和`agent2.shineline.com`.

#### 配置所有 agent 的 repo 使他们能连接到 server 上的额自建 yum 源
为了让 agent1 和 agent2 都能够连接上搭建在 server 上的 yum 源，他们也分别需要更新 yum 源，所以要把 server 上的 .repo 文件拷贝到 agent1 和 agent2 上去。由于上面已经配置好了免密登录那么拷贝的工作就简单多了。


切换到 /etc/yum.repos.d 目录并执行命令：

`scp ambari.repo HDP.repo HDP-UTILS-1.1.0.20.repo agent1.shineline.com/etc/yum.repos.d`和

`scp ambari.repo HDP.repo HDP-UTILS-1.1.0.20.repo agent2.shineline.com/etc/yum.repos.d`
同样的，执行 `yum clean all`和 `yum repolist`两个命令更新本机的 yum 源记录。

#### 安装 Ambari server
你需要现在三台机器上安装 JDK1.7 ，教程很多这里就不多说了。
执行`yum install ambari-server`，yum 会从上面已经设置好的本地 yum 源下载 rpm 包。
之后执行命令`ambari-server setup`进行 server 的配置，我这里为了快速装完只在 JDK 一项选择了输入本地的 JDK 目录，其他的设置全部使用了默认选项。具体的设置基本都是可以定制的，可以 google 一些资料自己设置。这里不再细说。
另外有一个问题就是，当配置好之后 Amabri 会自动将主机名自改成 server 和 agent1,agnet2 也就是 hosts 文件里的最底层域名。
好了，现在你可以体验一波 Ambari 的自动化集群安装和运维了，执行命令`ambari-server start`，在浏览器输入 server 的 IP 地址加上端口号 8080 回车看看吧。