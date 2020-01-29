---
layout: post
title: Updating DHCP Servers on Nokia SROS
comments: True
---

I recently got a request from our network team to come up with a way to update all the DHCP server entries in our Nokia nodes. With about 300 nodes and assuming 3 VPRN interfaces per node, there is no way they could easily update all the interfaces in a timely manner.

I ended up creating [sros-dhcp](https://github.com/nlgotz/sros_dhcp) with Ansible to facilitate this update.

## How to Use

0. Ensure you have the latest Ansible (2.9.4 as of this writing) and paramiko installed

    `pip install ansible paramiko`

1. Clone the Repository

    `git clone https://github.com/nlgotz/sros_dhcp.git && cd sros_dhcp`

2. Edit the mpls file with the nodes you want to run this on

    `vi mpls`

3. Edit the group_vars/all.yml with your username, current DHCP Servers, and future DHCP Servers

    `vi group_vars/all.yml`

4. Run the Gather playbook to gather the interfaces to change. This will also create a file with "bad interfaces" which are interfaces that don't match the supplied current_dhcp_servers variable. This is a nice way to catch interfaces that may have a typo.

    `ansible-playbook -i mpls gather_dhcp_interfaces.yml`

5. Review the bad_interfaces.csv file. Update those interfaces if you need to. If you make any changes to the "bad interfaces", you'll need to rerun the gather_dhcp_interfaces.yml playbook from step 4.

6. Once everything looks good, push out your new DHCP Servers

    `ansible-playbook -i mpls update_dhcp_interfaces.yml`

7. Update the group_vars/all.yml file with the new current DHCP servers.

    `vi group_vars/all.yml`

8. Run the gather playbook to make sure all the changes took effect.

  `ansible-playbook -i mpls gather_dhcp_interfaces.yml`

9. Review the bad_interfaces.csv file again to make sure that nothing got missed.

## Gotchas I ran into

When initially developing in our lab, I had no issues running the playbooks with no failures. On the corporate side, I realized that although I had Ansible installed, I needed to install Paramiko to get the SROS commands to run.

## Closing Thoughts

Although a fairly simple task, the sheer amount makes this a very difficult manual task and is a good way to demonstrate the usefulness of automation for our network.

This is definitely not the most efficient way to update the DHCP interfaces, but it's still quicker than SSH'ing into every single node and updating every single VPRN interface.

Now that I have a way figured out for finding and updating DHCP interfaces, the possibilities for other bulk updates becomes a lot easier.

If Nokia included a REST API or even JSON output for the 7750 SR platform and 7705 SAR platform, things would be a lot easier.
