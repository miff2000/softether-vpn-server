[![Build Status](https://travis-ci.org/miff2000/softether-vpn-server.svg?branch=master)](https://travis-ci.org/miff2000/softether-vpn-server)

# softether-vpn-server
An Ansible role for installing SoftEther VPN server on remote systems.

## Requirements
None. All required components will be installed as part of the playbook.

## Usage
The basic steps are:
* Log onto a Linux machine, which will be your Ansible control host
* Install Ansible 2.6 or above on it
> If you're using a CentOS server, you can just add the [EPEL](https://fedoraproject.org/wiki/EPEL) repo following the instructions on that link.
* Pull the Ansible Galaxy package onto your control host. This will download the package into `~/.ansible/roles/miff2000.softether-vpn-server`
```bash
ansible-galaxy install miff2000.softether-vpn-server
```
* Update your Ansible Inventory file. For example, in INI style:
```ini
[softether-servers]
softether-ansible-test ansible_host=192.168.3.26 ansible_user=centos ansible_become=true
```

> The default location is `/etc/ansible/hosts`, but if you prefer you can create a separate file for it and include it with `ansible-playbook -i <filename>` instead.

* Create your Ansible Playbook file. For example:
```YAML
- hosts:
    - softether-servers

  vars:
    softether_vpn_users:
      - name: my_user
        password: my_password
    softether_ipsec_presharedkey: "[1KH;+r-X#cvhpv7Y6=#;[{u"

  roles:
    - miff2000.softether-vpn-server
```
You'll see I've set up a separate `vars:` stanza to override the defaults, which is a nice way to see everything in one file.
> Any options specified in the **Ansible Playbook** or `vars/main.yml` will override the defaults in `defaults/main.yml`

> The above is the very minimum you should specify. You can remove `softether_ipsec_presharedkey` if you're not planning to use IPSEC. In that case you should disable it with `softether_option_ipsec: false` somewhere in the `vars:` section above

> **Full config options** are in [defaults/main.yml](defaults/main.yml), and I highly encourage you to have a look through. In the case of all options except `softether_vpn_users`, you can probably leave the defaults however

* If you prefer to keep the variables separate from the playbook, you can create a `vars/main.yml` file with options in. For example:

```YAML
option_reset_softether_config: True
option_setup_letsencrypt: True
softether_fqdn: 185.209.77.13.myservers.net
softether_option_split_routing: True
softether_dhcp_range_start: 192.168.40.10
softether_dhcp_range_finish: 192.168.40.200
softether_dhcp_netmask: 255.255.255.0
softether_dhcp_gateway: 192.168.40.1
softether_dhcp_dns1: 192.168.40.1
softether_server_cipher: ECDHE-RSA-AES128-SHA256
softether_option_securenat: True
softether_option_ipsec: False
softether_option_openvpn: False
softether_option_local_bridge: False
option_build_new_binaries: False

# VPN users
softether_vpn_users:
  - realname: User 1
    name: user1
    password: U4er1Pa55
  - name: user2
    password: U4er2Pa55
```

* If you do use a separate vars file, You'll then need to include that file in the playbook, like shown below. The name of the file doesn't need to be specific - it's just what you want to call it.
```YAML
- hosts:
    - softether-servers
  vars_files:
    - vars/main.yml
  roles:
    - miff2000.softether-vpn-server
```

## Known Issues
### SecureNAT on multi-homed servers.
One issue that I'm aware of is that SecureNAT will choose a server interface to NAT behind, and it will use that same interface IP address on the server regardless of which interface the routing table send the traffic out of. I believe it will choose `eth0` by default, but haven't had chance to confirm that logic.

What this means in practice is, if you have something like this in place:
```
+------------------+      +-----+      +---------------------------+
| VPN Client       |      | F/W |      | SoftEther Server          |      +---------------------------+
+------------------+      |-----|      +-------------------------- +      | Remote Server             |
|                  |      |--|  |      | Gateway: 192.168.3.1/24   |      +---------------------------+
| VPN adapter IP:  | ---> |  |--| ---> | eth0 IP: 192.168.3.6/24   | ---> | Gateway: 192.168.4.1/24   |
|   192.168.40.10  |      |--|  |      | eth1 IP: 192.168.4.55/24  |      | eth0 IP: 192.168.4.60/24  |
|                  |      |  |--|      | VPN DHCP pool range:      |      +---------------------------+
|                  |      |--|  |      |   192.168.40.0/24         |
+------------------+      +-----+      +---------------------------+
```
SecureNAT will have decided that when your VPN client tries to talk to the Remote Server (192.168.4.60), the packet will reach the SoftEther Server at IP 192.168.40.1. SoftEther will then NAT the source IP to 192.168.3.6, as that's the first IP on eth0. It'll then see that the destination is reachable via eth1, and so will send the packet out of eth1 towards 192.168.4.60. The Remote Server gets the packet and builds its response. However, as SecureNAT changed the source IP to be 192.168.3.6, which isn't a network that Remote Server is aware of, it then sends the response packet to 192.168.3.6 via its default Gateway, 192.168.4.1/24.

This behaviour is described as asymmetric routing, and is not good news for stateful packet inspection or firewalling.

The easiest solution to this is to only have one IP address on the SoftEther Server (192.168.3.6/24 in our case), and to allow the Gateway to do its job in routing between the 192.168.3.0/24 and 192.168.4.0/24 networks.

Two other potential solutions may be (these have not been tested):
* Configure `eth0` and `eth1` the other way round, so SoftEther will choose 192.168.4.55 for SecureNAT. That's fine if we don't need to reach 192.168.3.0/24 from the VPN.
* Use IPTables Masquerading to change the source IP to be 192.168.4.55 if the connection is going out of eth1.

### SSL Certificates and Lets Encrypt
Presently the options to configure Let Encrypt, `option_setup_letsencrypt` and `softether_fqdn`, don't actually install or configure Let Encrypt for you. It will however, if you have set up `certbot` yourself and have a certificate in the default location, use that certificate in the VPN server config.

I will be working on another role which works alongside this role to provision them for you (https://github.com/miff2000/ansible-letsencrypt).

# Copyright and license

Code licensed under the [MIT License](http://opensource.org/licenses/MIT), as per [the project this was forked from](https://github.com/softasap/sa-vpn-softether).
