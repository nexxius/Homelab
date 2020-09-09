# Homelab

A "Homelab" generally refers to an information technology workspace/testing space implemented at your home. It provides you with an opportunity to use, test, implement, and learn IT solutions that are normally only implemented in business environments. Additionally, a Homelab should hopefully be useful to you and you should test out services that you might use yourself (such as media hosting, file hosting, or home automation).

For more information, you should check out communities such as [/r/Homelab](https://reddit.com/r/homelab) on Reddit.

## Is Running a Homelab Expensive?
If you went over to [/r/Homelab](https://reddit.com/r/homelab), you might be under the impression that running a Homelab requires you to buy a full server rack or to invest thousands of dollars up-front to have a usable environment. That is not the case! Below are some of the considerations you may want to take into account:

- **Developed Over Time:** A good Homelab is developed over time as you figure out what your needs, interests, and limitations are. It is easy to get caught up with having the newest, fanciest equipment and forget that the main point of running a Homelab is to learn.
- **Your Devices Do Not Need to be Enterprise-Grade:** You can start with old, repurposed devices. Another popular starting point are [Raspberry Pis](https://www.raspberrypi.org/), which are relatively straightforward to use and cheap to buy. As your needs increase, you can improve your hardware, move up to bigger (and sometimes more expensive) devices. Keep in mind that older decommissioned enterprise-level equipment is also cheap. If you are running a Homelab in your home, the additional things you may wish to consider when picking hardware might include:
  - *Purpose*. What role will the device fill? Is it for learning and testing, or do you need it to be reliable because other people will be using the device?
  - *Upfront Cost*. How much does it cost up front?
  - *Projected Lifespan and Use*. How long will you use the device for? Do you anticipate it breaking down after a certain point? Can it be upgraded? After it has served its main purpose, can it be repurposed later?  
  - *Power draw*. Consider whether your device will be running 24/7 and how much power it will consume over a month or a year.
  - *Noise*. Repurposed enterprise-level servers may have multiple fans and may be extremely noisy. The sound of a few hard drives in a RAID setup reading/writing data may also be noisy. Depending on your home, this may be a deal breaker.
  - *Size*. If you live in an apartment, you might have some trouble storing a server rack, as opposed to a credit-card sized Raspberry Pi.
- **Consider Cost Savings:** If you pay for certain cloud solutions (such as Dropbox for file storage), consider whether your needs could be addressed by performing these services on your home network (taking into account your time, electricity, and equipment costs). Over a long enough period of time, this may result in a cost savings for you.

## Basic Homelab Philosophies
In this section, I will set out some basic Homelab philosophies that can help to organize your Homelab. Keep in mind that these are guidelines and not requirements. Your ability to do these things may be limited by cost, space, or reliability requirements. Part of running a Homelab is figuring out what works for you, so you should take this with a grain of salt.
- **Security First:** When building a Homelab, you may start hosting services on your network (such as an instance of Transmission with a web interface). You might then think to yourself "it would be great if I can access this service from anywhere!". You then go down the rabbit hole of exposing ports and making your services available via the internet. You will likely find that you are quickly compromised. Take the time to learn security concepts. Deploy a VPN or a reverse-proxy. Security is a crucial aspect of maintaining any IT infrastructure and do not cheat yourself by failing to consider and learn it. I will refer generally in this article to the three concepts in the CIA triad:
  - *Confidentiality*: Your data and systems should be safe, private, confidential, and free from unauthorized access or disclosure.
  - *Integrity*: Your data and systems should not be subject to unauthorized or unintentional modification.
  - *Availability*: Your data and systems should be available to its authorized users. You should avoid downtime (a period of time where a service, system, or data is unavailable) where possible.
- **Division Between 'Production' and 'Testing':** A 'production' environment is an environment that is actively being used and relied upon. A 'testing' environment is one where new solutions are being tested, tweaked, and configured with the hope of maybe moving these solutions to the 'production' environment. A production environment should be secure, with a particular focus on it being available. This is particularly important in the days of COVID-19 where many people are working from home. The time to debug a new firewall you built is while it is in 'testing' and is not being actively used or relied upon, not 5 minutes before your Microsoft Teams call when your firewall decides to drop all traffic for some reason.
- **Division of Infrastructure:** When starting out, all of your testing and services may be done on one device for cost reasons. For example, a Raspberry Pi 4 (4 GB) can generally handle hosting a VPN server, a DNS-level ad-blocking solution, and a few docker containers. However, if you are working remotely using the VPN on your Raspberry Pi and you either accidentally misconfigure its network settings or reboot the Pi, you will be kicked off your VPN connection. If the Pi doesn't reboot properly or return to normal on its own, you may have no way of getting back into your network to service it remotely. I have tried to separate out several "buckets" of infrastructure that sit on dedicated devices, including:
  - *Network Infrastructure:* This may include VPN solutions, DNS services, routing, DHCP services, and firewall functionality. I try to keep these functions separate from other devices so that the above scenario with the Raspberry Pi VPN is not a significant concern.
  - *Media and File Hosting:* This may include file hosting, network attached storage solutions, media maintenance, and downloading services. I try to keep these separate from other devices since other devices may rely on the file hosting. You may want to avoid causing outages for all of your devices that rely on your media server every time you have to reboot your media server because it is also running web services.
  - *General Services:* This may include self-hosted services like Organizr, Home Assistant, or Node Red. Although it isn't critical for this to be separate from your Media and File Hosting services, it can be helpful to run services like Organizr or Home Assistant on a dedicated, always-on, power-efficient device that can turn other devices on/off as needed.
  - *Personal Computer:* I try to avoid running these devices on a personal computer that you use for 'every day' things. Although running all the services on a single personal desktop computer may be tempting and cost-efficient, you are starting to blur the lines between a 'production' and 'testing' environment. If your computer is a gaming PC with a large graphics card, this also may be inefficient from an: (1) energy perspective; and (2) wear-and-tear on your computer if you leave it running 24/7.
- **Accessibility:** This concept needs to be balanced against the "Security First" approach described above. If a Raspberry Pi on your network goes down, you may not be in a position to (or want to) walk over to it, plug a keyboard, mouse, and monitor into it, and physically address it. Instead, you may want to consider allowing yourself remote access to the device. This could mean giving yourself access to SSH on the device, plugging the Raspberry Pi into a smart plug that you can toggle on and off if necessary, or running a VNC or RDP solution so you can remotely administer your devices. Done security (and combined with a secure implementation of a VPN), this can allow you to fix issues or work on your lab from anywhere.

## Getting Started with Your Homelab
Take a look at the following projects for some easy ideas to get started with a Homelab.
- [Build a Raspberry Pi 4 server as a Homelab starting point](RaspberryPi4_Server.md).
  - Infrastructure needs you can address:
    - Network Infrastructure: Run a VPN server and a DNS-level ad-blocker.
    - Media and File Hosting: Host files on your Pi for the rest of the network to use.
    - General Services: Run services such as Home Assistant or Organizr to benefit your network.
  - Skills you can learn:
    - Linux use and administration
    - Docker
    - Networking
    - Security
- Build, set up, and configure your own router/firewall with pfsense.
  - Infrastructure needs you can address:
    - Network Infrastructure: Set up services such as a VPN server or DNS-level ad-blocking. Keep these services separate from other devices for increased availability.
  - Skills you can learn:
    - Networking and routing
    - Firewall administration
    - Security
- [Set up a Synology Device (or other NAS, including custom-built ones) to host, store, and manage your files and media](Self-Hosted_NAS.md).
  - Infrastructure needs you can address:
    - Network Infrastructure: Synology makes certain packages available to install, including a VPN solution.
    - Media and file hosting: Host your files for your other systems.
    - General Services: Run docker on your system to host additional services.
  - Skills you can learn:
    - Docker
    - File permissions
    - Security
    - Cloud hosting
    - Availability technologies including RAID and the use of an Uninterruptible Power Supply
- Set up a Proxmox Server for Virtual Machine management, deployment, and use
  - Infrastructure needs you can address:
    - Proxmox is a platform that can be used to create and run virtual machines. As a result, it can address almost any needs (subject to your system's processing power, storage, and memory). Note that if you use Proxmox to build your environment virtually (such as by running a virtual instance of pfsense and other virtual servers), your Proxmox system becomes a single point of failure for your network as a whole.
