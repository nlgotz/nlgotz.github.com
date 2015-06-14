---
layout: post
title: SCADA Bridging How-to
---

Intro
-----

When the company I work for upgraded their WAN infrastructure, one of the selling points of going to [Alcatel-Lucent](http://alcatel-lucent.com) was that they had better support for TDM and legacy equipment. It would have the ability to slowly (but eventually) allow us to migrate away from Digital Cross Connects (DCS) and channel banks while still providing those legacy circuits.

Well, I recently got a new toy at work: an Alcatel-Lucent ISC (Integrated Services Card). This allows us to do RS232 bridging. Now, I don't know how all company's do SCADA bridging, but we prefer to have active/active masters. This isn't a feature that Alcatel-Lucent has in the ISC by default, but we can work around it. Here's what our final end state should look like:

![Circuit end state](http://gotz.co/assets/scada_circuit4.png)

The Old Way (TDM)
-----------------

The old way involved a lot of predefined TDM links. All of the routes are predetermined by an engineer and if one of the links has a failure, that circuit is down.

![A simplified version of the old way of doing things](http://gotz.co/assets/scada_tdm.png)

The above drawing is simplified as it shows only one Master and one Remote. Typically, you'll have two masters that connect to any number of remotes.

The New Way (MPLS)
------------------

So now that we've seen how things were done with TDM, why would we want to use MPLS? One of the benefits of MPLS over TDM is that MPLS can route around failures. So if your sites are connected via multiple links, the odds of failure goes down and also means that instead of needing to call out a tech at 0200 on a Sunday morning, the RTU will stay up and the tech can start troubleshooting on Monday. MPLS allows us to remove another point of failure and route around it. Yes, equipment failure can still happen, but if the failure is not on the terminal equipment, clients will be less likely to notice.

Circuit Building
----------------

Before we can get to the end state, we need to start from the basics. Here are the circuits we are going to build:

1. 1 Master, 2 Remotes on a single Alcatel-Lucent 7705 SAR
2. 2 Masters (active/standby), 2 Remotes on a single Alcatel-Lucent 7705 SAR
3. 2 Masters (active/active), 2 Remotes on a single Alcatel-Lucent 7705 SAR
4. 2 Masters (active/active), 2 Remotes on multiple Alcatel-Lucent 7705 SARs

Before we get to the actual config, here is some conventions:

- SCADA Bridges are on 1/2/1 through 1/2/16
- RS232 ports are on 1/6/1 through 1/6/12
- RS232 to SCADA C-Pipe id's start with 6xxx
- SCADA Bridge to SCADA Bridge C-Pipe id's start with 7xxx
- B1 is the Alcatel-Lucent 7705 SAR with the ISC Card
- M1 and M2 are the Alcatel-Lucent 7705 SARs with the Masters attached (only for bridge configuration 4)
- R1, R2, and R2 are the Alcatel-Lucent 7705 SARs with the Remotes attached (only for bridge configuration 4)

### Port configuration ###

In order to build bridges, we need RS232 ports configured properly.

    A:B1# /configure port 1/6/1
    A:B1>config>port# info
    ----------------------------------------------
        description "to Master 1"
        serial
            rs232
                device-mode asynchronous
                channel-group 1
                    encap-type cem
                    no shutdown
                exit
                no shutdown
            exit
        exit
        no shutdown
    ----------------------------------------------

You will need to do this for each RS232 port you use. And obviously you will need to update the description.

I will be using 1/6/1 for Master 1, 1/6/2 for Master 2, 1/6/3 for RTU 1, and 1/6/4 for RTU 2.

### Circuit 1: 1 Master, 2 Remotes ###
![Circuit 1](http://gotz.co/assets/scada_circuit1.png)

Let's start with something simple. 1 Master and two remotes.

    A:B1>config>port# /configure scada 1/2/1
    A:B1>config>scada# info
    ----------------------------------------------
        description "Master Bridge 1"
        branch 1
            description "to Master 1"
            no shutdown
        exit
        branch 3
            description "to RTU 1"
            no shutdown
        exit
        branch 4
            description "to RTU 2"
            no shutdown
        exit
        no shutdown
    ----------------------------------------------
    A:B1>config>scada# /configure service cpipe 6001
    A:B1>config>service>cpipe# info
    ----------------------------------------------
      description "Master 1 to Bridge 1"
      sap 1/2/1.1 create
          description "to Bridge 1"
      exit
      sap 1/6/1.1 create
          description "to Master 1"
      exit
      no shutdown
    ----------------------------------------------
    A:B1>config>service>cpipe# /configure service cpipe 6002
    A:B1>config>service>cpipe# info
    ----------------------------------------------
      description "RTU 1 to Bridge 1"
      sap 1/2/1.3 create
          description "to Bridge 1"
      exit
      sap 1/6/3.1 create
          description "to RTU 1"
      exit
      no shutdown
    ----------------------------------------------
    A:B1>config>service>cpipe# /configure service cpipe 6003
    A:B1>config>service>cpipe# info
    ----------------------------------------------
      description "RTU 2 to Bridge 1"
      sap 1/2/1.4 create
          description "to Bridge 1"
      exit
      sap 1/6/4.1 create
          description "to RTU 2"
      exit
      no shutdown
    ----------------------------------------------

### Circuit 2: 2 Masters (active/standby), 2 Remotes ###
![Circuit 2](http://gotz.co/assets/scada_circuit2.png)

And now we'll add a second master. The way the masters work is that one is active and one is standby. There is no automatic failover (hopefully Alcatel-Lucent adds active/active support to a future release). You can manually fail between the two masters if needed. The example shows that the second master is primary.

One other thing to note is that branches 1 and 2 are reserved for masters. The other 30 branches are reserved for slaves. There is no way to define if the branch is a master or slave.

    A:B1>config>port# /configure scada 1/2/1
    A:B1>config>scada# info
    ----------------------------------------------
        description "Master Bridge 1"
        mddb
            force-active master 2
        exit
        branch 1
            description "to Master 1"
            no shutdown
        exit
        branch 2
        branch 3
            description "to RTU 1"
            no shutdown
        exit
        branch 4
            description "to RTU 2"
            no shutdown
        exit
        no shutdown
    ----------------------------------------------
    A:B1>config>scada# /configure service cpipe 6001
    A:B1>config>service>cpipe# info
    ----------------------------------------------
      description "Master 1 to Bridge 1"
      sap 1/2/1.1 create
          description "to Bridge 1"
      exit
      sap 1/6/1.1 create
          description "to Master 1"
      exit
      no shutdown
    ----------------------------------------------
    A:B1>config>service>cpipe# /configure service cpipe 6002
    A:B1>config>service>cpipe# info
    ----------------------------------------------
      description "RTU 1 to Bridge 1"
      sap 1/2/1.3 create
          description "to Bridge 1"
      exit
      sap 1/6/3.1 create
          description "to RTU 1"
      exit
      no shutdown
    ----------------------------------------------
    A:B1>config>service>cpipe# /configure service cpipe 6003
    A:B1>config>service>cpipe# info
    ----------------------------------------------
      description "RTU 2 to Bridge 1"
      sap 1/2/1.4 create
          description "to Bridge 1"
      exit
      sap 1/6/4.1 create
          description "to RTU 2"
      exit
      no shutdown
    ----------------------------------------------
    A:B1>config>service>cpipe# /configure service cpipe 6004
    A:B1>config>service>cpipe# info
    ----------------------------------------------
      description "Master 2 to Bridge 1"
      sap 1/2/1.2 create
          description "to Bridge 1"
      exit
      sap 1/6/2.1 create
          description "to Master 2"
      exit
      no shutdown
    ----------------------------------------------


### Circuit 3: 2 Masters (active/active), 2 Remotes ###
![Circuit 3](http://gotz.co/assets/scada_circuit3.png)

Time to make the magic happen. Alcatel-Lucent doesn't let you have active/active masters by default in a single bridge. So the way that we work around this is to take two bridges and put them back to back. This cuts the limit of bridges we can do in half, but we can still do active/active.

    A:B1>config>port# /configure scada 1/2/1
    A:B1>config>scada# info
    ----------------------------------------------
        description "Master Bridge 1"
        branch 1
            description "to RTU Bridge (1/2/2)"
            no shutdown
        exit
        branch 3
            description "to Master 1"
            no shutdown
        exit
        branch 4
            description "to Master 2"
            no shutdown
        exit
        no shutdown
    ----------------------------------------------
    A:B1>config>port# /configure scada 1/2/2
    A:B1>config>scada# info
    ----------------------------------------------
        description "RTU Bridge 1"
        branch 1
            description "to Master Bridge (1/2/1)"
            no shutdown
        exit
        branch 3
            description "to RTU 1"
            no shutdown
        exit
        branch 4
            description "to RTU 2"
            no shutdown
        exit
        no shutdown
    ----------------------------------------------
    A:B1>config>scada# /configure service cpipe 6001
    A:B1>config>service>cpipe# info
    ----------------------------------------------
      description "Master 1 to Bridge 1"
      sap 1/2/1.3 create
          description "to Bridge 1"
      exit
      sap 1/6/1.1 create
          description "to Master 1"
      exit
      no shutdown
    ----------------------------------------------
    A:B1>config>service>cpipe# /configure service cpipe 6004
    A:B1>config>service>cpipe# info
    ----------------------------------------------
      description "Master 2 to Bridge 1"
      sap 1/2/1.4 create
          description "to Bridge 1"
      exit
      sap 1/6/2.1 create
          description "to Master 2"
      exit
      no shutdown
    ----------------------------------------------
    A:B1>config>service>cpipe# /configure service cpipe 6002
    A:B1>config>service>cpipe# info
    ----------------------------------------------
      description "RTU 1 to Bridge 2"
      sap 1/2/2.3 create
          description "to Bridge 2"
      exit
      sap 1/6/3.1 create
          description "to RTU 1"
      exit
      no shutdown
    ----------------------------------------------
    A:B1>config>service>cpipe# /configure service cpipe 6003
    A:B1>config>service>cpipe# info
    ----------------------------------------------
      description "RTU 2 to Bridge 2"
      sap 1/2/2.4 create
          description "to Bridge 2"
      exit
      sap 1/6/4.1 create
          description "to RTU 2"
      exit
      no shutdown
    ----------------------------------------------
    A:B1>config>service>cpipe# /configure service cpipe 7001
    A:B1>config>service>cpipe# info
    ----------------------------------------------
      description "Bridge 1 (Master) to Bridge 2 (RTU)"
      sap 1/2/1.1 create
          description "to Bridge 1"
      exit
      sap 1/2/2.1 create
          description "to Bridge 2"
      exit
      no shutdown
    ----------------------------------------------


### Circuit 4; 2 Masters (active/active), 2 Remotes on multiple SARs ###
![Circuit 4](http://gotz.co/assets/scada_circuit4.png)

So far all of the code has been on a single SAR. This is not how it works in the real world. So how do you do it when your working with multiple boxes? Well, those C-Pipes we've been creating can be tweaked just a bit to achieve what we need. Instead of 2 SAPs, we will use 1 SAP and 1 spoke-sdp going to the other box.

#### B1 ####

    A:B1>config>port# /configure scada 1/2/1
    A:B1>config>scada# info
    ----------------------------------------------
      description "Master Bridge 1"
      branch 1
          description "to RTU Bridge (1/2/2)"
          no shutdown
      exit
      branch 3
          description "to Master 1"
          no shutdown
      exit
      branch 4
          description "to Master 2"
          no shutdown
      exit
      no shutdown
    ----------------------------------------------
    A:B1>config>port# /configure scada 1/2/2
    A:B1>config>scada# info
    ----------------------------------------------
      description "RTU Bridge 1"
      branch 1
          description "to Master Bridge (1/2/1)"
          no shutdown
      exit
      branch 3
          description "to RTU 1"
          no shutdown
      exit
      branch 4
          description "to RTU 2"
          no shutdown
      exit
      no shutdown
    ----------------------------------------------
    A:B1>config>scada# /configure service cpipe 6001
    A:B1>config>service>cpipe# info
    ----------------------------------------------
      description "Master 1 to Bridge 1"
      sap 1/2/1.3 create
          description "to Bridge 1"
      exit
      spoke-sdp M1:6001 create
      exit
      no shutdown
    ----------------------------------------------
    A:B1>config>service>cpipe# /configure service cpipe 6004
    A:B1>config>service>cpipe# info
    ----------------------------------------------
      description "Master 2 to Bridge 1"
      sap 1/2/1.4 create
          description "to Bridge 1"
      exit
      spoke-sdp M2:6002 create
      exit
      no shutdown
    ----------------------------------------------
    A:B1>config>service>cpipe# /configure service cpipe 6002
    A:B1>config>service>cpipe# info
    ----------------------------------------------
      description "RTU 1 to Bridge 2"
      sap 1/2/2.3 create
          description "to Bridge 2"
      exit
      spoke-sdp R1:6002 create
      exit
      no shutdown
    ----------------------------------------------
    A:B1>config>service>cpipe# /configure service cpipe 6003
    A:B1>config>service>cpipe# info
    ----------------------------------------------
      description "RTU 2 to Bridge 2"
      sap 1/2/2.4 create
          description "to Bridge 2"
      exit
      spoke-sdp R2:6003 create
      exit
      no shutdown
    ----------------------------------------------
    A:B1>config>service>cpipe# /configure service cpipe 7001
    A:B1>config>service>cpipe# info
    ----------------------------------------------
      description "Bridge 1 (Master) to Bridge 2 (RTU)"
      sap 1/2/1.1 create
          description "to Bridge 1"
      exit
      sap 1/2/2.1 create
          description "to Bridge 2"
      exit
      no shutdown
    ----------------------------------------------

#### M1 ####

    A:M1>config>scada# /configure service cpipe 6001
    A:M1>config>service>cpipe# info
    ----------------------------------------------
      description "Master 1 to Bridge 1"
      sap 1/6/1.1 create
          description "to Master 1"
      exit
      spoke-sdp B1:6001 create
      exit
      no shutdown
    ----------------------------------------------

#### M2 ####
    A:M2>config>scada# /configure service cpipe 6004
    A:M2>config>service>cpipe# info
    ----------------------------------------------
      description "Master 2 to Bridge 1"
      sap 1/6/1.1 create
          description "to Master 2"
      exit
      spoke-sdp B1:6004 create
      exit
      no shutdown
    ----------------------------------------------

#### R1 ####
    A:R1>config>scada# /configure service cpipe 6002
    A:R1>config>service>cpipe# info
    ----------------------------------------------
      description "RTU 1 to Bridge 2"
      sap 1/6/1.1 create
          description "to RTU 1"
      exit
      spoke-sdp B1:6002 create
      exit
      no shutdown
    ----------------------------------------------

#### R2 ####
    A:R2>config>scada# /configure service cpipe 6003
    A:R2>config>service>cpipe# info
    ----------------------------------------------
      description "RTU 2 to Bridge 2"
      sap 1/6/1.1 create
          description "to RTU 2"
      exit
      spoke-sdp B1:6003 create
      exit
      no shutdown
    ----------------------------------------------

## Conclusion ##
Setting up SCADA bridging is fairly simple in Alcatel-Lucent's 7705 SARs once you get going. One thing that I didn't include was a QOS policy on any of the SAPs. This is something that you definitely need for a production environment to ensure that the SCADA data is properly tagged and handled.
