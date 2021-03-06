---
title: Firewall API Architecture and Troubleshooting
markdown2extras: tables, cuddled-lists
apisections:
---
<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2014, Joyent, Inc.
-->

# Firewall API Architecture and Troubleshooting

This document is intended to give an overview of the Firewall API
architecture: its various pieces, and how to interact and troubleshoot
them.



# Components

These are the components that propagate changes to firewalls in SDC:


                        +------------+
                        |            |
          +-------------+  cloudapi  |
          |             |            |
          |             +------+-----+
          |                    |
          |                    |
          v                    v
    +-----------+         +---------+       +---------+
    |           |         |         |       |         |
    |   VMAPI   +-------->|  WFAPI  +------>|  FWAPI  |
    |           |         |         |       |         |
    +-----------+         +---------+       +----+----+
                                                 |
                               +-----------------+-------------+------+
                               |                 |             |  ... |
                               v                 v             v      v
                     +--------------+     +--------------+
                     |              |     |              |
                     |  firewaller  |     |  firewaller  |    firewallers
                     |     CN1      |     |     CN2      |        ...
                     |              |     |              |
                     +--------------+     +--------------+


## On the CN

* rules are managed using the **fwadm** tool
* fwadm converts high-level FWAPI rules into an ipf ruleset for each zone
* rules and remote VMs get stored in **/var/fw/**
* ipf policies are stored in /zones/:uuid/config/ipf.conf
* logs from each fwadm action are stored in **/var/log/fw/logs**


## Interaction with ipfilter

fwadm controls its own ipf instance, which **is separate from the one
visible in the zone**.  The zone can't see or modify the GZ-controlled ipf
rules.  When filtering, the rulesets are evaluated like so:

    Inbound:

        nic ---> [ GZ-controlled rules ] ---> [ per-zone rules ] ---> zone

    Outbound:

        nic <--- [ GZ-controlled rules ] <--- [ per-zone rules ] <--- zone


## Add / Update / Delete rule workflow

Modifying a firewall rule goes to the Firewall API directly, which pushes
the change to the firewaller agents on the Compute Nodes:


        +----------+
        |  client  |
        +----+-----+
             |
             v
    +----------------+
    |                |
    |      FWAPI     |
    |                |
    +-+------+-----+-+
      |      |     |
      |      |     |
      v      v     v
     +--+   +--+  +--+
     |  |   |  |  |  |
     +--+   +--+  +--+

        firewallers


## Provision / Update / Destroy VM workflow

Modifying any of the VM parameters that can affect firewalls (tags and
nics) causes an update to be posted to FWAPI (in the fwapi.update workflow
task), which is then pushed to the firewaller agents on the Compute Nodes:


        +----------+
        |  client  |
        +----+-----+
             |
             v
    +----------------+
    |                |
    |      VMAPI     |
    |                |
    +--------+-------+
             |
             |
             v
        +----------+
        |  WFAPI   |
        +----+-----+
             |
             |
             v
    +----------------+
    |                |
    |      FWAPI     |
    |                |
    +-+------+-----+-+
      |      |     |
      |      |     |
      v      v     v
     +--+   +--+  +--+
     |  |   |  |  |  |
     +--+   +--+  +--+

        firewallers



# Troubleshooting


## Is the firewall enabled?

### Is firewall_enabled set (VMAPI)?

Via VMAPI:

    $ sdc-vmapi /vms/<uuid> | json -Ha firewall_enabled
    true

On the CN:

    # vmadm get <uuid> | json firewall_enabled
    true

    # fwadm status f7dc262f-dd0b-475f-9461-aefb0b6cb2b1
    running


### Is the instance on an SDC 7.0 Compute Node?

There is no fwadm on 6.5.x CNs, and FWAPI is only supported on 7.0 CNs.

Via CNAPI:

    $ sdc-cnapi /servers/$(sdc-vmapi /vms/<uuid> | json -H server_uuid) | json -Ha 'sysinfo["SDC Version"]'
    7.0

On the CN:

    # sysinfo | json "SDC Version"
    7.0


## What VMs does a rule apply to?

Via FWAPI (in the FWAPI zone):

    # fwapi vms <uuid> | json -a uuid
    ae513916-4ea3-cece-dfc0-e28b51cb74fa

On the CN:

    # fwadm vms <uuid>
    ae513916-4ea3-cece-dfc0-e28b51cb74fa


## What rules apply to a VM?

Via FWAPI:

    [root@50716241-ac8d-4e63-a9e4-77ff07cede61 (coal:fwapi0) ~]# fwapi rules <uuid>
    [
      {
        "enabled": true,
        "owner_uuid": "e6fcbc64-3f32-11e2-a144-bf78292e9628",
        "rule": "FROM any TO vm ae513916-4ea3-cece-dfc0-e28b51cb74fa ALLOW udp PORT 54",
        "uuid": "c21912b0-e2b7-471d-aee2-a024d82621ba",
        "version": "1386926482712.093012"
      },
      {
        "enabled": true,
        "owner_uuid": "e6fcbc64-3f32-11e2-a144-bf78292e9628",
        "rule": "FROM any TO vm ae513916-4ea3-cece-dfc0-e28b51cb74fa ALLOW tcp PORT 80",
        "uuid": "e449bf0a-a3f6-4b2f-a426-73bd16b8219a",
        "version": "1386926716940.093012"
      },
      {
        "enabled": true,
        "global": true,
        "rule": "FROM any TO all vms ALLOW icmp TYPE 8 CODE 0",
        "uuid": "27775f65-d377-4979-9c7c-63c9d4f98525",
        "version": "1386743867795.003240"
      }
    ]

On the CN:

    # fwadm rules <uuid>
    UUID                                 ENABLED RULE
    27775f65-d377-4979-9c7c-63c9d4f98525 true    FROM any TO all vms ALLOW icmp TYPE 8 CODE 0
    e449bf0a-a3f6-4b2f-a426-73bd16b8219a true    FROM any TO vm ae513916-4ea3-cece-dfc0-e28b51cb74fa ALLOW udp PORT 54
    c21912b0-e2b7-471d-aee2-a024d82621ba true    FROM any TO vm ae513916-4ea3-cece-dfc0-e28b51cb74fa ALLOW tcp PORT 80


## What rules are being hit on a VM?

On the CN:

    # fwadm stats <uuid>
    0 pass out quick proto tcp from any to any flags S/SA keep state
    0 pass out proto tcp from any to any
    0 pass out proto udp from any to any keep state
    0 pass out quick proto icmp from any to any keep state
    0 pass out proto icmp from any to any
    0 pass in quick proto icmp from any to any icmp-type routerad
    0 pass in quick proto tcp from any to any port = smtp
    0 pass in quick proto icmp from any to any icmp-type echo code 0
    130 block in all


## What remote VMs are stored on a CN?

On the CN:

    # fwadm list-rvms | json -a uuid
    6b1db4be-63bb-11e3-bf59-db53ba2a8cef


## Something has gone wrong - what information should I gather?

### Logs

* In the FWAPI zone: **svcs -L fwapi**
* On the CN: **svcs -L firewaller**
* On the CN: logs in /var/log/fw/ and /var/log/fw/logs


### Rule and Remote VM data

The output of:

    # fwadm list-rvms
    # fwadm list


### Other data to gather

If appropriate:

* A gcore of the firewaller process

