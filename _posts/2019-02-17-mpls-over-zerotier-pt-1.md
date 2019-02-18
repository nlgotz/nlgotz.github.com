---
layout: post
title: Running MPLS over ZeroTier Part 1
comments: True
---

I had a use case to run an MPLS network over the internet. Talking with other people running MPLS networks, most of them either avoided running MPLS over public networks or ended up running GRE. Those that didn't run MPLS over public networks but needed to connect sites by cellular or internet, would usually run DMVPN. My main objective was to make the MPLS nodes connected over the internet look just like any other node in our network.

Thanks to [The Network Collective](https://thenetworkcollective.com/) Slack group, I was introduced to [ZeroTier](http://zerotier.com/). ZeroTier is an SDN VPN. The main selling feature for me is that ZeroTier is a layer 2 network that just shows up as an interface in Linux with an MTU of 2800. The capabilities to bridge the ZeroTier network to a physical interface in Linux and the MTU size of 2800 make it an excellent choice for an MPLS underlay.

Typically, installing and setting up a ZeroTier network is really simple (even simpler if you trust random bash scripts from the internet).

    curl -s 'https://pgp.mit.edu/pks/lookup?op=get&search=0x1657198823E52A61' | gpg --import && \
    if z=$(curl -s 'https://install.zerotier.com/' | gpg); then echo "$z" | sudo bash; fi
    sudo zerotier-cli join <network>

And with the [MyZeroTier](https://my.zerotier.com) Controller, you authorize the new node and it's part of the network.

Unfortunately, we don't want our nodes going through the central ZeroTier controller so that makes it a little bit more complicated. Luckily, Key Networks has [ZTNCUI](https://github.com/key-networks/ztncui) which is an Open Source Controller. It's layout is not as nice as the MyZeroTier Controller, but it has pretty much all the same functionality.

ZeroTier's concept of planets and moons means that by default, your node will attempt to contact 2 ZeroTier controlled controllers (Alice and Bob). ZeroTier has 12 servers per controller that your node could attempt to connect to. Setting up your own planets isn't something that is supported in the current 1.2.12 version in a simple way. Again, we don't want our network talking to ZeroTier at all. If you do want to run your own planets, you need to build your own planets file.

Here's the steps I used to create my own planets file.

1. Check out ZeroTier from Github:

       git clone https://github.com/zerotier/ZeroTierOne.git
2. Go into the world folder inside the attic

       cd ZeroTierOne/attic/world
3. Edit the mkworld.cpp file. Remove the IP addresses of the ZeroTier Controllers and add your own. ZeroTier has 2 controllers, but more can be added.
4. Build the file

        source ./build.sh
5. Run the mkworld file

        ./mkworld
6. A new world.bin file should be created. This will be the file that all of your nodes need
7. Copy the world.bin file to your ZeroTier-One folder (this works on Linux)

        cp world.bin /var/lib/zerotier-one/planet
8. Restart ZeroTier

        sudo systemctl restart zerotier-one.service
9. Copy the world.bin file and repeat Steps 7 and 8 to each node you want to use.

And with that you can run your own private ZeroTier network.

In Part 2, I'll go over the hardware and software implementation used to make this work.
