# Raspberry Pi 4 as an Always-On Server

The Raspberry Pi 4 (4 GB) is nothing to scoff at. It has a fair bit of power and is relatively low-cost and energy-efficient. Overall, it makes an excellent always-on server for docker containers, but it requires a few tweaks to improve performance and stability. The Raspberry Pi 4's power is sometimes underestimated and many users often limit its use to single purposes. This guide will cover the deployment of a Raspberry Pi as an always-on server, providing the following services:
- **Service Management**: Portainer and Watchtower to manage your docker containers. Portainer provides a web GUI for managing docker. Watchtower automatically updates your docker containers. Organizr puts all of your services in one easy to access location.
- **Home Automation**: Home Assistant and Node-RED are automation platforms that will allow you to take control of your devices.
- **Home Monitoring**: Home Assistant, Grafana, and InfluxDB work together to visualize events.

As a general note, some installation methods in this guide involve piping a script directly to bash. This is not recommended and can be a security threat. I would encourage you to review any scripts that you run on your system carefully before executing them.

# Raspberry Pi Setup
This guide will not focus on the initial setup of a Raspberry Pi and will assume that you have a working installation of Raspberry Pi OS (previously called Raspbian).
Beyond having Raspberry Pi OS installed and configured, there are a couple of additional items for you to consider when configuring your Raspberry Pi:
- The Raspberry Pi uses an SD card as storage for its operating system and files. SD cards have poor read/write speeds and low IOPS (Input/Output Operations Per Second). SD cards are also not designed for constant writes, meaning that over time, the SD cards will likely become corrupt and your system will fail. You will find that on the Raspberry Pi 4 (4 GB), the SD card will become the main bottleneck for your system. As a result, I recommend buying a cheap SSD and running Raspberry Pi OS off of it, particularly if you are going to use services such as Home Assistant, Grafana, and InfluxDB.
- The Raspberry Pi 4 gets hot when it is under load. I recommend using a fan.

You may also want to consider installing some useful tools on your Pi with the following command:
```
sudo apt-get install zsh tmux vim nmap etherwake curl git wget htop ntfs-3g exfat-fuse exfat-utils samba samba-common-bin cec-utils
```

In particular, I recommend the following less-conventional tools:
- *tmux*: I run my Raspberry Pis headless (i.e., not connected to a monitor) and instead use SSH to facilitate managing the system. Tmux can keep your sessions alive if you lose your SSH connection.
- *etherwake*: Etherwake sends Wake On Lan packets to devices. This can enable you to use your Raspberry Pi to turn on devices on your home network (such as other servers or your personal computer).
- *cec-utils*: If your Raspberry Pi is plugged into a CEC-enabled TV or monitor, this package allows you to control the TV or monitor. I use this package in conjunction with Home Assistant to automatically control a TV.

# Common Raspberry Pi Services
There are two common services that people run on their Raspberry Pis: (1) Pi-Hole; and (2) PiVPN. I do not run them, but I will cover their installation here. Note that both of these services can alternatively be installed in Docker.

