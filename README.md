# Network Access Control (NAC) Lab: FreeRADIUS + nftables

*Logical architecture: client RADIUS requests are authenticated by FreeRADIUS; upon success, the client’s IP is dynamically added to an nftables set, granting network access.*

## Overview

This project implements a functional **Network Access Control (NAC)** system using **FreeRADIUS** for AAA (Authentication, Authorization, Accounting) and **nftables** for dynamic traffic filtering. When a client successfully authenticates via RADIUS, its IP address is automatically added to an nftables set, allowing network access for a limited time. Unauthenticated or unauthorized clients are blocked at the firewall level. The lab demonstrates core NAC concepts and is fully contained in a virtualized environment.

## Objectives

- Set up a **RADIUS server** (FreeRADIUS) to authenticate users or devices.
- Enforce **authorization** based on RADIUS responses.
- Implement **dynamic network segmentation** using nftables.
- Automate IP allow‑listing upon successful authentication.
- Validate the complete AAA flow with real RADIUS clients.
- Demonstrate that only authenticated clients can communicate through the firewall.

## Technologies Used

| Component          | Purpose                                                       |
|--------------------|---------------------------------------------------------------|
| **FreeRADIUS 3.x** | RADIUS server providing authentication and authorization      |
| **nftables**       | Linux firewall for dynamic packet filtering                   |
| **Ubuntu Server**  | Host OS for both NAC server and client                        |
| **radclient**      | RADIUS client utility for testing                             |
| **Bash scripting** | (Optional) Automation of IP set management                    |

## Lab Architecture

┌─────────────────┐
│ RADIUS Client │ e.g., ub1 (192.168.88.10/24)
│ (radclient) │
└────────┬────────┘
│ RADIUS requests (UDP 1812)
▼
┌─────────────────┐
│ NAC Server │ srv1 (192.168.88.17)
│ ┌───────────┐ │ - FreeRADIUS
│ │ FreeRADIUS│ │ - nftables
│ └─────┬─────┘ │ - Allowed IPs set
│ │ │
│ ┌─────▼─────┐ │
│ │ nftables │ │ Set "allowed_ips" with timeout
│ └───────────┘ │
└────────┬────────┘
│ Forwarded traffic (if IP in allowed_ips)
▼
┌─────────────────┐
│ Authorized │
│ Network │
└─────────────────┘
text


All VMs are connected to the same isolated network (e.g., `192.168.88.0/24`). The NAC server acts as a gateway; only clients whose IPs are in the nftables set can forward traffic to the authorized network.

## Setup & Installation

### 1. Prepare the NAC Server (srv1)

Install FreeRADIUS and nftables:

```bash
sudo apt update
sudo apt install freeradius nftables -y
sudo systemctl enable freradius nftables
```

### 2. Prepare the RADIUS Client (ub1)

Install the RADIUS utilities for testing:
```bash

sudo apt install freeradius-utils -y
```


### 3. Configure FreeRADIUS
#### 3.1 Define RADIUS Clients
Edit /etc/freeradius/3.0/clients.conf and add the client network:
```bash

client ub1 {
    ipaddr = 192.168.88.0/24
    secret = testing123
    shortname = labclient
}
```

#### 3.2 Add a Test User

Edit /etc/freeradius/3.0/users and create a user:


bob   Cleartext-Password := "secretbob"

### 4. Configure nftables for Dynamic NAC

Create or edit /etc/nftables.conf with the following ruleset:
```bash

#!/usr/sbin/nft -f

flush ruleset

table inet nac {
    set allowed_ips {
        type ipv4_addr
        flags timeout
        timeout 1h
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
        ip saddr @allowed_ips accept
    }
}
```

This defines:

    A set allowed_ips with a default timeout of 1 hour.

    A forward chain that drops all traffic by default, but accepts packets from source IPs present in allowed_ips.

Check the configuration and restart nftables:
bash

sudo nft -c -f /etc/nftables.conf
sudo systemctl restart nftables
sudo systemctl status nftables

### 5. Integrate FreeRADIUS with nftables

FreeRADIUS can be extended to add the client’s IP to the nftables set upon successful authentication. This can be done via the exec module or a custom script. For this lab, we will manually verify the flow and later automate it.

A simple approach: create a script /usr/local/bin/add_ip.sh:
```bash

#!/bin/bash
IP="$1"
nft add element inet nac allowed_ips { $IP }
```

Make it executable: sudo chmod +x /usr/local/bin/add_ip.sh.

Then in FreeRADIUS, in the post-auth section of /etc/freeradius/3.0/sites-enabled/default, add:
```text

exec add_ip {
    wait = yes
    program = "/usr/local/bin/add_ip.sh \"%{Calling-Station-Id}\""
}
```

    Note: Calling-Station-Id usually holds the client MAC address; you may need to adjust based on your RADIUS client attributes. Alternatively, use the source IP from the request. A more robust solution would extract the NAS‑IP or Framed‑IP.

For simplicity, the lab will first test with manual addition, then demonstrate automation.
## Testing & Validation
### 1. Test RADIUS Authentication with radclient

On the client (ub1), create a request file request.txt:
```text

User-Name = bob
User-Password = secretbob
NAS-IP-Address = 192.168.88.10
NAS-Port = 0
```

Send the request to the RADIUS server:
```bash

radclient -x 192.168.88.17 auth testing123 < request.txt
```

Expected output:
text

Sent Access-Request ...
Received Access-Accept ...

### 2. Verify Dynamic IP Addition

On the NAC server, check the nftables set:
```bash

sudo nft list set inet nac allowed_ips
```

If the integration is active, the client’s IP should appear. If not, manually add the IP to test the firewall behavior:
```bash

sudo nft add element inet nac allowed_ips { 192.168.88.10 }
```

Now the client should be able to forward traffic through the server (e.g., ping or connect to a service on the authorized network). Test by pinging from ub1 to an IP beyond the server (simulate an internal network).
### 3. Verify Blocking

Remove the IP from the set (or wait for timeout) and confirm that traffic is dropped:
```bash

sudo nft delete element inet nac allowed_ips { 192.168.88.10 }
```

Now pings or connections from ub1 should fail.
## Results

  Authentication: RADIUS correctly authenticates user bob with password secretbob.

  Authorization: After successful authentication, the client’s IP can be added to the nftables allow set.

  Dynamic enforcement: nftables allows traffic only from IPs in the set, with a configurable timeout.

  Full AAA flow demonstrated: The lab proves that a client gains network access only after RADIUS authentication, and access is automatically revoked after a defined period or upon manual removal.

## Lessons Learned

  FreeRADIUS is highly extensible: Its exec module allows integration with external scripts, making it possible to update firewall rules dynamically.

  nftables sets with timeouts provide a simple yet powerful way to implement temporary network access.

  RADIUS attributes like Calling-Station-Id or Framed-IP-Address are essential for mapping authentication events to network‑layer actions.

  Testing with radclient is invaluable for debugging RADIUS configurations without a full NAS device.

  Separation of control and data – RADIUS handles control (authentication), while nftables enforces the resulting policy.

## Future Improvements

  Replace manual scripting with a dedicated FreeRADIUS module (e.g., rlm_exec or rlm_perl) for more robust integration.

  Use VLAN assignment instead of IP‑based filtering for true network segmentation.

  Integrate with a wireless access point or switch that supports RADIUS‑based VLAN assignment (802.1X).

  Add accounting (RADIUS accounting) to track client session times and data usage.

  Deploy a web portal for guest access with temporary credentials.

  Automate the entire lab setup with Ansible or Vagrant for reproducibility.

Author

Esso Maléki TONINZIBA
