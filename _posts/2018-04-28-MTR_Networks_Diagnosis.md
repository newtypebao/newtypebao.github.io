---
layout: post
title: 使用 MTR 诊断网络问题
date: 2018-04-28 
tag: network
---

最近一阵子某天的早晨，运维同事发现我们部署在某PaaS平台上的系统通过域名使用非常慢，而通过内网访问正常，于是初步怀疑是客户端到服务器域名之间的网络问题，当时平台同学给出的建议是：<span>在访问慢的时候，去ping下域名然后提供以下信息以便定位具体问题：</span><span>（1）两端IP；</span><span>（2）具体出现问题的持续时间；</span><span>（3）在两端IP上做的正反向ping和MTR测试截图。</span><span>需要针对具体的测试来判断具体问题。</span>

<span>前两个信息比较好确认，而第三点相对就比较专业点。虽然后续公司网络工程师已经发现了是网络出口方面的问题，但是我还是在网上找了一些MTR的资料，以下是我从网络上找到个人觉得还算讲述MTR比较全面的</span><span>一篇</span><span>文章，供自己和大家学习下：</span>

MTR 是一款强大的网络诊断工具，网络管理员使用 MTR 可以诊断和隔离网络问题，并且为上游 ISP 提供有用的网络状态报告。

MTR 是传统 traceroute 命令的进化版，并且可以提供强大的数据样本，因为他集合了 traceroute 和 ping 这两个命令的精华。本文带您深入了解 MTR ，从数据如何生成，到如果正确理解报告样本并得出相应的结论。

