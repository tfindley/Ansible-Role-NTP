# NTP (Chrony)

A simple role for setting up Chrony as an NTP client or NTP server.

## Requirements

Installation of this role requires an internet connection to install Chrony. Should you wish to set your time using a public NTP server, then you will need an internet connection as well as port 151 (outbound) open to allow NTP traffic.

This role makes use of

- Inventory Groups
- Playbook Variables

This role has been tested successfully on:

- Debian 9
- Debian 10
- CentOS 7
- CentOS 8

During the setup the role will attempt to configure FirewallD to open/close port 151 outbound on servers and clients appropriatey. If FirewallD is not installed then this step will fail.

## Background

Network Time is set using NTP Servers. These servers have what is known as a Stratum number / level. The lower the number; the closer to the actual time source the server is.
An RTC (Real time Clock) would be considered a Stratum 1 time source. This would typically be an NTP Server attached to an atomic clock or a GPS clock (set using PPS).
An NTP Server with a Stratum number of 2 is communicating directly with a Stratum 1 server (RTC).
An NTP Server with a Stratum number of 4 is two hops away from the Stratum 1 server (RTC).

NTP Clients also have stratum numbers assigned to them, which will be the NTP Server they are syncing to, +1. If clients were connected to our last example then they would have a Stratum number of 5.

A good NTP server should maintain a low stratum number, therefore it is a good idea to set your internal NTP server to sync from a Stratum 1 or Stratum 2 server. Remember that if you configure your NTP server to have multiple sources which have different stratum levels, your servers stratum number will change depending on where it is getting it's time from.

## Scenarios

This role will work in a number of scenarios

### External NTP Server source, Internal Server

```plaintext
                                                                           --------> Chrony Client
 --------                                          --------              /
| Public |                /-------\               | Chrony |            / 
| NTP    | --Internet--> | Gateway | --Network--> | NTP    | --Network-------------> Chrony Client
| Server |                \-------/               | Server |            \
 --------                                          --------              \
                                                                           --------> Chrony Client
```

This is the most common application for this role. All role targets will be configured as a Chrony Client, unless they are in the NTP_Server group in your Inventory.
Configure the ntp_serverpool with the address of the external NTP server(s) you wish to set the time from. It is recommended that you try to find Stratum 1 servers, but set a Stratum 2 server as well just in case. I suggest using the server that your ISP provides. This ntp_serverpool variable will be set on your NTP Server only, so long as the ntp_localserver variable is set.
Configure the ntp_localserver with the address of your internal Chrony NTP Server(s). This will override the ntp_serverpool variable, configuring all clients to use the internal server only.
Should you need to take your Internal Chrony NTP server(s) out of production, simply remove the ntp_localserver pool variables and clients will revert to using an external time server (See External NTP Server source, below)

### External NTP Server source

```plaintext
                                                    --------> Chrony Client
 --------                                         /
| Public |                /-------\              / 
| NTP    | --Internet--> | Gateway |  --Network-------------> Chrony Client
| Server |                \-------/              \
 --------                                         \
                                                    --------> Chrony Client
```

For this scenario, The role is targeted at all clients you wish to use NTP Client on.
Specify the external NTP Servers using ntp_serverpool variable in the playbook and do not specify the ntp_localserver variable.

### Internal RTC, Internal NTP Server

```plaintext
                                                 --------> Chrony Client
 --------                --------              /
| Int.   |              | Chrony |            / 
| RTC    | --Network--> | NTP    | --Network-------------> Chrony Client
| Server |              | Server |            \
 --------                --------              \
                                                 --------> Chrony Client
```

If you have an internal RTC source for your time, then so long as it is presenting an NTP service you can use this server as your ntp_serverpool. your RTC Server is a Stratum 1 server in this instance. You may wish to configure a stratum 2 backup using the public internet in case the RTC server goes down.

Currently this role does not support configuration of Chrony as a Stratum 1 RTC Source. This functionallity is planned for when I can build myself an Stratu 1 server using a Raspberry Pi, a GPS and a RTC module.

### Internal RTC

```plaintext
                         --------> Chrony Client
 --------              /
| Int.   |            / 
| RTC    | --Network-------------> Chrony Client
| Server |            \
 --------              \
                         --------> Chrony Client
```

You can also set your time directly from the RTC server. Simply set the RTC server address in your ntp_serverpool variable and leave ntp_localserver blank. If you have multiple RTC's then set them all in ntp_serverpool.

## Role Variables

### Optional variables

- ntp_serverpool:
The serverpool is the list of servers you wish to sync the time from. If no local server is specified in ntp_localserver then this variable will be set across all nodes this role touches. If you have specified the ntp_localserver variable then this variable only applies to nodes in the ntp_server inventory group.