## Pi-Hole
[Pi-Hole](https://pi-hole.net/) is a DNS sinkhole that allows you to block ads on your home network. This is done by dropping DNS requests for ads. A major advantage to a solution such as Pi-Hole over a traditional browser ad-blocking add-on is that ads are blocked before they get to your device. This means that each device will expend less resources blocking ads (since it is done centrally) and that ads will be blocked on mobile devices.

You can install Pi-Hole with the following command:
```
curl -sSL https://install.pi-hole.net | bash
```
You will then need to configure your network to use the new Pi-Hole service, which is described more in [Pi-Hole's documentation](https://docs.pi-hole.net/main/post-install/).

## PiVPN
[PiVPN](https://www.pivpn.io/) is an easy-to-manage VPN server meant to be hosted on a Raspberry Pi. The VPN server will allow you to securely connect to your home network remotely.

You can install PiVPN with the following command:
```
curl -L https://install.pivpn.io | bash
```
After you have installed PiVPN, you will need to configure it and issue certificates. See [PiVPN's documentation](https://github.com/pivpn/pivpn) for more information. You will also need to forward the VPN port from your router to your Raspberry Pi. Please note that this can be a security concern.

## Why Do I Not Use Pi-Hole or PiVPN?
Although I have used both of these services in the past, and I highly recommend them, I prefer to keep my network infrastructure services on a dedicated device. The problem with using Pi-Hole and PiVPN on an undedicated Raspberry Pi is that you may experience network unavailability. For example, consider the following:
- You are using your Raspberry Pi to host Pi-Hole and a few other services. You are updating one of your other services and your device breaks, bringing down your home network's DNS service until you can bring your Raspberry Pi back online.
- You log into your home network remotely using PiVPN to administer your Raspberry Pi. An unrelated service on your Raspberry Pi breaks and your VPN connection is severed. You cannot connect back to your home network remotely now and must go in person to fix it.

With these considerations in mind, I host similar services on a dedicated system running [pfsense](https://www.pfsense.org/).

# Docker

## What is Docker and Why Do I Care?
[Docker](https://www.docker.com/) is a platform that allows you to install and run containerized applications, which are basically bundles of software and all relevant dependencies which (generally) remain separate and distinct from the rest of your system.

There are a few main advantages to running and using docker for your services:
- **Better System Stability:** This is a hard lesson that is only learned after installing several traditional services on a server (such as a VPN, pi-hole, reverse-proxy, web server, file hosting and syncing software, and home automation software), spending hours configuring the system, then having the last service you install break your operating system. With Docker, you can (quickly) set up dozens of containers and if one breaks, it (normally) doesn't take the whole system with it. Just delete the container and try again!
- **Use Software Not Packaged for your System:** There are a wide variety of Docker containers available to install. Synology only provides a limited number of packages, so Docker provides you with a huge amount of freedom.
- **Easily Defined Ports:** If you're looking to run multiple services with a web interface, you may run into the problem of conflicting ports. Normally you can configure things, but this might be tricky, difficult, or a pain. With docker, you can configure which of the host's ports are used for each container (and change them easily later).
- **Quick Deployment:** Many of these services are basically a 'one command and running' solution. There isn't any fiddling with dependencies or getting any packages to talk to each other.
- **Portability:** If you keep your containers' data stored in a persistent folder on your host system, the files can be easily copied onto a new system.
- **Somewhat Increased Security:** I mention this one last since there are arguments that could be made both ways. On the one hand, if someone compromises a container, the compromise will generally be limited to that one container. This is obviously not going to be helpful if your container has significant privileges

## Installing Docker
You can install Docker on the Raspberry Pi with the following commands. This will also enable the user "pi" to run docker commands.

```
curl -sSL https://get.docker.com | sh
sudo usermod -aG docker pi
```

To avoid file privilege issues with docker containers, you will need find the user identifier (UID) and group IDs (GID) of your user account (the one you normally use to log in with, likely `pi`). You can find these values with the following command:
```
id [username]
```
Which will reply:
```
> uid=####([username]) gid=###(users) groups=###(users),###(administrators)
```
Take note of the UID and GID for your user. For the user `pi`, these values are likely both `1000`.


# Installing Useful Services in Docker
Many docker containers can store their configuration files in a volume, so as a part of installing the container, we will make folders for each container to use as storage. I prefer to keep these configuration files in a centralized location, such as `/home/pi/Docker/[container_name]`. This is the method we will be using below.

## Portainer
[Portainer](https://www.portainer.io/) allows you to manage your docker containers via a nice web interface. Portainer provides some fine-grained control over your containers.
```
docker run --name=portainer -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v /home/pi/Docker/portainer:/data --restart always portainer/portainer
```
After running the above command, you can find the new portainer interface active at `http://hostname:9000/`. You can now use Portainer (or the default Docker manager in on your NAS) to manage your containers.

## Watchtower
[Watchtower](https://github.com/containrrr/watchtower) automatically updates your docker containers. The below command sets up a Watchtower container that monitors your containers for updates and automatically updates and restarts them with the same parameters as they were made.

You can customize when Watchtower runs, which is important because it may create some downtime for your docker services. The below command has Watchtower update the containers (including stopped containers) at 6:00 AM every morning (presumably when no one is using the containers). Refer to the [Watchtower Documentation](https://containrrr.dev/watchtower/) for customization options.
```
docker run -d --name watchtower -v /var/run/docker.sock:/var/run/docker.sock -e TZ=[Time Zone] -e WATCHTOWER_CLEANUP=true -e WATCHTOWER_SCHEDULE="0 0 6 * * *" -e WATCHTOWER_INCLUDE_STOPPED=true --restart unless-stopped containrrr/watchtower:armhf-latest
```

## Organizr
[Organizr](https://organizr.app/) is a web service that can organize all of your various services with web GUIs (such as portainer) into 'tabs', allowing you to easily access them in one central location.
```
mkdir ~/Docker/organizr
docker run -d --name=organizr -e PUID=1000 -e PGID=1000 -e TZ=[Time Zone] -p 9984:80 -v /home/pi/Docker/organizr:/config --restart unless-stopped linuxserver/organizr:arm32v7-latest
```

## Home Assistant
[Home Assistant](https://www.home-assistant.io/) is a powerful and flexible self-hosted home automation platform. Its usefulness cannot be understated. It provides monitoring services whereby Home Assistant can integrate with other devices or services and receive updates or data from them. It can also control a number of devices and services (both manually and automatically in response to certain triggers, such as the time of day or whether any devices are on the network).
```
mkdir ~/Docker/homeassistant
docker run -d --init -d --name=homeassistant -v /home/pi/Docker/homeassistant:/config -e “TZ=[Time Zone]” -e PGID=1000 -e PUID=1000 --net=host homeassistant/raspberrypi4-homeassistant
```

## Grocy
[Grocy](https://grocy.info/) is a home grocery and consumable tracking application. I use it to monitor our groceries and (in conjunction with Home Assistant), remind me when things are expiring or when groceries are running low.
```
mkdir ~/Docker/grocy
docker run -d --name=grocy -e PUID=1000 -e PGID=1000 -e TZ=[Time Zone] -p 9283:80 -v /home/pi/Docker/grocy:/config --restart unless-stopped linuxserver/grocy:arm32v7-latest
```

## Node-RED
[Node-RED](https://nodered.org/) is an automation platform that allows you to control a number of different services and devices, similar to Home Assistant. They can be used in conjunction, with Node-RED providing more of a flow-chart interface for automations.
```
mkdir ~/Docker/nodered
sudo chown -R 1000:1000 ~/Docker/nodered
docker run -d --name nodered -p 1880:1880 -v /home/pi/Docker/nodered:/data --restart unless-stopped nodered/node-red
```

## MotionEye
[MotionEye](https://github.com/ccrisan/motioneye/wiki) is an IP camera platform allowing you to monitor and control network-connected cameras. I use this on the Raspberry Pi  4 in conjunction with Home Assistant to automatically start detecting motion near my front door when certain conditions are met. For a camera, I use [this one](https://www.amazon.ca/gp/product/B07TXKMPGH/ref=ppx_yo_dt_b_asin_title_o08_s01?ie=UTF8&psc=1) from Amazon and I pass the hardware through to the MotionEye container using the following command:
```
mkdir ~/Docker/motioneye
mkdir ~/Docker/motioneye_captures
docker run -d --name=motioneye -p 8765:8765 -p 8081:8081 --hostname="motioneye" -v /etc/localtime:/etc/localtime:ro -v /home/pi/Docker/motioneye:/etc/motioneye --device /dev/vchiq -v /home/pi/Docker/motioneye_captures:/var/lib/motioneye --restart unless-stopped --detach=true ccrisan/motioneye:master-armhf
```

## InfluxDB
[InfluxDB](https://www.influxdata.com/) InfluxDB is a database that can be used in conjunction with Home Assistant and Grafana. Personally, I have Home Assistant writing data to InfluxDB so that any monitoring data generated by Home Assistant can be easily used by other services, such as Grafana.
```
mkdir ~/Docker/influxdb
docker run -d --name=influxdb --volume=/home/pi/Docker/influxdb:/data -p 8086:8086 --restart unless-stopped hypriot/rpi-influxdb
```

## Grafana
[Grafana](https://grafana.com/) is a visualization platform that creates graphs out of data. I use it in conjunction with InfluxDB to generate graphs about items such as system performance, network performance, weather, and storage space.
```
mkdir ~/Docker/grafana
docker run -d --name=grafana -p 3000:3000 -v /home/pi/Docker/grafana:/var/lib/grafana -e GF_SECURITY_ALLOW_EMBEDDING=true --restart unless-stopped grafana/grafana
```

## FreshRSS
[FreshRSS](https://freshrss.org/) is a self-hosted RSS reader.
```
mkdir ~/Docker/freshrss
docker run -d --name=freshrss -e PUID=1000 -e PGID=1000 -e TZ=[Time Zone] -p 9008:80 -v /home/pi/Docker/freshrss:/config --restart unless-stopped linuxserver/freshrss:arm32v7-latest
```
