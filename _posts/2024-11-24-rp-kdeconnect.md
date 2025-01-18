---
title: Using KDE Connect on Raspberry Pi
description: Setting up and configuring KDE Connect on a headless Raspberry Pi
author: melihguclu
date: 2024-11-24 
categories: [Linux, Utilities]
tags: [shell, linux, raspberrypi, kdeconnect, en]
toc: true
lang: en
---

## Introduction and Purpose
In this post we will install KDE Connect on a headless Raspberry Pi device. Then we will learn how to control it using its Android app or another computer. 

KDE Connect is highly dependent on graphical user interfaces and dbus services. Thats why we need to perform a few additional steps during installation.


## Setup


KDE Connect uses Linux's default MDNS utility, [avahi](/posts/en/avahi), to communicate with local devices on the network. In addition, we will install the [dbus](/posts/en/dbus) package.

```bash
sudo apt update && sudo apt upgrade
sudo apt install kdeconnect avahi dbus
```
## Configuration

***kdeconnect*** service starts automatically after the installation and we need to disable it. Since it cannot find any graphical interface, it will quit.

Locating this service file might not be as straightforward as it seems. 
Therefore first

1. `systemctl --user status | grep kdeconnect`
to find its full name

2. `systemctl --user disable --now <servis-adı>`
to disable it

3. Additionally we need to configure the firewall to allow kdeconnect. [^ref1]

4. Then we will pass `-platform offscreen` argument to start KDE Connect without graphical user interface. [^ref1]

### New Service File
Lets create a new [***systemd***](/posts/en/systemd) service file using our favourite text editor.

`vim /home/melih/.config/systemd/user/kdeconnectd-offscreen.service`

```bash
[Unit]
Description=KDE Connect user service for headless setup

[Service]
Restart=always
Type=simple
WorkingDirectory=/home/melih

ExecStart=/usr/lib/arm-linux-gnueabihf/libexec/kdeconnectd -platform offscreen

[Install]
WantedBy=default.target
```
 
Then, start our new service with the following command:

`systemctl --user enable --now kdeconnectd-offscreen.service`

Although we have configured the service, we still need either SSH or monitor-keyboard pair to login and trigger the service. 

`loginctl enable-linger` command allows KDEConnect to start with the system, eliminating the need to login repeatedly.

> This might cause other similar services to start automatically.
{: .prompt-warning }

## Usage

1. `kdeconnect-cli -l` command lists other KDEConnect devices and their ID's.
2. `kdeconnect-cli --pair -d <device-id>` command sends a pair request to the other device.

### Running Commands

KDE Connect's command line interface has somewhat limited features compared to graphical user interface. Yet with a little investigation, we can understand how **"Run Command"** feature functions.

- The configuration file associated with the Run Command feature is located in **~/.config/kdeconnect/\<deviceid>/kdeconnect_runcommand/config**

- To add commands, you can use this basic tool I created.
<https://github.com/melihguclu0/rcgenerate>

```bash
[General]
 commands="@ByteArray({\"_12345678_90ab_cdef_0123_000000000000_\":{\"command\":\"/home/melih/scripts/system_temp.sh\",\"name\":\"CPUTemp\"},\"_12345678_90ab_cdef_0123_000000000001_\":{\"command\":\"/home/melih/scripts/temp.sh\",\"name\":\"temp\"},\"_e0067ef0_902b_447a_878c_000000000002_\":{\"command\":\"/home/melih/scripts/service_stat.sh\",\"name\":\"Query-Stats\"}})"
```

If we look at the first block closely:
1. *`_12345678_90ab_cdef_0123_000000000000_`*
: Is a unique identifier which specifies the command.

2. *`/home/melih/scripts/system_temp.sh`*
: The command to be executed.

3. *`CPUTemp`*
: The name of the command.

By changing this file, we can use "Run Command" feature. But the final result will be limited to the device which is specified by ***\<device_id>***. To use the "Run Command" feature for multiple devices, create symbolic links:

```bash
ln -s /home/<username>/.config/kdeconnect/<device-id>/kdeconnect_runcommand/config \
/home/<username>/.config/kdeconnect/<device2-id>/kdeconnect_runcommand/config
```

> While creating Symbolic Links, use the absolute path**(/home/\<username>/absolute/path)** instead of the relative path**(~/relative/path)**. 
{: .prompt-warning }

Finally we can run commands from our other devices. For instance: `/home/melih/scripts/system_temp.sh` script sends a notification with the CPU temperature to my phone:

```bash
kdeconnect-cli -d <device-id> --ping-msg "System Temperature is: $(vcgencmd measure_temp | cut -d'=' -f2)"
```

## Warnings

> If **NOPASSWD** is active by default, `sudo` and similar privilege escalation software will not request password. To avoid potential vulnerabilities, it is recommended to enable password authentication. [^ref2] 
{: .prompt-danger }

## References
[^ref1]: <https://userbase.kde.org/KDEConnect#Troubleshooting>
[^ref2]: <https://www.raspberrypi.com/documentation/computers/configuration.html#secure-your-raspberry-pi>