- ntp_localserver:
This specifies the local server on your network. For NTP Clients, this will override your ntp_serverpool variable. This does not affect NTP Servers however as they still require a server to be specified.

- ntp_allow:
This variable should be set to your local network range. It's format is 'Network_Range/CIDR' (i.e: 192.168.1.0/24)
If this is not set then your NTP Server will not be addressible by NTP clients.

### Default variables

These can be set in your playbook. The defaults are as below.

- chrony_dumponexit: (Default: true)
- chrony_dumpdir:
- chrony_rtcsync: (Default: true)
- logging_level: (Default: "tracking measurements statistics")
- logging_path: (Default: "/var/log/chrony")
- chrony_changeint: (Default"0.5")

### Fallbacks

If the variable 'ntp_serverpool' is not set in your playbook, then the following defaults will be called on a OS Family or OS Distribution basis under default_serverpool:

#### Debian OS Family

```yml
  - addr: 2.debian.pool.ntp.org
    type: pool
    mode: iburst
    offline: true
```

#### RedHat OS Family

```yml
  - addr: "0.centos.pool.ntp.org"
    mode: iburst
    type: server
    offline: false
  - addr: "1.centos.pool.ntp.org"
    mode: iburst
    type: server
    offline: false
  - addr: "2.centos.pool.ntp.org"
    mode: iburst
    type: server
    offline: false
  - addr: "3.centos.pool.ntp.org"
    mode: iburst
    type: server
    offline: false
```

## Dependencies

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.

## Example Playbook

This NTP playbook example shows a configuration for an upstream NTP server which may be communicating with a stratum 1 timesource (in this case, 3) and acting as a time source of truth itself for the local network. In this instance it is allows only two servers to communicate with it using the 'allow' list. These two servers act as relays for the rest of the network

```yml
- hosts: linux
  become: yes
  gather_facts: true
  roles:
    - role: 'ntp'
  vars:
    chrony:
      pool:
        - addr: "tick.usask.ca"
          mode: iburst
          type: server
        - addr: "68.69.221.61"
          mode: iburst
          type: server
        - addr: "ntp1.torix.ca"
          mode: iburst
          type: server
      server:
        enabled: true
        allow:
          - '192.168.69.5/32'
          - '192.168.69.6/32'
      firewall:
        enabled: true
      authkeys:
        - id: 05
          hash: SHA1
          key: 'HEX:DDE741FFD9D919222DFB832CFB31CA8A4C45C0422A3601788C8FE2847ECF22DBEFE554B76CCEE41BFEA39F3C95DA6A1F287F9DDA4892DA653E1DD547B9CDCBCD'
        - id: 06
          hash: SHA1
          key: 'HEX:328DB77DAEFE7E2B6B4C035B690F4CE84C475DB6B82F2F8E28181734CBBA2CA5B5BC5534DB5E71614C2FAC6D592389E997229C48B38B5F705FB47EA42BFB720C'

```

The following playbook example shows a configuration where the server you're configuring will communicate with another Chrony server. In this instance, the target chrony server (located on 192.168.69.2) already has key ID 05 generated. We have to load that symetric key onto the new server and tell it to communicate with the existing chrony server identifying with key ID 5.

```yml
- hosts: linux
  become: yes
  gather_facts: true
  roles:
    - role: 'ntp'
  vars:
    chrony:
      pool:
        - addr: "192.168.69.2"
          mode: iburst
          type: server
          offline: false
          keyid: 05
      server:
        enabled: true
        allow:
          - '192.168.69.0/24'
      firewall:
        enabled: true
      authkeys:
        - id: 05
          hash: SHA1
          key: 'HEX:DDE741FFD9D919222DFB832CFB31CA8A4C45C0422A3601788C8FE2847ECF22DBEFE554B76CCEE41BFEA39F3C95DA6A1F287F9DDA4892DA653E1DD547B9CDCBCD'
```

## Keys

### key Generation

Run the following command to from a system with Chrony installed to generate a symetric key:

```shell
chronyc keygen 11 SHA1 512
11 SHA1 HEX:DDE741FFD9D919222DFB832CFB31CA8A4C45C0422A3601788C8FE2847ECF22DBEFE554B76CCEE41BFEA39F3C95DA6A1F287F9DDA4892DA653E1DD547B9CDCBCD
```

## Roadmap

- Consider use of ntp_serverpool and ntp_localserver , possibly changing them to ntp_serverpool and ntp_clientpool and adjusting template files accordingly.
- Support NTP RTC time source server (requires purchase of additional hardware and testing)
- Support more Chrony configuration options.
- Allow serverpools to be set at an inventory level instead of at a playbook level.

## License

BSD

## Author Information

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
