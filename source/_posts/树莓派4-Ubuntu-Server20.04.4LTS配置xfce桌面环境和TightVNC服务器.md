---
title: 树莓派4-Ubuntu-Server20.04.4LTS配置xfce桌面环境和TightVNC服务器
date: 2022-06-09 16:25:51
categories: Raspberry Pi
tags: [Raspberry Pi]
toc: true
description: 好きなことに 出会えただけで、 幸せだ。 好きなことに 出会えない人 も大勢いるんだ。
---

# 步骤1 – 安装桌面环境和VNC服务器

默认情况下，Ubuntu Server 20.04没有安装图形桌面环境或VNC服务器，所以我们首先安装它们。具体来说，我们将为最新的Xfce桌面环境和官方Ubuntu存储库中提供的TightVNC软件包安装软件包。

在您的服务器上，更新您的包列表：

```shell
sudo apt update
```

现在在您的服务器上安装Xfce桌面环境：

```shell
sudo apt install xfce4 xfce4-goodies
```

安装完成后，安装TightVNC服务器：

```shell
sudo apt install tightvncserver
```

要在安装后完成VNC服务器的初始配置，请使用该vncserver命令设置安全密码并创建初始配置文件：

```shell
vncserver
```

系统将提示您输入并验证密码以远程访问您的计算机：

```shell
You will require a password to access your desktops.
Password:
Verify:
```

密码长度必须介于六到八个字符之间。超过8个字符的密码将自动截断。

验证密码后，您可以选择创建仅查看密码。使用仅查看密码登录的用户将无法使用鼠标或键盘控制VNC实例。如果您想使用VNC服务器向其他人演示内容，这是一个有用的选项，但这不是必需的。

然后，该过程为服务器创建必要的默认配置文件和连接信息：

```shell
Would you like to enter a view-only password (y/n)? n
xauth:  file /home/sammy/.Xauthority does not exist

New 'X' desktop is your_hostname:1

Creating default startup script /home/sammy/.vnc/xstartup
Starting applications specified in /home/sammy/.vnc/xstartup
Log file is /home/sammy/.vnc/your_hostname:1.log
```

现在让我们配置VNC服务器。

# 步骤2 – 配置VNC服务器VNC服务器

需要知道启动时要执行的命令。具体来说，VNC需要知道它应该连接到哪个图形桌面。

这些命令位于主目录下xstartup的.vnc文件夹中调用的配置文件中。启动脚本是vncserver在上一步中运行时创建的，但我们将创建自己的脚本以启动Xfce桌面。

首次设置VNC时，它会在端口5901上启动默认服务器实例。该端口称为显示端口，由VNC称为:1。VNC可以在其他显示端口上启动多个实例，例如:2，:3等等。

因为我们将要更改VNC服务器的配置方式，所以首先5901使用以下命令停止在端口上运行的VNC服务器实例：

```shell
vncserver -kill :1
```

输出应该如下所示，尽管您会看到不同的PID：

```shell
Killing Xtightvnc process ID 17648
```

在修改xstartup文件之前，请备份原始文件：

```shell
mv ~/.vnc/xstartup ~/.vnc/xstartup.bak
```

现在创建一个新xstartup文件并在文本编辑器中打开它：

```shell
nano ~/.vnc/xstartup
```

无论何时启动或重新启动VNC服务器，都会自动执行此文件中的命令。如果尚未启动，我们需要VNC启动我们的桌面环境。将这些命令添加到文件中：

```shell
# !/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
startxfce4 &

[ -x /etc/vnc/xstartup ] && exec /etc/vnc/xstartup
[ -r $HOME/.Xresources ] && xrdb $HOME/.Xresources
xsetroot -solid grey
```

文件中的第一个命令不去研究了。第二个命令告诉服务器启动Xfce，在这里您可以找到舒适地管理服务器所需的所有图形软件。

为确保VNC服务器能够正确使用此新启动文件，我们需要使其可执行。

```shell
sudo chmod +x ~/.vnc/xstartup
```

现在，重新启动VNC服务器。

```shell
vncserver
```

您将看到类似于此的输出：

```shell
New 'X' desktop is your_hostname:1

Starting applications specified in /home/sammy/.vnc/xstartup
Log file is /home/sammy/.vnc/your_hostname:1.log
```

配置到位后，让我们从本地计算机连接到服务器。

# 步骤3 – 安全地连接VNC桌面

