---
layout: post
title: 5620 SAM DNS Part 2
comments: True
---

Intro
-----

This is part 2 of 3 of how to use Nokia's 5620 SAM to automatically create and delete DNS records for network interfaces. This is an on going project, so not all the details have been worked out.

[Part 1](http://gotz.co/2016/04/13/sam-dns-part-1/) will be the background and tools I'm planning on using.

[Part 2](http://gotz.co/2016/05/23/sam-dns-part-2/) (aka this post) will be on getting it to actually work and making it a bit more modular so it can be customized for any company.

Part 3 (link sometime way in the future) will be on adding more automation, like tracking interfaces that were removed and delete them after a set period of time.

All of the code will be available on GitHub.

The code
--------

[SAMOpy](https://github.com/nlgotz/samopy) has the base code for interacting with Nokia's 5620 SAM. Right now, the only function that is implemented for end users is the get_network_interfaces(). I am planning on breaking it up and a little bit easier to use for more situations. The nice thing with the SAM OSS XML interface is that you can reuse a lot of the XML. This means that once I figure out how to implement some functionality (like filtering), it will be easy to use it in any CRUD action. For the time being, I'm not planning on implementing a lot more to the code. I'll probably add some additional stuff if I get time to work on automating other parts of 5620 SAM, but for the time being, this is a good first step and a good demo to get others on board.

I haven't published the sam2dns.py code yet as it's not really fit for public release. I will release a stripped down version to show the concept, but as of now, it's too custom.

The process for sam2dns.py is pretty simple:

1. Connect to the 5620 SAM server
2. Read the network interfaces with the get_network_interfaces() function to a variable
3. Get the contents of the variable into a dict with xmltodict
4. Connect to the Infoblox server
5. Run a for loop on the dict creating both A and PTR records

For testing and demonstration purposes, I also put in a way to see what is going to get added to DNS. This is good for troubleshooting issues.

One thing that I didn't list in the process is checking if the interface is the system address. The way I have it set up is that if the interface is the system address, it will create the A and PTR records with just the device hostname.

Conclusion
----------

A benefit of automatically creating DNS records is that if an addition fails because it isn't a valid DNS record, you can easily find the interface. I came across a couple of interfaces like this and it was pretty simple to change the interface (during a change window).

Having automated DNS records created for Nokia routers allows for easier troubleshooting because you can see the interface name in traceroutes. It also means that you don't have to waste your time adding records.
