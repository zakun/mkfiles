## CentOS设置 SAMBA

> 中文参考: https://wiki.centos.org/zh/HowTos/SetUpSamba

> 英文参考: https://wiki.centos.org/HowTos/SetUpSamba

### 概述 
> Samba是一个能让Linux系统应用Microsoft网络通讯协议的软件，SMB（Server Message Block）服务器消息块
> Samba最大的功能是可以用于Linux与windows系统直接的文件共享和打印共享，既可以用于Windows与linux之间的文件共享也可以用于linux与linux之间的资源共享
> 基于客户机/服务器的协议，因而一台Samba服务器既可以充当文件共享服务器，也可以充当一个Samba客户端
> Samba在windows下使用的是NetBIOS协议，要使用linux下共享出来的文件，要确认windows系统安装了NetBIOS协议


由于防火墙（iptables）及 SELinux 保护，在 CentOS 上设置 Samba 是比较难。这其实是一件好事，因为安全性是非常重要，但要令 Samba 与服务器的外界沟通，我们需要花点功夫及对它有所认识。

SAMBA 采用端口 137 — 139 及 445。为什么会是这些端口？让我们看得再详细一点。微软的旧款主从通讯协议是 netbios。它的端口是 137、138 及 139。这款网络设置依赖一台 netbios 服务器（WINS）［Windows Internet Naming Service］来提供传送给客户端的名称。换句话说，WINS 就是昔日的 DNS。它可以是一项非常不安全的服务，但设置却很容易。如果你曾经采用过 net view 或 net use 这些指令，你便是使用 WINS 服务。

时移世易，WINS 被新进的 Active Directory（AD）所淘汰。这是微软对 Novell 的 Networking 服务（NDS）的反击，而它又是 Novell 对 UNIX 的 NFS 服务器的反应。最大的分别就是 Active Directory 依赖一台 DNS 服务器而不是 netbios。这意味着端口上的改动。微软的 AD 服务改用 445 号端口（UDP 及 TCP）。现时，除非你就这个服务拥有一台 Windows 服务器，否则 Windows 缺省会采用 netbios，而你必须通过控制台内的「系统」图示进行特别设置才能更改它。如果你不会采用 Netbios，你亦可以在 Windows 控制台的「网络连接」图示下的 tcp/ip 服务来停用它。再一次，这些设置都超越了这份文档的范畴。

现在让我们再次返回那些端口及 Samba。以下的清单详细列出 Samba 在你的系统上运行时所需的端口。请注意一个 TCP 端口只是个服务端口。80 号端口是供网页使用，而 22 号端口是供安全远程连接（ssh）使用。UDP（用户定义端口）是变种的 TCP 端口。这篇文章不会以解释当中的分别作为焦点，但你可以轻易地在网上寻找。现时你只需知道它们是有分别的。

（旧的系统）

	137 号端口 —— UDP NetBIOS 命名服务（WINS）
	138 号端口 —— UDP NetBIOS 数据包
	139 号端口 —— TCP NetBIOS 工作阶段（TCP）、Windows 文件及打印机共享（这是最不安全的端口）
（Active Directory）

	445 号端口 —— Microsoft-DS Active Directory、Windows 共享资源（TCP）
	445 号端口 —— Microsoft-DS SMB 文件共享（UDP）
为何要知道这一切呢？因为你须要知道为 SAMBA 打开及不要打开哪些端口，否则你便不能令它在 CentOS 上运作。

### 1. 放宽防火墙
请进到你的防火墙文件 `/etc/sysconfig/iptables。`

请利用你所喜欢的文字编辑器（例如 vi、joe、或任何适用的程序）按实际情况在这个文件加入下列数行。

如果你采用 Active Directory 并且想在 Samba 内单单启用这个功能。

	-A RH-Firewall-1-INPUT -s 192.168.10.0/24 -m state --state NEW -m tcp -p tcp --dport 445 -j ACCEPT
	-A RH-Firewall-1-INPUT -s 192.168.10.0/24 -m state --state NEW -m udp -p udp --dport 445 -j ACCEPT
请不要被词法吓怕。我不会涵盖整个防火墙，但你可理解一些基础。

- -s （ip 地址） 限定你的安装上的 C 级 IP 地址。当然你可以按你的需要将它改至乎合你的网络，这样比开放你的网络给整个世界访问更为安全。

- --state NEW ［基本上是开始新规则之意。］

- -p ［你想打开的 tcp 或 udp 端口。我已经为你下了功夫，因此你不必推断该打开哪种端口］

- dport 445 ［这是端口号。再一次我们针对 AD 采用 445 号端口。］

现在，如果你的 Samba 设置须要呼唤旧款的 netbios：

	-A RH-Firewall-1-INPUT -s 192.168.10.0/24 -m state --state NEW -m udp -p udp --dport 137 -j ACCEPT
	-A RH-Firewall-1-INPUT -s 192.168.10.0/24 -m state --state NEW -m udp -p udp --dport 138 -j ACCEPT
	-A RH-Firewall-1-INPUT -s 192.168.10.0/24 -m state --state NEW -m tcp -p tcp --dport 139 -j ACCEPT
请留意尺寸写，并分辨 `tcp` 和 `udp`，否则 samba 便会不能正常运作。这些都必须是正确的 —— 我便是从错字里学回来！

现在重新打开防火墙。你可遁两个途径在 CentOS 下重新引导服务

1. service iptables restart
2. /etc/init.d/iptables restart

任何一个方法都可行。你喜欢的话，亦可以重新引导该台服务器。

`注：`你可以利用 Redhat 的系统工具来编辑防火墙，但我们不推荐如此做。它不会加入 -s 这个参数，并会打开所有 samba 端口 137 — 139 和 445，是我们不提倡的局面。

