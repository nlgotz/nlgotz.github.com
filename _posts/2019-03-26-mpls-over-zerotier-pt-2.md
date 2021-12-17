---
layout: post
title: Running MPLS over ZeroTier Part 2
---

In [Part 1](https://gotz.co/2019/02/17/mpls-over-zerotier-pt-1/), I
went over the use case and how to build your own planets file. In Part
2, we'll go over the hardware and some of the issues I ran into setting up the software.

## Hardware

For the environment I'd be deploying the remote nodes in, there were some important features
needed for the hardware.

1. Needs to be able to run ZeroTier. (Hopefully obvious?)
2. DIN Rail or wall mountable. If it needs to be in a rack, we can
mount DIN rail to the rack.
3. -48VDC power. We want our equipment to be able to use the DC plant at the
sites.
4. Built in LTE.
5. Multiple Ethernet interfaces (2 or more).

Something that would have been nice would have been some SFP ports,
but that wasn't critical.

Doing some research for the remote side, I came across industrial PCs that have LTE modems built in. These are typically
used in manufacturing or other industrial environments. I ended up going with the [Cincoze DA-1000
from Logic Supply](https://www.logicsupply.com/da-1000/) as it had most of the features we need. With a change in processor and the LTE modem, this works as a great remote node.

On the hub side, I ended up using some servers that were running virtual
machines. 

## Software

I ended up choosing Ubuntu 18.04 as the operating system for both sides. This was mainly due to my familiarity with Ubuntu. ZeroTier should work on most Linux operating systems and I believe it also works on OpnSense. Installing and configuring the Ubuntu onto the device went fairly smoothly.

There were two issues with getting ZeroTier setup on Ubuntu to act as a bridge
and that was with Netplan.io and getting the LTE
modem to work.

Netplan.io has a bug in it where it won't allow you to set the MTU
size of an interface which is a major issue since we want to run the
interface at 2800 to match the ZeroTier interface and allow for the
MPLS tags to pass through without modifying the interface on the MPLS
node. I replaced Netplan.io with ifupdown and ifupdown2 to get around
this.

The LTE modem issue took a lot of trial and error to get working
properly. I was having issues getting the LTE modem to be recognized
as a network interface by ifupdown. I believe this will get it working
for future nodes.

        sudo apt install unzip libqmi-utils libmbim-utils openresolv
        git clone https://github.com/danielewood/sierra-wireless-modems.git
        cd sierra-wireless-modems
        sudo chmod +x autoflash-7455.sh
        sudo ./autoflash-7455.sh


It will pop up with "Are you sure you want to continue? (CTRL+C to
exit). Type 'y' and hit enter.

Then we need to get the interface to actually come up and properly get
an LTE IP address

        git clone
https://github.com/andrewbasterfield/debian-thinkpad-wwan-EM7455.git
        cd debian-thinkpad-wwan-EM7455
        sudo install -o root -g root -m755 wwan.sh /etc/network/wwan.sh
        sudo install -o root -g root -m755 wwan_parse_ip_info /etc/network/wwan_parse_ip_info

        echo 'APN=<cell-provider-APN>' | sudo tee /etc/mbim-network.conf

  Add the following to /etc/network/interfaces

        allow-hotplug <interface_name>
        iface <interface_name> inet manual
         pre-up /etc/network/wwan.sh start
         pre-down /etc/network/wwan.sh stop

Then bring the new interface up

        sudo ifup <interface_name>

The 'debian-thinkpad-wwan-EM7455' GitHub repository doesn't completely
work for the the LTE setup in the Cincoze DA-1000 as there are some
options that aren't needed or give a warning. I'm sure with some
tweaking, I can get it to be error free. When I do, I'll make a new
GitHub repository for this setup.

Other than those two items, most of it was setting up the DA-1000 with
the proper IP addressing scheme and installing ZeroTier. I went over
installing ZeroTier and the custom planets file in [Part
1](https://gotz.co/2019/02/17/mpls-over-zerotier-pt-1/).


In Part 3 we'll wrap this up by going over setting up the ZeroTier networks, some initial MPLS results, and future improvements I'd like to make.