连接时VNC本身不使用安全协议。我们将使用SSH隧道安全地连接到我们的服务器，然后告诉我们的VNC客户端使用该隧道而不是直接连接。

在本地计算机上创建SSH连接，以便安全地转发到localhostVNC连接。您可以使用以下命令通过Linux或macOS上的终端执行此操作：

```shell
ssh -L 5901:127.0.0.1:5901 -C -N -l sammy your_server_ip
```

该-L开关指定的端口绑定。在这种情况下，我们将5901远程连接的端口5901绑定到本地计算机上的端口。该-C开关启用压缩，而-N开关告诉ssh我们不希望执行远程命令。该-l开关指定远程登录名。

记得替换sammy，并your_server_ip与您的服务器的须藤非root用户名和IP地址。

如果您使用的是图形化SSH客户端（如PuTTY），请将your_server_ip用作连接IP，并在程序的SSH隧道设置中设置localhost:5901为新的转发端口。

隧道运行后，使用VNC客户端进行连接localhost:5901。系统将提示您使用在步骤1中设置的密码进行身份验证。

接下来让我们将VNC服务器设置为服务。

# 步骤4 – 将VNC作为系统服务运行

接下来，我们将VNC服务器设置为systemd服务，以便我们可以根据需要启动，停止和重新启动它，就像任何其他服务一样。这还将确保在服务器重新启动时VNC启动。

首先，使用您喜欢的文本编辑器创建一个新的`/etc/systemd/system/vncserver@.service`单元文件：

```shell
sudo nano /etc/systemd/system/vncserver@.service
```

@名称末尾的符号将让我们传入一个我们可以在服务配置中使用的参数。我们将使用它来指定我们在管理服务时要使用的VNC显示端口。

将以下行添加到该文件中。请务必更改`用户`，`组`，`WorkingDirectory`的值以及`PIDFILE`值中的用户名以匹配您的用户名：

```shell
[Unit]
Description=Start TightVNC server at startup
After=syslog.target network.target
[Service]
Type=forking
User=sammy
Group=sammy
WorkingDirectory=/home/sammy

PIDFile=/home/sammy/.vnc/%H:%i.pid
ExecStartPre=-/usr/bin/vncserver -kill :%i > /dev/null 2>&1
ExecStart=/usr/bin/vncserver -depth 24 -geometry 1280x800 :%i
ExecStop=/usr/bin/vncserver -kill :%i

[Install]
WantedBy=multi-user.target
```

如果VNC已经运行，该ExecStartPre命令将停止。该ExecStart命令启动VNC并将颜色深度设置为24位颜色，分辨率为1280×800。您也可以修改这些启动选项以满足您的需求。

保存并关闭文件。

接下来，让系统知道新的单元文件。

```shell
sudo systemctl daemon-reload
```

启用单元文件。

```shell
sudo systemctl enable vncserver@1.service
```

在1以下的@符号表示，其显示编号的服务应该出现过，在这种情况下，默认:1为在步骤2中进行了讨论..

如果VNC服务器仍然在运行，请停止它的当前实例。

```shell
vncserver -kill :1
```

然后启动它，就像启动任何其他systemd服务一样。

```shell
sudo systemctl start vncserver@1
```

您可以使用此命令验证它是否已启动：

```shell
sudo systemctl status vncserver@1
```

如果它正确启动，输出应如下所示：

```shell
● vncserver@1.service - Start TightVNC server at startup
   Loaded: loaded (/etc/systemd/system/vncserver@.service; indirect; vendor preset: enabled)   
   Active: active (running) since Mon 2018-07-09 18:13:53 UTC; 2min 14s ago  
  Process: 22322 ExecStart=/usr/bin/vncserver -depth 24 -geometry 1280x800 :1 (code=exited, status=0/SUCCESS)   
  Process: 22316 ExecStartPre=/usr/bin/vncserver -kill :1 > /dev/null 2>&1 (code=exited, status=0/SUCCESS) Main PID: 22330 (Xtightvnc)
  
...
```

重新启动计算机后，您的VNC服务器现在可用。

再次启动SSH隧道：

```shell
ssh -L 5901:127.0.0.1:5901 -C -N -l sammy your_server_ip
```

然后使用VNC客户端软件建立新连接localhost:5901以连接到您的计算机。

# Reference

[安装配置TightVNC和xfce4](http://www.fromars.com/tightvnc_xfce4.html)
[Remote GUI access to a Linux computer using Tightvnc with systemd](http://www.penguintutor.com/raspberrypi/tightvnc)