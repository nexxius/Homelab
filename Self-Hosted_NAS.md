# Self-Hosted NAS with the Synology DS920+

Eventually, running all of your services and file/media hosting on a Raspberry Pi gets to be a bit limiting. Synology's network attached storage (NAS) devices provide a useful and easy-to-setup server for people looking to host your files locally. There are tons of guides on getting your server set up. This guide is meant to provide a guide to more advanced features you can set up on your Synology NAS.

I am personally running a [Synology DS920+](https://www.synology.com/en-us/products/DS920+), but many aspects of this guide are general and may apply to other models (particularly plus models).

## Initial Setup
Although this guide will not cover the initial basic setup of your NAS, some useful settings you may wish to consider tweaking are:
- Enabling WOL (found in the 'Hardware & Power' Control Panel Page).
- Disabling access to your NAS directly from the internet (including by disabling QuickConnect). If external access to your NAS is required, this should only be done through a VPN or via a reverse-proxy.
- Give your NAS a static IP. This can be done on your NAS itself, but I prefer to set a static IP on the router itself. This provides a one-stop interface for managing the static IPs on your network, and also creates less hassle if you don't own your router and need to move your NAS to a different network (such as if you move).
- Run the Security Advisor application and check to see if there are any security concerns. You will have the opportunity to set a baseline for what will constitute a security concern. I recommend running this on a scheduled/regular basis.

## Docker

### What is Docker and Why Do I Care?
[Docker](https://www.docker.com/) is a platform that allows you to install and run containerized applications, which are basically bundles of software and all relevant dependencies which (generally) remain separate and distinct from the rest of your system.

There are a few main advantages to running and using docker for your services:
- **Better System Stability:** This is a hard lesson that is only learned after installing several traditional services on a server (such as a VPN, pi-hole, reverse-proxy, web server, file hosting and syncing software, and home automation software), spending hours configuring the system, then having the last service you install break your operating system. With Docker, you can (quickly) set up dozens of containers and if one breaks, it (normally) doesn't take the whole system with it. Just delete the container and try again!
- **Use Software Not Packaged for your System:** There are a wide variety of Docker containers available to install. Synology only provides a limited number of packages, so Docker provides you with a huge amount of freedom.
- **Easily Defined Ports:** If you're looking to run multiple services with a web interface, you may run into the problem of conflicting ports. Normally you can configure things, but this might be tricky, difficult, or a pain. With docker, you can configure which of the host's ports are used for each container (and change them easily later).
- **Quick Deployment:** Many of these services are basically a 'one command and running' solution. There isn't any fiddling with dependencies or getting any packages to talk to each other.
- **Portability:** If you keep your containers' data stored in a persistent folder on your host system, the files can be easily copied onto a new system.
- **Somewhat Increased Security:** I mention this one last since there are arguments that could be made both ways. On the one hand, if someone compromises a container, the compromise will generally be limited to that one container. This is obviously not going to be helpful if your container has significant privileges

### Setting Up Docker
- **Installing Docker:** To install Docker, log into your Synology NAS and open "Package Center". Then browse to "All Packages" and select "Docker" for installation. This should also result in the creation of a new "docker" Shared Folder, which we will use later with our images.
- **Enabling SSH:** You will likely want to enable SSH to allow you to run docker commands. To do this, log into your Synology NAS and open "Control Panel", select "Terminal & SNMP", then check the box beside "Enable SSH service". Note that this will make your server less secure. You can change the port number that SSH is running on (the default is 22), but this is essentially "security through obscurity" and is not a material increase in security. If SSH is a serious security concern for you, you can enable/disable SSH as needed.
- **Logging Into SSH:** You can now SSH into your Synology NAS. Most linux computers will have an SSH client installed by default. If you are on a Windows computer, you can use [PuTTY](https://www.putty.org/). I am personally a fan of [Terminus](https://eugeny.github.io/terminus/), which has a built-in SSH client. Once you have your SSH client set up, log into your NAS using your username, hostname, and password. From the Linux command line, this looks like:
    `ssh [username]@[host]`
- **Make The Relevant Folders:** Generally, you will want to store the files for your docker containers in a folder that can persist in the event you delete your container. This allows your configurations to easily survive deleting the container. I normally store these folders in the `docker` Shared Folder that was created by docker. Assuming your volume is `volume1`, I would create these folders in `/volume1/docker/[containername]`.
- **Use Sudo:** You will need root privileges to run docker commands. You can get a root shell by running the following command and entering your password when prompted:
    `sudo -i`
- **Get Your UID and GID:** To avoid file privilege issues with docker containers, you will need the user identifier (UID) and group IDs (GID) of your user account (the one you normally use to log in and use your Synology NAS). You can find these values with the following command:
    ```
    id [username]
    ```
    Which will reply:
    ```
    > uid=####([username]) gid=###(users) groups=###(users),###(administrators)
    ```
    Take note of the UID for your user and the GID for the administrators group.

### Useful Docker Containers
The following containers will get you started on Docker. Most of these docker containers are from [LinuxServer.io](https://docs.linuxserver.io/). I highly recommend reading through their documentation to get familiar with docker's syntax, as well as to get familiar with what each of the switches and arguments mean.

**Make sure you tweak each command for your own needs.** You will need to customize the file locations, time zone, PUID, and GUID, as well as any other information below that is user-specific. Failure to do so may break your container (or worse).

#### Portainer
[Portainer](https://www.portainer.io/) allows you to manage your docker containers via a nice web interface. Portainer provides some fine-grained control over your containers.
```
docker run --name=portainer -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v /volume1/docker/portainer:/data --restart always portainer/portainer
```
After running the above command, you can find the new portainer interface active at `http://hostname:9000/`. You can now use Portainer (or the default Docker manager in on your NAS) to manage your containers.

#### Watchtower
[Watchtower](https://github.com/containrrr/watchtower) automatically updates your docker containers. The below command sets up a Watchtower container that monitors your containers for updates and automatically updates and restarts them with the same parameters as they were made.

You can customize when Watchtower runs, which is important because it may create some downtime for your docker services. The below command has Watchtower update the containers (including stopped containers) at 6:00 AM every morning (presumably when no one is using the containers). Refer to the [Watchtower Documentation](https://containrrr.dev/watchtower/) for customization options.
```
docker run -d --name watchtower -v /var/run/docker.sock:/var/run/docker.sock -e TZ=Europe/London -e WATCHTOWER_CLEANUP=true -e WATCHTOWER_SCHEDULE="0 0 6 * * *" -e WATCHTOWER_INCLUDE_STOPPED=true --restart unless-stopped containrrr/watchtower
```

#### Calibre-Web
[Calibre-Web](https://docs.linuxserver.io/images/docker-calibre-web) provides you with a nice way to host your calibre database on your home network.

```
docker create --name=calibre-web -e PUID=[UID] -e PGID=[GID] -e TZ=Europe/London -e DOCKER_MODS=linuxserver/calibre-web:calibre -p 9002:8083 -v /volume1/docker/calibre-web:/config -v /volume1/Files/Books:/books --restart unless-stopped linuxserver/calibre-web
```
    
Once the container is running, the calibre-web interface will be available at http://host:9002. You will need to point it to an existing Calibre database in order for it to host your books.

#### Ubooquity
[Ubooquity](https://docs.linuxserver.io/images/docker-ubooquity) provides an interface to host and read your comics online. It can also host books, but I find that it struggles with a large books database. As a result, I use Ubooquity only for comics.

```
docker create --name=ubooquity -e PGID=[UID] -e PGID=[GID] -e TZ=Europe/London -p 2202:2202 -p 2203:2203 -v /volume1/docker/ubooquity:/config -v /volume1/Files/Books:/books -v /volume1/Files/Comics:/comics --restart unless-stopped linuxserver/ubooquity
```
    
Once the container is running, The Ubooquity admin interface will be available at http://host:2203/ubooquity/admin/. The user interface is available at http://host:2202/ubooquity.

## Local Dotfile Syncing
I use my Synology NAS to manage my dotfiles on my home network. I do this primarily using a combination of Cloud Sync and [Syncthing](https://syncthing.net/). At the time of writing, there is no eligible community Syncthing package for the DS920+, so I have temporarily solved this need using Docker to install Syncthing. However, this solution is not perfect, and I will note the limitations below. My current dotfile syncing solution is as follows:
- Set up Cloud Sync on the DS920+ to sync with a cloud service (such as Dropbox or Google Drive).
- Move my dotfiles into a folder on the cloud storage service.
- Install the Syncthing docker container (after creating a `syncthing` folder in the `docker` Shared Folder):

```
docker create --name=syncthing -e PUID=[PID] -e PGID=[FID] -e TZ=Europe/London -e UMASK_SET=022 -p 8384:8384 -p 22000:22000 -p 21027:21027/udp -v /volume1/docker/syncthing:/config -v [Path to your dotfiles in the cloud]:/Dotfiles -v [Path to any additional folders you want to share]:/data1 -v [Path to any additional folders you want to share]:/data2  --restart unless-stopped linuxserver/syncthing
```
    
- Install Syncthing on all clients that you would like to sync your dotfiles to. On Linux, I generally use a docker container similar to the above. On Windows, I have had success with [SyncTrayzor](https://github.com/canton7/SyncTrayzor).
- Configure Syncthing with the appropriate settings. The Syncthing interface can be accessed via `http://host:8384`. I normally do the following:
    - Give the device a readable name.
    - Disable Anonymous Usage Reporting.
    - Provide a GUI Authentication Username and Password.
    - Disable "NAT Traversal", "Global Discovery", and "Enable Relaying". My Syncthing service is not exposed to the internet, so these options are not desirable in my case. If you want to expose Syncthing to the internet (which is a security concern), you may wish to look into these options.
- Share your dotfiles folder via Syncthing on your Synology NAS to each your other computers.
- On each of your computers, create links to the dotfiles as necessary. For example, in Linux, if you have a custom `.zshrc` in your `dotfiles` folder (synced by Syncthing to your `~/Dotfiles` folder), you can create the link by typing the following command:
    `ln -s ~/Dotfiles/.zshrc ~/.zshrc`

Now, when one computer makes a change to a dotfile, such as your `~/.zshrc` file, Syncthing syncs that change back to your Synology NAS, which then shares that change with all of the other computers. Since the folder Syncthing is sharing is also in a cloud service *presumably*, the files would get uploaded to your cloud service provider as well (as a backup, and also providing a potential option to sync your dotfiles to laptops that aren't always connected to your home network). **[BUG]** **HOWEVER**, this doesn't happen because Docker does not create an inotify event so your Cloud Sync service never sees the change (except when the service is starting up). Once a Syncthing client is available for installation on the DS920+, installing it outside of Docker and using it should resolve this issue in theory. As a temporary workaround, I have created two scheduled events that run in the early morning when no one will be using the NAS: one that stops the Cloud Sync service, and another that starts it 5 minutes later. All of your dotfile changes are then backed up to your cloud service on a daily basis when the Cloud Sync service restarts.

## Memory Upgrade
The DS920+ comes with 4 GB of RAM, which is fine for Plex and using it as a light docker server. You can upgrade the RAM in your DS920+ using official Synology RAM, which is  the officially supported method, but is generally more expensive. The DS920+ only has one open slot for an additional stick of RAM. Officially, this slot is only documented to be compatible with a 4 GB official Synology RAM module (giving you a total of 8 GB of RAM).

The DS920+ can take an unofficial memory expansion. I have only tested this with a Crucial 16GB Single DDR4 2666 MT/s (PC4-21300) DR x8 SODIMM 260-Pin Memory - CT16G4SFD8266 (16 GB Dual Rank), giving the DS920+ 20 GB of RAM.

**NOTE THAT INSTALLING THIS TYPE OF UNOFFICIAL UPGRADE IS UNSUPPORTED, MAY BREAK YOUR DEVICE, MAY VOID YOUR WARRANTY, AND MAY CAUSE DATA LOSS. I am not responsible for anything you do to your device and you do this at your own risk**. Although this worked for me, other users have reported having issues.

To install the Crucial memory stick, I did the following:
  1. Initiate a Memory Test using Synology Assistant before installing any additional RAM module to ensure that your base system is viable.
  2. Wait for the test to complete. If your NAS reboots successfully, the test was successful. You can also check by running the following command (which is case sensitive) on your NAS via SSH: `sudo cat /var/log/messages | grep 'Memtest'`
  3. Power off your NAS.
  4. Unplug your power supply.
  5. Remove all hard drive bays. Keep note of their order, as you will have to reinstall them in the same order.
  6. Install the RAM Module (preferably using anti-static safeguards). Be sure not to touch the pins/chips on the bottom.
  7. Reinsert the hard drives in the correct order.
  8. Reattach your power supply.
  9. Power on your NAS.
  10. Cross your fingers and hope that it boots.
  11. If it boots, log into your NAS and check the total memory available via "Resource Monitor". It should say 20 GB.
  12. Initiate a Memory Test using Synology Assistant again to ensure that the new RAM works.
  13. Wait for the test to complete. If your NAS reboots successfully, the test was successful and you have completed your RAM install. You can also check by running the following command (which is case sensitive) on your NAS via SSH: `sudo cat /var/log/messages | grep 'Memtest'`
  14. Enjoy your 20 GB of memory!
