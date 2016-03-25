---
categories:
- 技术文章
date: 2016-03-24T11:20:54+08:00
keywords:
- Ambari
title: "Ambari安环境搭建和精神分裂"
url: "/2016/03/10/ambari/"

---

###搭建Ambari环境的前期准备
首先得有几台干净的Linux主机或者电脑配置够得话多开几个虚拟机也可以，这里是在主机上开了三个 CentOS6.5 系统的虚拟机，他们分别是 server.shineline.com agent1.shineline.com agent2.shineline.com .前期准备的这几步，如果你也是用虚拟机来搭建环境的话你完全可以先在一台虚拟机上配置好之后直接把虚拟机文件复制，另外几个副本稍微修改下就可以作为 agent 来使用了。

####开启 SSH 服务
后面需要配置 SSH 免密登录，这里必须要先开启 SSH 服务。
1. 先输入`service sshd status`看下 SSH 服务的状态，如果显示`openssh-daemon (pid  xxxx) is running...`说明 SSH 已经启动，可以跳过这一步了。不过为了保险起见，最好先看下你的 SSH 服务是不是默认开机自启动：输入`ntsysv`命令，找到 sshd 的服务如果前面有 * 号就是默认自启（使用空格选中或取消）。
2. 如果 SSH 没启动，那么你需要像上面那样先把 SSH 设置成开机自启之后运行 `service sshd Start`将SSH服务开启。

####同步系统时间
集群的时间不一样的话可能会出问题，所以最好先同步一下网络时间。输入`date -R`命令显示当前时间；输入`ntpdate time.windows.com && hwclock -w`命令进行同步，同步之后可以再输入`date -R`命令看下是不是已经同步成功了。

####使主机 SELinux 失效
执行命令 `sudo vi /etc/selinux/config` ，在打开的文件里将 SELINUX=xxxx 这样改为`SELINUX=disabled`.

####关闭防火墙
执行命令 `sudo chkconfig iptables off`.

####设置 umask 值
执行 `umask` 命令，如果显示不是 0022 ，那么打开 ~/.bashrc 文件，在最后添加 `umask 022` .

###搭建 Ambari HDP 和 HDP-UTILS 的本地 yum 源
下面都是在 server 上的操作。
因为 Ambari 会自动下载搭建 Hadoop 环境的组件，但是使用公共的 yum 源速度会非常非常慢，我这里反正连下载 Ambari 都成问题，所以还是自己撸一个本地 yum 源吧。

####搭建 Apache 服务器
这里是使用 Apache 作为文件服务器，安装执行 `sudo yum install httpd`安装 Apache 服务器。
服务器的工作根目录在 `/var/www/html`，配置文件在这里 `/etc/httpd/conf/httpd.conf`.
安装成功之后执行 `service httpd start` 启动 Apache，之后最好把它设置成开机自启动。在 ntsysv 里面Apache 服务名字叫 httpd .
####下载 rpm 软件包
[Ambari 的下载地址](http://public-repo-1.hortonworks.com/ambari/centos6/ambari-2.1.0-centos6.tar.gz)
[HDP 的下载地址](http://public-repo-1.hortonworks.com/HDP/centos6/HDP-2.2.0.0-centos6-rpm.tar.gz)
[HDP-UTILS 的下载地址](http://public-repo-1.hortonworks.com/HDP-UTILS-1.1.0.20/repos/centos6/HDP-UTILS-1.1.0.20-centos6.tar.gz)
如果直接浏览器下载速度太慢的话可以用迅雷什么的，找个账号加个速会快很多。至于账号怎么找，google 搜下一下你就知道了。另外如果不想用我上面的版本，可以替换其他的版本，地址自己摸索着改下就好。
在上面的 Apache 服务器根目录新建一个文件夹 centos-6(名字随意)，将上面的压缩包下载好之后放进 centos-6 目录中。现在你在浏览器输入 server 的IP地址加上 /centos-6 因该可以看到 centos-6 目录下面的内容了。
####创建本地 yum 源
先看下你的主机上面有没有 createrepo 工具包，执行 `which createrepo`命令，显示目录说明已经安装了，如果没有显示目录执行`yum install createrepo` 命令安装 createrepo 工具。
安装好之后输入命令 `createrepo /var/www/html/centos-6/` 创建本地 yum 源的 repo .
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

###真正开始单间 Ambari 环境

首先，把三台机器的 hosts 文件配置好。在每个主机的 /etc/hosts 文件的后面追加：
```
192.168.248.135         server.shineline.com
192.168.248.137         agent1.shineline.com
192.168.248.138         agent2.shineline.com
```
这样以后 SSH 登录之类的操作就可以直接使用域名代替 IP 地址。

####配置 server 免密登录所有的 agent
所有的 SSH 登录最好都是使用 root 用户来进行，以免有些权限不支持。
1. 在 server 的 /root/.ssh 目录下执行`ssh-keygen`命令生成密钥。
2. 执行 `ssh-copy-id root@agent1.shineline.com` 和 `ssh-copy-id root@agent2.shineline.com` 将公钥拷贝到两个 agent 的 /root.ssh/authorized_keys文件中，就能实现免密登录了，具体原理自己搜下吧。

####配置所有 agent 的 repo 使他们能连接到 server 上的额自建 yum 源
执行三次 createrepo 写三个 .repo 文件

####安装 Ambari server
执行`yum install ambari-serve`，yum 会从上面已经设置好的本地 yum 源下载 rpm 包。