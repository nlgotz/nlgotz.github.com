---
layout: post
title: Creating DNS records for DHCPv6 and DHCPv4 leases
---

![DHCPv4 and DHCPv6 to DNS](/assets/dhcp_dns_header.png)

DHCP is great for not having to worry about statically assigning IP addresses to devices. But then you need to figure out the IP addresses somehow. DNS is great because then you don't need to worry about what the underlying IP Address is. But how do you create DNS records for something that may change? The answer is TSIG (Transfer Signature) keys.

## What is a TSIG key?

TSIG keys ensure that DNS packets are coming from a trusted source. When DHCP hands out a new reservation, it will send that info over to the DNS server to create the record.

## Technology

For this work, I'm using PowerDNS for the DNS Server and ISC's DHCP Server for DHCPv4 and DHCPv6. This tutorial isn't how to setup PowerDNS or ISC servers, but to tie them together.

## Sample info

For this, we'll be using 2001:db8:7aco::/64, 192.168.99.0/24 for the network and dhcp.gotz.co for the domain. This is also assuming that the DNS and DHCP applications are on the same server.

## PowerDNS Configuration

Before configuring too much of the DHCP servers, the first step is to generate and apply the TSIG keys to the domain. What I've found is that for each domain/subdomain you want to create DNS records, you need separate TSIG keys. This may not be completely necessary but makes things a lot simpler if you ever need/want to send the DNS requests to different servers.

The first step is to generate the keys and apply it to the domains:

```bash
sudo pdnsutil generate-tsig-key dhcp.gotz.co-key hmac-sha256        
sudo pdnsutil activate-tsig-key dhcp.gotz.co dhcp.gotz.co-key master  
```

Once that's done, you need to get the keys so that it can be added to DHCP.

```bash
sudo pdnsutil list-tsig-keys
```

## DHCP Configuration

Dynamic DNS configuration in ISC DHCP has two modes of update styles (standard and interim). This works to our advantage as you can't have both DHCPv4 and DHCPv6 use the same update style. It doesn't really matter either way which you configure each server, just make sure not to have both using the same style.

Neither of these configurations are complete, they just have the DDNS configuration. You'll need more to make the full DHCP configuration operational.

### DHCPv6 configuration

Let's start with the configuration of the current version of IP.

This would go in the /etc/dhcp/dhcpd6.conf file.

```bash
ddns-updates on;
ddns-update-style standard;
ddns-dual-stack-mixed-mode true;
update-conflict-detection true;
ddns-ttl 300;

# Add the TSIG key generated with pdnsutil
key "dhcp.gotz.co-key" {
    algorithm hmac-sha256;
    secret "<key>";
}

zone dhcp.gotz.co {
    primary 127.0.0.1;
    key "dhcp.gotz.co-key";
}

subnet6 2001:db8:7aco::/64 {
  range6 2001:db8:7aco::10 2001:db8:7aco::ffff;
  option dhcp6.name-servers <DNS server>;
  option dhcp6.domain-search "dhcp.gotz.co";
  ddns-domainname "dhcp.gotz.co";
}
```

With DHCPv6, you can't give out a default route so you need to have either Neighbor Discovery or Router Advertisements. That's outside of the scope of this work.

### DHCPv4 configuration

And now for the legacy IPv4 protocol

This would go in the /etc/dhcp/dhcpd.conf file.

```bash
dns-updates on;
ddns-update-style interim;
ddns-dual-stack-mixed-mode true;
update-conflict-detection true;
ddns-ttl 300;

# Add the TSIG key generated with pdnsutil
key "dhcp.gotz.co-key" {
    algorithm hmac-sha256;
    secret "<key>";
}

zone dhcp.gotz.co {
    primary 127.0.0.1;
    key "dhcp.gotz.co-key";
}

subnet 192.168.99.0 netmask 255.255.255.0 {
        option domain-name-servers <DNS server>;
        option domain-search "dhcp.gotz.co";
        option routers 192.168.99.1;
        ddns-domainname "dhcpgotz.co";
        pool {
                range 192.168.99.20 192.168.99.254;
        }
}

```

## DHCPv6 on Linux

One snag I ran into is getting DHCPv6 to create DNS records for Linux devices. I've found

## Finishing it up

Once all of this work is done, restart the services and then you can start enjoying using DNS to access all of your DHCP'd devices on your network. This requires modifying the end device's /etc/dhclient.conf to send the fqdn.

```bash
send fqdn.fqdn = gethostname();
```

Once this is updated, then the Linux client will send it's hostname to DHCP which can then create the DNS records.

### References

Here's some of the resource's I've used to get this setup up and working:

- [Get up and running on with your own DNS and DHCP server from scratch (PowerDNS + isc-dhcp-server)](https://carll.medium.com/get-up-and-running-on-with-your-own-dns-and-dhcp-server-from-scratch-powerdns-isc-dhcp-server-4b9d6185d275)
- [ISC DHCP Client bug report](https://bugs.launchpad.net/ubuntu/+source/isc-dhcp/+bug/991360)