### 2. SELinux
希望你跟得上！接下来我们需要安抚 SELinux。SELinux 会基于安全理由自动阻止任何共享资源被浏览。众多个解决方案中的等一个就是弄掉 SELinux —— 坏主意。其实要与这个优良的安全性功能合作并不困难。

`setsebool` 这个指令可以打开或关闭 SELinux 的保障。你可以通过 `getsebool -a` 来取得一份完整的清单。这是由于较新的版本会增加更多安全性的功能。

	getsebool -a | grep samba
	getsebool -a | grep smb
这样大致上会为你列出所有 samba 选项，至少是最重要的那一些。留意我们利用 grep 这个指令进行过滤。如果你从未使用过 grep，你错过了不少。grep 是在 linux 下一个值得认识的奇妙工具。

如果你想 samba 作为本地控制站：

	setsebool -P samba_domain_controller on
如果你想分享缺省的主目录，请输入这个指令：

	setsebool -P samba_enable_home_dirs on
截至 CentOS 5.3 这是你所须做的一切。现在我们要利用 semanage 这个指令（SELinux 组件内的一部份）来打开你期望在网络上分享的目录。真的。没有这样做，当你引导 samba 时只会获得一堆空置的目录，导致你误信服务器已经删除你的所有数据！

	semanage fcontext -a -t samba_share_t '/<共享目录>(/.*)?'
	restorecon -R /<共享目录>
有关主目录的备注：

主目录的意思就是唯有你 —— 用户本人 —— 才能连接到你的主目录。亳无疑问这个主目录可以成为一个流动组态并且存储用户所创建的文件。同样地，你亦可以用 useradd 这个指令来定义主目录到任何位置。容让我继续探讨在这个问题。

试想像以下情况。你想去掉操作系统。你希望连同主目录一并保留你的数据。可能吗？可能，而且做法是这样。

### 3. 创建数据分区
请在硬盘上创建另一个分区，然后挂载这个分区及将它分享出来。再一次，个中原因就是要将它与系统的文件分隔开，好让当你需要重装操作系统时，这个数据扇区不会受你的动作所影响。（我们有这个经历）另外请留意整个 CentOS 操作系统只占用很少的空间。在这个样例里我们为操作系统保留 12G（足够有余）并利用（/）这个挂载点。然后我们为开机（/boot）保留 100M，并将余下的空间变为共享资源（/data）。然后我们将 /data 挂载在挂载目录（/mnt/data）上。在大容量的硬盘上，你可以为客户端提供一个庞大的 data 目录。

让我们应用一个样例。假设我们在 /dev/hda（首个硬盘）上创建了另一个分区并称它为 /data。以下是内中的步骤

	mkdir /mnt/data
	mount /dev/hd3 /mnt/data
（你当然可以随自己的首选为这个目录命名。这注意在我的样例中它是第三个分区 /dev/hd3。视乎你如何设由置你的分区，它在你的硬盘上可能会在不同位置。你可以利用 fdisk /dev/hda 及 p 这个命令来取得分区清单。如果你从未使用过 fdisk，请先在一个测试用的硬盘上进行实验。fdisk 是有点儿……危险）

现在请应用 semanage 这个指令在该目录上。

	semanage fcontext -a -t samba_share_t '/mnt/data(/.*)?'
	restorecon -R /mnt/data
如此，不论整个内的目录的路径有多深，它们的 SELinux 访问权都会被修改。请留意 -R 是递回的意思，你也可以在 rm 及 cp 等很多指令里采用它。

此刻你应该为 /data 这个分区设置权限及拥有者。这部份由你来决定。如果时间紧迫的话，你可以抉捷、不择手段地完成这件事情。有关 chmod 及 chown 的细节，请参阅它们的使用说明，因为这些都超越了本篇文章的范畴。

	chmod 770 -R /mnt/data
	chmod -R root:(主群组名称) /mnt/data
这样会将访问整个碟盘所有权限开放给它的拥有者及该群组。你可以此作为起点，按你的需要作出修订。下一节将会告诉你如何在新分区上设置该用户及相关的主目录。

### 4. 新增用户
既然我们已经办妥安全措施，是时候加入户用了。在这个样例里，我将会创建一位名叫 dave 的用户（碰巧这是我的名字）

	useradd dave -d /mnt/data/home/dave
**（请留意 -d 这个选项）。它会在新的数据碟盘上创建我的主目录，离开操作系统。这样你便可两全其美。**

我们推荐你为每位用户设置以下权限。

	chown (用户): (用户) /mnt/data/home/(用户)
	chown dave:dave /mnt/data/home/dave
这些都应该由 useradd 这个指令所设置，但你仍可检查它是否正确。至于用户的访问权便再次由你作主。Ubuntu 有一个不错的理念。所有用户的权限都是以 chmod 640 来设置。

	passwd dave (你选用的密码)
	smbpasswd -a dave
（-a 的意思就是将它加进数据库。当然，请勿用 -a 来修改一个现存的用户）如果这是首位用户，你的划面将会出现错误信息。问题不大。它只是在创建一个新的数据库，而你不会再看见这些信息。

最后一个行动（smbpasswd）将密码加进 smbpasswd 数据库内。这个密码档随着时间而有所更改。现时它名叫 passtb.tdb，但它先前名叫 smbpasswd。你可以想像一个指令与文件拥有相同名称所带来的混乱！

取后，请再新引导 smb

	service smb restart (或者 /etc/init.d/smb restart)
这样便会运用新的改动。现在你可以你的安装上设置 samba。外面有多不胜数的文章关于这个题目，本人曾考虑新增一篇 samba 的文章，但一篇短文实在不能涵盖如此丰富的主题。你应该开始着手做了！