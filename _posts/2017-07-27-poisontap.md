---
layout: post
title: "Building a PoisonTap"
date: 2017-07-27
tags: [raspberry pi, local attacks, projects, browser]
comments: false
share: true
---

I am not at DC or BH this year, so to at least get in the spirit (in addition to reading *PoC||GTFO*) I decided to try out a fun little project I've wanted to mess with for a while, Samy Kamkar's [PoisonTap](https://github.com/samyk/poisontap). So I went out, grabbed a Raspberry Pi Zero, and got to work.

![alt text](https://github.com/cgiliberto/cgiliberto.github.io/blob/master/images/poistontap.jpg "RasbPi Zero")

For details about what PoisonTap does I highly recommend reading the project page and watching Samy's introduction video, but the gist of it is that you make the pi emulate a USB ethernet interface, which when plugged into a target machine, harvests cookies and optionally sets up a semi-persistent web backdoor. This works as long as a browser is running, even if the device is locked.

Setup is really simple. I used a fresh, fully-patched Rasbpian install and simply cloned the repo onto both the pi and my C2C server, which in this case was simply one of my lab machines (running Linux).  If you want to try out the backdoor, just edit `target_backdoor.js` and `backdoor.html` to point to your C2C server. On the pi, I created an `install.sh` following the documentation, containing

```
echo -e "\nauto usb0\nallow-hotplug usb0\niface usb0 inet static\n\taddress 1.0.0.1\n\tnetmask 0.0.0.0" >> /etc/network/interfaces
echo "dtoverlay=dwc2" >> /boot/config.txt
echo -e "dwc2\ng_ether" >> /etc/modules
sudo sed --in-place "/exit 0/d" /etc/rc.local
echo "/bin/sh /home/pi/poisontap/pi_startup.sh" >> /etc/rc.local
mkdir /home/pi/poisontap
chown -R pi /home/pi/poisontap
apt-get update && apt-get upgrade
apt-get -y install isc-dhcp-server dsniff screen nodejs
```

After running this, just replace the existing `dhcpd.conf` with the one in the repo, reboot, and you're all set. The main functionality, the cookie harvesting, worked perfectly against the target machine, a Mac Mini running fully-patched macOS Sierra.

![alt text](https://github.com/cgiliberto/cgiliberto.github.io/blob/master/images/tapped.png?raw=true "Optional splash")

The splash is of course there for testing purposes and can be disabled for a real attack. The harvested cookies are stored locally on the pi, and this functionality worked perfectly.

![alt text](https://github.com/cgiliberto/cgiliberto.github.io/blob/master/images/injectionredacted.png?raw=true "Stolen cookie")

The backdoor was a little bit more finnicky. In principle, the browser should open a socket to the C2C server whenever the target browses to a poisoned cached domain. However, the sockets would not reliably open at first. They eventually started working with no configuration changes, so I'll have to debug that later. Regardless, the backdoor does indeed work.

![alt text](https://github.com/cgiliberto/cgiliberto.github.io/blob/master/images/BACKDOORredacted.png?raw=true "socket")

![alt text](https://github.com/cgiliberto/cgiliberto.github.io/blob/master/images/commandandcontrolredacted.png?raw=true "C2C")

I intend to play with the backdoor more in the coming days, but even without it the cookie harvesting ability is impressive enough for me to highly recommend this as a fun, easy, cheap project resulting in a tool that is actually useful in local attacks.
