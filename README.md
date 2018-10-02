[![Build Status](https://travis-ci.org/miff2000/softether-vpn-server.svg?branch=master)](https://travis-ci.org/miff2000/softether-vpn-server)

# softether-vpn-server

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
    - my_softether_vpn_users:
      - {
          name: "my_user",
          password: "my_password"
        }

    - my_softether_ipsec_presharedkey: "[1KH;+r-X#cvhpv7Y6=#;[{u"

  roles:
    - {
        role: "miff2000.softether-vpn-server",
        softether_vpn_users: "{{my_softether_vpn_users}}",
        softether_ipsec_presharedkey: "{{my_softether_ipsec_presharedkey}}"
      }
```
> Any options specified in the **Ansible Playbook** or `vars/main.yml` will override the defaults in `defaults/main.yml`

> The above is the very minimum you should specify. You can leave `my_softether_ipsec_presharedkey` as default if you're not planning to use IPSEC. In that case you should disable it with `softether_option_ipsec: false` somewhere in the `role` section above. Remember to add a `,` to the end of all but the last config option

> **Full config options** are in [defaults/main.yml](defaults/main.yml). In the case of all options except `softether_vpn_users`, you can probably leave the defaults

* If you prefer to keep the variables separate from the playbook, you can create a `vars/main.yml` file with options in. For example:

```YAML
softether_option_securenat: True
softether_option_local_bridge: False
option_reset_softether_config: True

# Enable Split Routing (on by default)
softether_option_split_routing: true

# Set the FQDN to use
softether_fqdn: "fqdn-pointing-to-softether-server.com"

# Enable IPSec (default) and set options
softether_option_ipsec: True
softether_ipsec_l2tp: yes
softether_ipsec_etherip: no
softether_ipsec_presharedkey: "[1KH;+r-X#cvhpv7Y6=#;[{u"

# Disable OpenVPN
softether_option_openvpn: False

# VPN users
softether_vpn_users:
  - {
      realname: User 1,
      name: user1,
      password: U4er1Pa55
    }
  - {
      name: user2,
      password: U4er2Pa55
    }
```

* If you do use a separate vars file, You'll then need to include that file in the playbook, like this:
```YAML
- hosts:
    - softether-servers
  vars_files:
    - vars/main.yml
  roles:
    - {
        role: "miff2000.softether-vpn-server"
      }
```

# Copyright and license

Code licensed under the [MIT License](http://opensource.org/licenses/MIT), as per [the project this was forked from](https://github.com/softasap/sa-vpn-softether).
