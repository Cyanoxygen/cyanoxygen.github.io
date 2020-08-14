---
layout: post
title: "为运行非官方 Linux 的树莓派 4B 找回蓝牙功能"
date: 2020-08-14 17:10:42 +0800
author: Reliena
categories: [Linux, RaspberryPi]
tags: [raspberrypi, rpi, linux]
---

[former-article]:/posts/rpi-debian/

之前我在[这篇文章][former-article]描述了如何将树莓派官方的 Raspberry Pi OS 替换为 Debian GNU/Linux。但是随着我的需求变化，形同虚设的蓝牙就成了一个问题。我需要把周围的蓝牙设备（其实就是小爱同学）利用起来。经过一番研究，最终蓝牙也能工作了。

## 前言

树莓派的蓝牙不是 USB 连接的，而是连接在在 UART 上，使用 `hciattach` 将串口上的蓝牙注册为 HCI 设备。所以普通的方法并不能使蓝牙被主动发现。因为一些 Linux 发行版并不专门为树莓派设计，所以有些发行版安装完毕之后不会有树莓派相关的内容。

在对比了现有的 Debian 和官方的 Raspberry Pi OS 之后，我发现只需要 `hciattach` 就可以让蓝牙重获新生。我拷贝了 Raspberry Pi OS 的相关脚本，然后安装到我的 Debian 中，关闭串口之后蓝牙就可以工作了。

由于笔者水平有限，如果文章有不妥之处，欢迎在下方评论区留言。

## 准备工作

对于全新安装的 Debian ，你需要安装 `bluetooth` 软件包：

```shell
sudo apt install bluetooth
```

此时蓝牙需要的相应工具就会被安装。

接下来，将 Raspberry Pi OS 的自定义脚本放至相应位置。如果你没有 Raspberry Pi OS 安装，可以直接粘贴到终端，然后保存到相应位置。不要忘记赋予执行权限。

### 脚本文件

#### `/usr/bin/bthelper` 

```shell
#!/bin/sh
# For on-board BT, route SCO packets to the HCI interface (enables HFP/HSP)

set -e

if [ "$#" -ne 1 ]; then
   echo "Usage: $0 <bluetooth hci device>"
   exit 0
fi

dev="$1"
# Need to bring hci up before looking at MAC as it can be all zeros during init
/bin/hciconfig "$dev" up
/bin/hciconfig "$dev" | grep -qE "BD Address: (B8:27:EB|DC:A6:32):" || exit 0
/usr/bin/hcitool -i "$dev" cmd 0x3f 0x1c 0x01 0x02 0x00 0x01 0x01 > /dev/null
```

#### `/usr/bin/btattach` 

```shell
#!/bin/sh

HCIATTACH=/usr/bin/hciattach
if grep -q "Pi 4" /proc/device-tree/model; then
  BDADDR=
else
  SERIAL=`cat /proc/device-tree/serial-number | cut -c9-`
  B1=`echo $SERIAL | cut -c3-4`
  B2=`echo $SERIAL | cut -c5-6`
  B3=`echo $SERIAL | cut -c7-8`
  BDADDR=`printf b8:27:eb:%02x:%02x:%02x $((0x$B1 ^ 0xaa)) $((0x$B2 ^ 0xaa)) $((0x$B3 ^ 0xaa))`
fi

uart0="`cat /proc/device-tree/aliases/uart0`"
serial1="`cat /proc/device-tree/aliases/serial1`"

if [ "$uart0" = "$serial1" ] ; then
        uart0_pins="`wc -c /proc/device-tree/soc/gpio@7e200000/uart0_pins/brcm\,pins | cut -f 1 -d ' '`"
        if [ "$uart0_pins" = "16" ] ; then
              $HCIATTACH /dev/serial1 bcm43xx 3000000 flow - $BDADDR
        else
              $HCIATTACH /dev/serial1 bcm43xx 921600 noflow - $BDADDR
        fi
else
        $HCIATTACH /dev/serial1 bcm43xx 460800 noflow - $BDADDR
fi
```

### Systemd Unit File

#### `/lib/systemd/system/hciuart.service`

```ini
[Unit]
Description=Configure Bluetooth Modems connected by UART
ConditionFileNotEmpty=/proc/device-tree/soc/gpio@7e200000/bt_pins/brcm,pins
Requires=dev-serial1.device
After=dev-serial1.device

[Service]
Type=forking
ExecStart=/usr/bin/btuart

[Install]
WantedBy=multi-user.target
```

#### `/lib/systemd/system/bthelper@.service`

注意，本文件可能无法工作。由于笔者没有尝试过，所以其可行性未知。建议对文件名和文件内容做些许修改。

```ini
[Unit]
Description=Configure Bluetooth Modems connected by UART
ConditionFileNotEmpty=/proc/device-tree/soc/gpio@7e200000/bt_pins/brcm,pins
Requires=dev-serial1.device
After=dev-serial1.device

[Service]
Type=forking
ExecStart=/usr/bin/btuart

[Install]
WantedBy=multi-user.target
```

## 启用蓝牙

在完成文件创建之后，执行 `sudo systemctl daemon-reload` 来载入新的 Unit Files。对于上述 Unit Files，完全可以不用原生版本，可以自己写一个。

确保你安装了 `bluetooth` 软件包,在 `config.txt` 中没有启用串口。

执行下面的命令尝试启用蓝牙，如果你自己写了一个 Unit File，则启动相应的文件：

```shell
sudo systemctl start hciuart.service
```

如果没有报错，那么蓝牙已经打开。

此时可以执行下面的命令来查看蓝牙是否工作：

```shell
bluetoothctl list
```

此时你就可以使用 `bluetoothctl scan`, `bluetoothctl pair`, `bluetoothctl connect` 连接你的蓝牙设备了。

总之，要使蓝牙工作非常简单。本文是根据经验所写，并无实际价值。如果你想知道更多，请在 Raspberry Pi OS 中进行研究。
