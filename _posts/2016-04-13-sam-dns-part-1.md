---
layout: post
title: 5620 SAM DNS Part 1
comments: True
---

Intro
-----

This is going to be part 1 of 3 of how to use Nokia's 5620 SAM to automatically create and delete DNS records for network interfaces. This is an on going project, so not all the details have been worked out.

[Part 1](http://gotz.co/2016/04/13/sam-dns-part-1/) (aka this post) will be the background and tools I'm planning on using.

[Part 2](http://gotz.co/2016/05/23/sam-dns-part-2/) will be on getting it to actually work and making it a bit more modular so it can be customized for any company.

Part 3 (link sometime way in the future) will be on adding more automation, like tracking interfaces that were removed and delete them after a set period of time.

All of the code will be available on GitHub.

Parts and Pieces
----------------

The plan is to use Python to write this. There is a 5620 SAM package out on PyPi called [SAM](https://pypi.python.org/pypi/sam) but it is outdated and doesn't work with more recent versions of 5620 SAM (or at least it didn't work with 13.0.R7-P2). So I am going to have to write a new package. It will actually be two packages. One that handles the basic connectivity to 5620 SAM. The other one will be more user friendly. The reason for doing this is that the SAM OSS connectivity seems to be pretty stable so once that's working, I can release that as a Python package. I'd rather have the basic connectivity and extra functionality separate so that developers can extend how they want.

To check if DNS records need to be created, I'll be using DNS Python. This will check if A and PTR records exist. If they do, it will either delete and create or skip, depending on what the user wants.

To create the A and PTR records, I'll be using netopsdeploy, a Python package I wrote that will create A and PTR records (along with other functionality).

This is a list of pre-built tools:
- [DNSPython](http://www.dnspython.org/)
- [NetopsDeploy](https://github.com/nlgotz/netopsdeploy)


SAM-OSS
-------

In order to interact with 5620 SAM via an external program, the way you do that is via SAM-O, an OSS hook. OSS stands for Operations Support System. Unfortunately, the way you interact with SAM-O is via SOAP XML. Personally, JSON would be a lot easier, but you work with what you have. I won't go over how you use SOAP, as the documentation on Nokia's website is really good.

One thing I found out early on when testing SAM-O is that the user that you use needs SAM-O permission. Another thing is, make sure to add filters, especially if you have a lot of nodes. When I was testing with two Google Chrome Applications (ARC (Advanced ReST Client) and Boomerang), both choked and crashed with the amount of data if you don't filter.

Conclusion
----------

So this post doesn't have a whole lot of actual action, but it allows me to setup what I'm planning on doing. And it doesn't have to be just adding network interfaces to DNS, there is a lot of room for automation with 5620 SAM.

Follow [https://github.com/nlgotz/samopy](https://github.com/nlgotz/samopy) to see the progress on the Python package.