关于网络诊断技术的基本理论请参考 [network diagnostics](https://www.linode.com/docs/using-linux/administration-basics#network_diagnostics) .如果您怀疑您的 Linux 系统有其他问题，请参考 [system diagnostics](https://www.linode.com/docs/using-linux/administration-basics#system_diagnostics) 。最后，我们假定您已经掌握了 [getting started guide](https://www.linode.com/docs/getting-started/) (入门指南) 。

### 网络诊断相关的背景知识

网络诊断工具 例如 ping traceroute mtr 都使用的 “ICMP” 包来测试 Internet 两点之间的网络连接状况。当用户使用 ping 命令 ping 网络上的主机后， ICMP 包将会发送到目的主机，然后在目的主机返回响应。这样，就可以得知本机到目的主机 ICMP 包传输所使用的往返时间。

相对于其他命令仅仅收集传输路径或响应时间，MTR 工具会收集更多的信息，比如 连接状态，连接可用性，以及传输路径中主机的响应性。由于这些额外的信息，我们建议您尽可能完整的展现 Internet 两个主机之间的网络连接信息。接下来我们讲述如何安装 MTR 软件，以及如何看懂这款软件的输出结果。

### 安装 MTR

在 Debian 和 Ubuntu 系统中，使用如下命令更新系统，然后安装 MTR：

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">apt-get update
apt-get upgrade
apt-get install mtr-tiny</pre>

在 CentOS 和 Fedora 系统中，使用如下命令更新系统，并安装 MTR：

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">yum update
yum install mtr</pre>

在 Arch Linux 系统中，按照如下命令更新系统并安装 MTR：

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">pacman -Sy
pacman -S mtr</pre>

如果您的本机使用的是 Linux 系统，并且想用 MTR 测试网络状况，请按照如上教程安装。

如果您的本机使用的 Mac OS X 系统，可以使用 [Homebre](http://brew.sh/) 或 [MacPorts](http://www.macports.org/) 来安装 MTR。使用 Homebrew 安装 MTR：

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">brew install mtr</pre>

使用 MacPorts 安装 MTR：

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">port install mtr</pre>

如果您的本机使用的是 Windows 系统，您可以使用 “WinMTR”。可以从这里下载：[WinMTR](http://sourceforge.net/projects/winmtr/) .

因为 MTR 提供两个主机之间的网络路径图，您可以把它想象成一款定向工具。另外，因为地址位置或上游ISP路由器的原因，路径有时候可能会有很大的不同。所以我们建议您尽可能多的收集 MTR 的报告信息。

如果您遇到网络方面的问题， VPS的技术支持经常建议您收集双向的 MTR 报告（从 VPS 出发和到 VPS 的往返路径）。这是因为有时候网络状况从一个方向不会出现错误，但是从另一个方向会出现丢包现象。当出现网络问题时候，双向 MTR 报告是十分有用的。

在本文中，运行 mtr 命令的主机称为 源主机，被查询的主机称为 目的主机。

### 在 Unix-based 系统中使用 MTR

当在Linux 或 Mac OS X 系统中安装 MTR 后，您可以使用如下命令来产生 MTR 报告：

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">mtr -rw [destination_host]</pre>

例如，我们需要测试到 example.com 的路由信息和网络连接质量，在源主机上运行如下命令：

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">mtr -rw example.com</pre>

如果我们遇到网络问题，需要联系 VPS的技术支持，VPS 需要我们提供双向的 MTR 报告。第一份是从您的本机到 VPS 的 MTR 报告，命令如下：

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">mtr -rw 87.65.43.21</pre>

使用您的VPS 的IP地址替换 87.65.43.21 。

然后我们需要搜集从您的 VPS 返回到您的本机的 MTR 报告，命令如下：

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">mtr -rw 12.34.56.78</pre>

请使用您的本地公网IP地址替换 12.34.56.78 。如果您不确定您的本地公网IP地址，您可以使用相关的第三方服务，如果 WhatIsMyIP.com。（译者著：国内可以使用 ip.c、ip138.com 、ipip.net 等第三方服务）

> 注
> 
> 我们使用 rw 参数是为了方便让 VPS 的技术支持看到更多的网络信息。
> 
> r 产生报告信息（–report 的缩写）
> 
> w 报告中带有 hostname 信息，VPS 技术支持可以看到每一跳的完整 hostname （–report-wide 的缩写）

### 在 Windows 系统下使用 MTR

<span>首先，Windows 下的 MTR 有 GUI 的，</span><font color="#555555" face="Helvetica, Arial, Hiragino Sans GB, Microsoft YaHei, WenQuanYi Micro Hei, sans-serif"><span style="font-size: 14px;">WinMTR是老外开发的工具,集成了tracert与ping这两个命令的图形界面,使用winmtr可以直接的看到各个节点的响应时间及丢包率,适合windows下做路由追踪及PING测试,使用方法简单,WinMTR 不需安裝,解压之后即可执行。</span></font><font color="#555555" face="Helvetica, Arial, Hiragino Sans GB, Microsoft YaHei, WenQuanYi Micro Hei, sans-serif"><span style="font-size: 14px;">ping与tracert通常被用來检测网络状况和服务器状态。ping命令会送出封包到指定的服务器,如果服务器有回应就会传送回封包,另外也会告诉我们封包来回的时间。而tracert命令则是用来告诉我们从用户的电脑到指定的服务器中间一共会经过那些节点(路由)和每个节点的回应速度。下载WinMTR后,</span></font><span>运行 WinMTR，</span><font color="#555555" face="Helvetica, Arial, Hiragino Sans GB, Microsoft YaHei, WenQuanYi Micro Hei, sans-serif"><span style="font-size: 14px;">打开后,我们可以看到Host一栏的文本框,</span></font><span>输入目的地址，然后选择开始即可，您就会看到输出内容。</span>

### 如何读懂 MTR 报告

因为 MTR 报告包括了丰富的信息，新手第一次阅读有点困难。下面是我本地到 google.com 的测试报告：

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">$ mtr --report google.com
HOST: example                  Loss%   Snt   Last   Avg  Best  Wrst StDev
  1\. inner-cake                    0.0%    10    2.8   2.1   1.9   2.8   0.3
  2\. outer-cake                    0.0%    10    3.2   2.6   2.4   3.2   0.3
  3\. 68.85.118.13                  0.0%    10    9.8  12.2   8.7  18.2   3.0
  4\. po-20-ar01.absecon.nj.panjde  0.0%    10   10.2  10.4   8.9  14.2   1.6
  5\. be-30-crs01.audubon.nj.panjd  0.0%    10   10.8  12.2  10.1  16.6   1.7
  6\. pos-0-12-0-0-ar01.plainfield  0.0%    10   13.4  14.6  12.6  21.6   2.6
  7\. pos-0-6-0-0-cr01.newyork.ny.  0.0%    10   15.2  15.3  13.9  18.2   1.3
  8\. pos-0-4-0-0-pe01.111eighthav  0.0%    10   16.5  16.2  14.5  19.3   1.3
  9\. as15169-3.111eighthave.ny.ib  0.0%    10   16.0  17.1  14.2  27.7   3.9
 10\. 72.14.238.232                 0.0%    10   19.1  22.0  13.9  43.3  11.1
 11\. 209.85.241.148                0.0%    10   15.1  16.2  14.8  20.2   1.6
 12\. lga15s02-in-f104.1e100.net    0.0%    10   15.6  16.9  15.2  20.6   1.7</pre>

使用 mtr –report google.com 命令来输出这篇报告。使用 report 选项可以给 google.com 主机发送10个 ICMP 包，然后输出报告。如果我们不使用 –report 参数， mtr 会不断的动态运行。在动态模式下， mtr 的输出结果表述每个主机的往返时间。大多数情况下，使用 –report 参数就可以提供足够的数据了。

在命令下面，就是 MTR 产生的输出报告 。在通常情况下， MTR需要几秒钟的时间来输出报告，但是偶尔可能需要更长的时间。MTR 报告是由一系列跳数组成的（在上述例子中是12跳）。“跳”意味着节点，或路由器，数据包通过它们才能到达目的主机。在上面例子中，数据包经过本地网络的“内层”和“外层”，然后到达 “68.85.118.13”，然后再到一系列的域名主机。主机的域名是通过反向 DNS 查找确定的。如果您想忽略 rDNS 查找，您可以使用 –no-dns 参数，使用 –no-dns 参数后，报告结果如下：

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">%  mtr  --no-dns --report google.com
HOST: deleuze                     Loss%   Snt   Last   Avg  Best  Wrst StDev
  1\. 192.168.1.1                   0.0%    10    2.2   2.2   2.0   2.7   0.2
  2\. 68.85.118.13                  0.0%    10    8.6  11.0   8.4  17.8   3.0
  3\. 68.86.210.126                 0.0%    10    9.1  12.1   8.5  24.3   5.2
  4\. 68.86.208.22                  0.0%    10   12.2  15.1  11.7  23.4   4.4
  5\. 68.85.192.86                  0.0%    10   17.2  14.8  13.2  17.2   1.3
  6\. 68.86.90.25                   0.0%    10   14.2  16.4  14.2  20.3   1.9
  7\. 68.86.86.194                  0.0%    10   17.6  16.8  15.5  18.1   0.9
  8\. 75.149.230.194                0.0%    10   15.0  20.1  15.0  33.8   5.6
  9\. 72.14.238.232                 0.0%    10   15.6  18.7  14.1  32.8   5.9
 10\. 209.85.241.148                0.0%    10   16.3  16.9  14.7  21.2   2.2
 11\. 66.249.91.104                 0.0%    10   22.2  18.6  14.2  36.0   6.5</pre>

当我们研究 MTR 报告时候，最好找出每一跳的任何问题。除了可以查看两个服务器之间的路径之外，MTR 在它的七列数据中提供了很多有价值的数据统计报告。 Loss% 列展示了数据包在每一跳的丢失率。 Snt 列记录的多少个数据包被送出。 使用 –report 参数默认会送出10个数据包。如果使用 –report-cycles=[number-of-packets] 选项，MTR 就会按照 [number-of-packets] 指定的数量发出 ICMP 数据包。

Last, Avg, Best 和 Wrst 列都标识数据包往返的时间，使用的是毫秒（ ms ）单位表示。 Last 表示最后一个数据包所用的时间， Avg 表示评价时间， Best 和 Wrst 表示最小和最大时间。在大多数情况下，平均时间（ Avg）列需要我们特别注意。

最后一列 StDev 提供了数据包在每个主机的标准偏差。如果标准偏差越高，说明数据包在这个节点的延时越不相同。标准偏差会让您了解到平均延时是否是真的延时时间的中心点，或者测量数据受到某些问题的干扰。

例如，如果标准偏差很大，说明数据包的延迟是不确定的。一些数据包延迟很小（例如：25ms），另一些数据包延迟很大（例如：350ms）。当10个数据包全部发出后，得到的平均延迟可能是正常的，但是平均延迟是不能很好的反应实际情况的。如果标准偏差很高，使用最好和最坏的延迟来确定平均延迟是一个较好的方案。

在大多数情况下，您可以把 MTR 的输出分成三大块。根据配置，第二或第三跳一般都是您的本地 ISP，倒数第二或第三跳一般为您目的主机的ISP。中间的节点是数据包经过的路由器。

例如，我们在本地电脑运行 MTR，目的地是您的 PaaS提供者的VPS，一般前三跳属于您的本地 ISP，后三跳属于 PaaS提供者数据中心这边的。中间的条数是属于中间节点的。当您在本地运行 MTR，如果您在前几跳发现异常，请联系您本地的 ISP 服务提供商。相反，如果您在接近目的地的几跳发现问题，请联系您目的地的服务器提供商（例如：阿里云）。如果您的问题出现在中间几跳，很不幸，两边的服务提供商的能力有限，可能不能完全为您解决问题喽。

### 分析 MTR 报告

#### 核查数据包的丢失

当分析 MTR 的输出时，您需要注意两点： loss 和 latency。首先，让我们讨论一下 loss。如果您在任何一跳上看到 loss 的百分比，这就说明这一跳上可能有问题了。当然，很多服务提供商人为限制 ICMP 发送的速率，这也会导致此问题。那么如何才能指定是人为的限制 ICMP 传输 还是确定有丢包的现象？我们需要查看下一跳。如果下一跳没有丢包现象，说明上一条是人为限制的。如下示例：

root@meiriyitie.com:~# mtr –report www.google.com
HOST: example               Loss%   Snt   Last   Avg  Best  Wrst StDev
1\. 63.247.74.43                  0.0%    10    0.3   0.6   0.3   1.2   0.3
2\. 63.247.64.157                50.0%    10    0.4   1.0   0.4   6.1   1.8
3\. 209.51.130.213                0.0%    10    0.8   2.7   0.8  19.0   5.7
4\. aix.pr1.atl.google.com        0.0%    10    6.7   6.8   6.7   6.9   0.1
5\. 72.14.233.56                  0.0%    10    7.2   8.3   7.1  16.4   2.9
6\. 209.85.254.247                0.0%    10   39.1  39.4  39.1  39.7   0.2
7\. 64.233.174.46                 0.0%    10   39.6  40.4  39.4  46.9   2.3
8\. gw-in-f147.1e100.net          0.0%    10   39.6  40.5  39.5  46.7   2.2

在此例中，第二跳发生了丢包现象，但是接下来几条都没任何丢包现象，说明第二跳的丢包是人为限制的。如果在接下来的几条中都有丢包，那就可能是第二跳有问题了。请记住，ICMP 包的速率限制和丢失可能会同时发生。如果发生包的丢失情况，我们要用最低百分比来衡量时间情况。为什么这么说呢？请看如下示例：

root@meiriyitie.com:~# mtr –report www.google.com
HOST: localhost                   Loss%   Snt   Last   Avg  Best  Wrst StDev
1\. 63.247.74.43                   0.0%    10    0.3   0.6   0.3   1.2   0.3
2\. 63.247.64.157                  0.0%    10    0.4   1.0   0.4   6.1   1.8
3\. 209.51.130.213                60.0%    10    0.8   2.7   0.8  19.0   5.7
4\. aix.pr1.atl.google.com        60.0%    10    6.7   6.8   6.7   6.9   0.1
5\. 72.14.233.56                  50.0%   10    7.2   8.3   7.1  16.4   2.9
6\. 209.85.254.247                40.0%   10   39.1  39.4  39.1  39.7   0.2
7\. 64.233.174.46                 40.0%   10   39.6  40.4  39.4  46.9   2.3
8\. gw-in-f147.1e100.net          40.0%   10   39.6  40.5  39.5  46.7   2.2

在这个例子中，您可以看打 第3跳和第4跳都有 60% 的丢包率，从接下来的几跳都有丢包现象，所以不像是人为限制 ICMP 速率的原因。但是最后几跳都是40%的丢包率，我们可以猜测到60%的丢包率除了网络糟糕的原因之前还有人为限制 ICMP。所以，当我们看到不同的丢包率时，通常要以最后几跳为准。

还有很多时候问题是在数据包返回途中发生的。数据包可以成功的到达目的主机，但是返回过程中遇到“困难”了。所以，当问题发生后，我们通常需要收集反方向的 MTR 报告。

此外，互联网设施的维护或短暂的网络拥挤可能会带来短暂的丢包率，当出现短暂的10%丢包率时候，不必担心，应用层的程序会弥补这点损失。

#### 读懂网络延迟

除了可以通过 MTR 报告看到丢包率，我们还可以看到本地到目的主机之间的延时。因为不同的物理位置，延迟通常随着跳数的增加而增加。所以，延迟通常取决于节点之间的物理距离和线路质量。

例如，在同样的传输距离下，dial-up连接比cable modem连接有更大的延迟。如下示例中显示 MTR 报告：

root@meiriyitie.com:~# mtr –report www.google.com
HOST: localhost                   Loss%   Snt   Last   Avg  Best  Wrst StDev
1\. 63.247.74.43                  0.0%    10    0.3   0.6   0.3   1.2   0.3
2\. 63.247.64.157                 0.0%    10    0.4   1.0   0.4   6.1   1.8
3\. 209.51.130.213                0.0%    10    0.8   2.7   0.8  19.0   5.7
4\. aix.pr1.atl.google.com        0.0%    10  388.0 360.4 342.1 396.7   0.2
5\. 72.14.233.56                  0.0%    10  390.6 360.4 342.1 396.7   0.2
6\. 209.85.254.247                0.0%    10  391.6 360.4 342.1 396.7   0.4
7\. 64.233.174.46                 0.0%    10  391.8 360.4 342.1 396.7   2.1
8\. gw-in-f147.1e100.net          0.0%    10  392.0 360.4 342.1 396.7   1.2

在这份报告中，从第三跳到第四跳的延迟猛增，直接导致了后面的延迟也很大。这可能是因为第四跳的路由器配置不当，或者线路很拥堵的原因。

然而，高延迟并不一定意味着当前路由器有问题。这份报告虽然看到第四跳有点问题，但是数据仍然可以正常达到目的主机并且返回给主机。延迟很大的原因也有可能是在返回过程中引发的。我这份报告我们看不到返回的路径，返回的路径可能是完全不同的线路，所以我们一般要进行双向测试了。

ICMP 速率限制也可能会增加延迟，如下：

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">root@meiriyitie.com:~# mtr --report www.google.com
HOST: localhost                   Loss%   Snt   Last   Avg  Best  Wrst StDev
  1\. 63.247.74.43                  0.0%    10    0.3   0.6   0.3   1.2   0.3
  2\. 63.247.64.157                 0.0%    10    0.4   1.0   0.4   6.1   1.8
  3\. 209.51.130.213                0.0%    10    0.8   2.7   0.8  19.0   5.7
  4\. aix.pr1.atl.google.com        0.0%    10    6.7   6.8   6.7   6.9   0.1
  5\. 72.14.233.56                  0.0%    10  254.2 250.3 230.1 263.4   2.9
  6\. 209.85.254.247                0.0%    10   39.1  39.4  39.1  39.7   0.2
  7\. 64.233.174.46                 0.0%    10   39.6  40.4  39.4  46.9   2.3
  8\. gw-in-f147.1e100.net          0.0%    10   39.6  40.5  39.5  46.7   2.2</pre>

乍一看，第4跳和第5跳直接的延迟很大。但是第5跳之后，延迟又恢复正常了。最后的延迟差不多为 40ms。像这种情况，是不影响实际情况的。因为可能仅仅是第5跳设备限制了 ICMP 传输速率的原因。所以我们一般要用最后一跳的实际延迟为准。

### 常见的 MTR 报告类型

很多网络问题十分麻烦，并且需要上级网络提供商来帮助。然而，这里有很多常见的 MTR 报告所描述的网络问题。如果您正在经历一些网络问题，并且想诊断出原因，可以参考如下示例：

#### 目的主机网络配置不当

在下面这个例子中，数据包在目的地出现了 100% 的丢包。乍一看是数据包没有到达，其实未必，很有可能是路由器或主机配置不当。

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">root@meiriyitie.com:~# mtr --report www.google.com
HOST: localhost                   Loss%   Snt   Last   Avg  Best  Wrst StDev
  1\. 63.247.74.43                  0.0%    10    0.3   0.6   0.3   1.2   0.3
  2\. 63.247.64.157                 0.0%    10    0.4   1.0   0.4   6.1   1.8
  3\. 209.51.130.213                0.0%    10    0.8   2.7   0.8  19.0   5.7
  4\. aix.pr1.atl.google.com        0.0%    10    6.7   6.8   6.7   6.9   0.1
  5\. 72.14.233.56                  0.0%    10    7.2   8.3   7.1  16.4   2.9
  6\. 209.85.254.247                0.0%    10   39.1  39.4  39.1  39.7   0.2
  7\. 64.233.174.46                 0.0%    10   39.6  40.4  39.4  46.9   2.3
  8\. gw-in-f147.1e100.net         100.0    10    0.0   0.0   0.0   0.0   0.0</pre>

MTR 报告数据包没有到达目的主机是因为目的主机没有发送任何应答。这可能是目的主机防火墙的原因，例如： iptables 配置丢掉 ICMP 包所致。

#### 家庭或办公室路由器的原因

有时候家庭路由器的原因导致 MTR 报告看起来有点误导。

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">% mtr --no-dns --report google.com
HOST: deleuze                     Loss%   Snt   Last   Avg  Best  Wrst StDev
  1\. 192.168.1.1                   0.0%    10    2.2   2.2   2.0   2.7   0.2
  2\. ???                          100.0    10    8.6  11.0   8.4  17.8   3.0
  3\. 68.86.210.126                 0.0%    10    9.1  12.1   8.5  24.3   5.2
  4\. 68.86.208.22                  0.0%    10   12.2  15.1  11.7  23.4   4.4
  5\. 68.85.192.86                  0.0%    10   17.2  14.8  13.2  17.2   1.3
  6\. 68.86.90.25                   0.0%    10   14.2  16.4  14.2  20.3   1.9
  7\. 68.86.86.194                  0.0%    10   17.6  16.8  15.5  18.1   0.9
  8\. 75.149.230.194                0.0%    10   15.0  20.1  15.0  33.8   5.6
  9\. 72.14.238.232                 0.0%    10   15.6  18.7  14.1  32.8   5.9
 10\. 209.85.241.148                0.0%    10   16.3  16.9  14.7  21.2   2.2
 11\. 66.249.91.104                 0.0%    10   22.2  18.6  14.2  36.0   6.5</pre>

不要为 100% 的丢包率所吓到，这并不表明这里有问题。你可以看打在接下来几跳是没有任何丢包的。

#### 运营商的路由器没有正确配置

有时候您的运营商的路由器配置原因，导致 ICMP 包永远不能到达目的地，例如：

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">root@meiriyitie.com:~# mtr --report www.google.com
HOST: localhost                   Loss%   Snt   Last   Avg  Best  Wrst StDev
  1\. 63.247.74.43                  0.0%    10    0.3   0.6   0.3   1.2   0.3
  2\. 63.247.64.157                 0.0%    10    0.4   1.0   0.4   6.1   1.8
  3\. 209.51.130.213                0.0%    10    0.8   2.7   0.8  19.0   5.7
  4\. aix.pr1.atl.google.com        0.0%    10    6.7   6.8   6.7   6.9   0.1
  5\. ???                           0.0%    10    0.0   0.0   0.0   0.0   0.0
  6\. ???                           0.0%    10    0.0   0.0   0.0   0.0   0.0
  7\. ???                           0.0%    10    0.0   0.0   0.0   0.0   0.0
  8\. ???                           0.0%    10    0.0   0.0   0.0   0.0   0.0
  9\. ???                           0.0%    10    0.0   0.0   0.0   0.0   0.0
 10\. ???                           0.0%    10    0.0   0.0   0.0   0.0   0.0</pre>

当没有额外的路由信息时，将会显示问号（???），下面例子也一样：

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">root@meiriyitie.com:~# mtr --report www.google.com
HOST: localhost                   Loss%   Snt   Last   Avg  Best  Wrst StDev
   1\. 63.247.74.43                 0.0%    10    0.3   0.6   0.3   1.2   0.3
   2\. 63.247.64.157                0.0%    10    0.4   1.0   0.4   6.1   1.8
   3\. 209.51.130.213               0.0%    10    0.8   2.7   0.8  19.0   5.7
   4\. aix.pr1.atl.google.com       0.0%    10    6.7   6.8   6.7   6.9   0.1
   5\. 172.16.29.45                 0.0%    10    0.0   0.0   0.0   0.0   0.0
   6\. ???                          0.0%    10    0.0   0.0   0.0   0.0   0.0 
   7\. ???                          0.0%    10    0.0   0.0   0.0   0.0   0.0
   8\. ???                          0.0%    10    0.0   0.0   0.0   0.0   0.0
   9\. ???                          0.0%    10    0.0   0.0   0.0   0.0   0.0
  10\. ???                          0.0%    10    0.0   0.0   0.0   0.0   0.0</pre>

有时候，一个错误配置的路由器，将会在一个环路中不断发送数据包，如下：

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">root@meiriyitie.com:~# mtr --report www.google.com
HOST: localhost                   Loss%   Snt   Last   Avg  Best  Wrst StDev
  1\. 63.247.74.43                  0.0%    10    0.3   0.6   0.3   1.2   0.3
  2\. 63.247.64.157                 0.0%    10    0.4   1.0   0.4   6.1   1.8
  3\. 209.51.130.213                0.0%    10    0.8   2.7   0.8  19.0   5.7
  4\. aix.pr1.atl.google.com        0.0%    10    6.7   6.8   6.7   6.9   0.1
  5\. 12.34.56.79                   0.0%    10    0.0   0.0   0.0   0.0   0.0
  6\. 12.34.56.78                   0.0%    10    0.0   0.0   0.0   0.0   0.0
  7\. 12.34.56.79                   0.0%    10    0.0   0.0   0.0   0.0   0.0
  8\. 12.34.56.78                   0.0%    10    0.0   0.0   0.0   0.0   0.0
  9\. 12.34.56.79                   0.0%    10    0.0   0.0   0.0   0.0   0.0
 10\. 12.34.56.78                   0.0%    10    0.0   0.0   0.0   0.0   0.0
 11\. 12.34.56.79                   0.0%    10    0.0   0.0   0.0   0.0   0.0
 12\. ???                           0.0%    10    0.0   0.0   0.0   0.0   0.0
 13\. ???                           0.0%    10    0.0   0.0   0.0   0.0   0.0
 14\. ???                           0.0%    10    0.0   0.0   0.0   0.0   0.0</pre>

通过报告可以看打第4跳的路由器没有正确配置。如果这种状况发生了，您可以连接当地的网络管理员或ISP解决问题。

#### ICMP 速率限制

ICMP 速率限制可引起数据包的丢失。如果数据包在这一跳有丢失，但是下面几条都正常，我们可以判断是 ICMP 速率限制的原因。如下：

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">root@meiriyitie.com:~# mtr --report www.google.com
 HOST: localhost                   Loss%   Snt   Last   Avg  Best  Wrst StDev
   1\. 63.247.74.43                  0.0%    10    0.3   0.6   0.3   1.2   0.3
   2\. 63.247.64.157                 0.0%    10    0.4   1.0   0.4   6.1   1.8
   3\. 209.51.130.213                0.0%    10    0.8   2.7   0.8  19.0   5.7
   4\. aix.pr1.atl.google.com        0.0%    10    6.7   6.8   6.7   6.9   0.1
   5\. 72.14.233.56                 60.0%    10   27.2  25.3  23.1  26.4   2.9
   6\. 209.85.254.247                0.0%    10   39.1  39.4  39.1  39.7   0.2
   7\. 64.233.174.46                 0.0%    10   39.6  40.4  39.4  46.9   2.3
   8\. gw-in-f147.1e100.net          0.0%    10   39.6  40.5  39.5  46.7   2.2</pre>

这种状况是没关系的。ICMP 速率限制是一种常见的手段，这样可以减少网络数据的负载，让更重要的流量先通过。

#### 超时

在很多种情况下会发生超时现象。例如：很多路由器可能会直接丢弃 ICMP 包，这时就会导致超时（???）。
另外，也有可能在数据返回的路上出现了问题。

<pre style="box-sizing: border-box; border: 0px; font-family: monospace, serif; font-size: 15px; margin-top: 0px; margin-bottom: 24px; outline: 0px; padding: 12px; vertical-align: baseline; hyphens: none; line-height: 1.6; background: rgb(245, 247, 248); max-width: 100%; overflow: auto; white-space: pre-wrap; word-wrap: break-word;">root@meiriyitie.com:~# mtr --report www.google.com
HOST: localhost                   Loss%   Snt   Last   Avg  Best  Wrst StDev
  1\. 63.247.74.43                  0.0%    10    0.3   0.6   0.3   1.2   0.3
  2\. 63.247.64.157                 0.0%    10    0.4   1.0   0.4   6.1   1.8
  3\. 209.51.130.213                0.0%    10    0.8   2.7   0.8  19.0   5.7
  4\. aix.pr1.atl.google.com        0.0%    10    6.7   6.8   6.7   6.9   0.1
  5\. ???                           0.0%    10    7.2   8.3   7.1  16.4   2.9
  6\. ???                           0.0%    10   39.1  39.4  39.1  39.7   0.2
  7\. 64.233.174.46                 0.0%    10   39.6  40.4  39.4  46.9   2.3
  8\. gw-in-f147.1e100.net          0.0%    10   39.6  40.5  39.5  46.7   2.2</pre>

超时不一定是数据包被丢失。如上例，数据包还是安全的到达目的地并且返回。中间节点的超时可能是路由器配置丢弃 ICMP 包，或者 QoS 设置引起的原因，这个是没关系的。

### 根据您的 MTR 报告解决路由和网络问题

MTR 报告显示的路由问题大都是暂时性的。很多问题在24小时内都被解决了。大多数情况下，如果您发现了路由问题，ISP 提供商已经监视到并且正在解决中了。当您经历网络问题后，可以选择提醒您的 ISP 提供商。当联系您的提供商时，需要发送一下 MTR 报告和相关的数据。没有有用的数据，提供商是没有办法去解决问题的。

然而大多数情况下，路由问题是比较少见的。比较常见的是因为物理距离太长，或者上网高峰，导致网络变的很慢。尤其是跨越大西洋和太平洋的时候，网络有时候会变的很慢。这种情况下，建议您选择 VPS 的物理距离尽量接近您的目标客户。

如果您遇到网络连接问题，并且不能解读 MTR 报告，您可以打开一个支持工单提交问题，可以请IaS供应商工作人员会帮您分析报告。

### 更多信息：

您可以参考如下连接了解更多与 MTR 有关的知识。

[The Official MTR Web Site](http://www.bitwizard.nl/mtr/)

[Understanding the Traceroute Command – Cisco Systems](http://www.cisco.com/en/US/products/sw/iosswrel/ps1831/products_tech_note09186a00800a6057.shtml#traceroute)

[Wikipedia article on traceroute](http://en.wikipedia.org/wiki/Trace_route)

[Traceroute by Exit109.com](http://www.exit109.com/~jeremy/news/providers/traceroute.html)

