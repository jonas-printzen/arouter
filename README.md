# A DIY Internet router

## Purpose

As a part of tutoring and sharing knowledge, this is a project about creating
your own gateway / router using **Debian Linux** and **Ansible**. This basic solution is primarily intended to be easy to use starting-point with open source building-blocks. However, this solution can be extended and adapted to handle  the most challenging networking solutions. 

To use this solution you need a **Mini-PC** that can run Debian Linux and has at least two network ports. You can also test and play around with this solution in a virtual environment such as QEMU/KVM  ( See [Testing](#testing) )

We use:

 * Debian 12 Linux - server install
 * nftables - firewall
 * dnsmasq  - DHCP & DNS

## Outline
As the common ground for most cases, this router is present in two or more 
networks, and provide means to control the traffic. One network is considered **external**, the other(s) **_internal_**. It is a router because it can route traffic between the networks. To be safe to use, it is also a **_firewall_**. This means that it can be configured to allow only selected traffic. 

## Usage

Create an Ansible playbook for your host, or group of hosts, and apply it. In this simple example all is included in the file `agateway.yml`, including the variables that define the internal network.

```bash
$> ansible-playbook -K agateway-yml
BECOME password: ************
....
PLAY RECAP *****************
agw: ok=8    changed=5  skipped 1 ...
$>  
```

Once the playbook has been applied your gateway is up and running.

## Ansible

Configuring Ansible starts with the file `ansible.cfg`. Here we stipulate what file contains the hosts and what the default administrative used is. We will use `guru` as the admin user, and the local file `inventory.yml` for the hosts and basic variables.

The all-in-one-playbook `agateway.yml`contains tasks and variables used in the setup. The following parameters, defining the network, can be found as variables near the top of the file.

The **configuration** section, with comments:
```yml
 EXT_NIC: enp1s0             # The network interface towards Internet
 EXT_TCP: ["ssh"]            # Allow ssh from outside ( empty list turns off [])
 LOCAL_NIC: enp2s0           # Network interface on the inside
 LOCAL_NET: 172.16/24        # Subnet used
 LOCAL_CIDR: 172.16.0.1/24   # The IP and netmask bits for the gateway host
 LOCAL_DOMAIN: "demo.org"    # The internal domain
 LOCAL_DHCP: "172.16.0.100,172.16.0.250"  # Use this range for DHCP leases
 LOCAL_STATIC:               # Static IP's the internal DNS knows about
   - { name: agw, ip: 172.16.0.1 }
 LOCAL_MACS:                 # Well kown hosts get fixed IP, identified by MAC
   - { name: asrv, mac: "52:54:00:04:09:3a", ip: 172.16.0.2 }
   - { name: agnome, mac: "52:54:00:2f:a4:a5", ip: 172.16.0.42 }
 LOCAL_CNAME:                # Aliases locally
   - { cname: gw, name: agw }
 DNS_FWD: 8.8.8.8            # Forward DNS-requests if not local domain
```

Here the names of the network interfaces (`enp1s0`,`enp2s0`) as well as the subnet of your local network (`172.16/24`) need to be adjusted to your configuration and hardware.

Better to update this configuration and re-apply the playbook, than manually editing on host.
That way you can save the entire configuration off host and re-apply if you need to.

## Testing

This solution has been tested in a virtual environment using QEMU / KVM. The lab has a typical connection to Internet via a gateway and an internal LAN. The virtual environment mimics the basic setup, treating the lab-LAN as Internet while hiding an internal virtual network at 172.16/24.

![](docs/atest.png)

## Next step

This is the default setup, which can be extended with multiple internal networks, routing and more. In preparation for this, it would make sense to split the configuration in **playbooks**, **roles** and **variables**, which can then be combined to setup differently for each **_router_**, **_server_** and **_workstation_**.

A more structured setup typically looks something like this:

```
├── group_vars
│   ├── all.yml             # Variables common to all hosts
│   ├── routers.yml         # Variables common for routers
│   ├── lan1.yml            # Variables describing lan1
│   └── lan2.yml            # Variables describing lan2
├── host_vars
│   └── agw.yml             # Unique or override variables per host
├── roles
│   ├── router
│   │   └── tasks
│   │       └── main.yml    # Tasks to setup a router
│   ├── lan
│   │   └── tasks
│   │       └── main.yml
│   └── nfs                 # Tasks to setup an internal NFS server
│       └── tasks
│           └── main.yml
└── templates
    ├── dnsmasq.conf.j2
    └── nftables.conf.j2
```

The grouping, represented by `lan1.yml`, and `öan2.yml` suggests an inventory:

```yml
all:
  children:
    routers:
      agw:
    lan:
      asrv:
```

Although a bit more complex, it handles multi-host setup's much better. We may also need to split the root-level playbook into one per **_group_** or **_host_**, referring to **_roles_** for the tasks, and possibly templates.

In this case a root-level playbook looks a bit different:

```yaml
- hosts: routers
  remote_user: "{{ADMIN_USER}}"
  become: true
  roles:
    - router
    - lan
```

Note that hosts inthe the **_routers_** group has two roles, `router`and `lan`.

**_TBC_**