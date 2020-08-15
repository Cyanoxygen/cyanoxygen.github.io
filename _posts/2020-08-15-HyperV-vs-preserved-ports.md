---
layout: post
title: "干掉 Windows 下 Hyper-V HNS 服务的预留端口"
date: 2020-08-15 01:43:11 +0800
author: Reliena
categories: [Windows, Network]
tags: [windows, eacces, network, hyperv]
---

这是一篇关于 Hyper-V 端口预留过多导致大部分程序无法使用的问题的文章。~~我已经弃用了 WSL 2 和 Hyper-V~~。

## 出现问题到发现问题

自从将 Windows 10 升级到了 2004，并启用了 WSL 2 之后，我电脑里的一些程序和游戏就突然无法工作。
起初一直以为是玄学问题，直到有一天我发现我的一个 Web 始终没有跑起来。经过一番排查，发现它始终无法绑定端口。  
用 `netstat -abn` 也没有发现该端口有被占用的迹象。

于是 `python3 -m http.server 1234`，Server 没有跑起来，抛了个异常就溜之大吉，错误码是 `WSAEACCES` :

```
OSError: [WinError 10013] An attempt was made to access a socket in a way forbidden by its access permissions
```

换了好几个端口，其中有大部分端口都无法绑定。为了查看有多少端口不能用，我简单写了一个[脚本](https://gist.github.com/Cyanoxygen/64938614406508aa1ee0db266d2625e8)，当时的测试结果为：

![运行结果 1](/assets/img/posts/porttest_1.jpg)

在几个小时的搜索之后，发现 [Stack Overflow 上也有人出现了相同的问题](https://stackoverflow.com/a/54727281)。该问题指出，使用 Docker for Windows 时不能绑定端口，无法正常使用容器。  
其中优质回答将问题矛头直指 Hyper-V，引用了一个 [GitHub issue](https://github.com/docker/for-win/issues/3171#issuecomment-459205576)，表示 Hyper-V 在启用之后向系统预留了一大堆端口，可以使用下面的命令查看：

```
netsh int ipv4 show excludedportrange protocol=tcp
```

执行了一遍，发现脚本报告的不能绑定的端口并不是全部：

![Hyper-V 预留的端口](/assets/img/posts/reserved_port_1.jpg)

Hyper-V 你在干嘛？预留这么多端口作甚？你要知道我们都没法正常用电脑了！

## 解决方法？

根据 GitHub 和 Stack Overflow 上回答的描述，总结了一下可能的解决方法。这些解决方案均来自 GitHub 和其他网站，不保证其可用性。

下面列出的方法不一定是最优解。我选择了彻底放弃 Hyper-V 和 WSL 2，因为实在是太麻烦了，而且和 Hyper-V 共存下的虚拟机软件效率低得惊人。

### 禁用 Hyper-V ，手动预留端口并重新启用

禁用 Hyper-V ，手动预留*你想要使用的端口*之后再重新启用，此时 Hyper-V 预留端口时就动不到你预留过的端口了。

举个例子：假设你的常用端口在 `1234` - `2345` 之间，你需要在禁用 Hyper-V 之后手动预留这个范围的端口，然后重新启用 Hyper-V 时 Hyper-V 就不会再抢占这些端口了。

要禁用 Hyper-V，你可以使用 `控制面板` - `程序` - `打开或关闭 Windows 功能` ，取消勾选下面的所有项目：

- Hyper-V
- Windows Hypervisior Platform
- Virtual Machine Platform

或者使用命令行，如果你只开启了 Hyper-V：

```
dism.exe /Online /Disable-Feature:Microsoft-Hyper-V
```

禁用完成会提示重启。重启电脑，然后预留你经常使用的端口：

假设你的常用端口范围是 `1234` - `2345`：

```
netsh int ipv4 add excludedportrange protocol=tcp startport=1234 numberofports=1112
```

这表示预留从 `1234` 端口开始的 1112 个端口，也就是 `1234` - `2345`。  
如果你有很多段端口要使用，可以重复执行上述命令。

在你完成全部操作之后，可以返回原路重新启用 Hyper-V 及其相关组件了。在这之后，你的端口使用将不会再受到影响。

不过 GitHub issue 那边有人指出这个操作会导致虚拟网卡不工作。

### 用注册表更改动态端口范围为不常用范围

在 Stack Overflow的优质回答下面，有一个[文章链接](https://dandini.wordpress.com/2019/07/15/administered-port-exclusions-blocking-high-ports/)，该文章讲述了更改动态端口范围之后 Hyper-V 也会相应更改独占端口范围。

另外文章中还有这么一句话：

> Following a support call with Microsoft we were informed of an entirely (at time of writing) undocumented registry key ‘EnableExcludedPortRange’ to disable the excluded port range.

使用注册表键 `EnableExcludedPortRange` 似乎可以禁用独占端口。

不过测试系统是 Windows 10 1809，可能对新版本不适用。

文章里有一个脚本，脚本检测 Hyper-V 开启之后即修改动态端口范围，你可以根据需求选择需要的部分执行：

```
rem Modify Dynamic Port Range for Development Users
dism /online /get-features | find /i "Microsoft-Hyper-V" && (
rem Modify Dynamic Port Range
start /wait "" netsh int ipv4 set dynamicport tcp start=20000 num=16384
start /wait "" netsh int ipv4 set dynamicport udp start=20000 num=16384

rem Add Registry Key
start /wait "" reg add HKLM\SYSTEM\CurrentControlSet\Services\hns\State /v EnableExcludedPortRange /d 0 /f

goto :eof

)
```

如果你想撤销更改，或者你已不再使用 Hyper-V，可以使用脚本的另一部分还原设置：

```
rem Set range to default
start /wait "" netsh int ipv4 set dynamicport tcp start=49152 num=16384
start /wait "" netsh int ipv4 set dynamicport udp start=49152 num=16384

rem Remove Registry Key
start /wait "" reg delete HKLM\SYSTEM\CurrentControlSet\Services\hns\State /v EnableExcludedPortRange /f
```

在这些步骤完成之后重启你的电脑，HNS 应该不会再霸占你需要的端口了。

---

Hyper-V 的端口预留为开发者造成了不少麻烦。为了 WSL 2，不得不使用 Hyper-V，没想到居然会带出这么多问题来。这些解决方案均来自 GitHub 和其他网站，不保证其可用性，因为我已经完全弃用了 Hyper-V/WSL 2，尽管在禁用之后还有两个虚拟网卡没有删除。

