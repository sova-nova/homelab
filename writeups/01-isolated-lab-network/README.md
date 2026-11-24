# 01: Isolating a security lab behind an OPNsense firewall

**Summary:** I built an isolated network inside my Proxmox server for security labs. Lab VMs sit on a virtual bridge with no physical port, and an OPNsense VM is the only way out. A firewall rule lets the lab reach the internet but blocks it from my home network, and I verified that with tests from a lab client.

## Why

My Proxmox host already runs household services, including a password manager and a NAS. I wanted to run vulnerable VMs and attack tools on the same hardware, which means a compromised or misbehaving lab machine must not be able to reach any of that. Segmentation and firewalling are also core skills in both IT and security work, so building it by hand was part of the point.

**Constraints:** one physical host, 32 GB RAM (about 16 GB free for lab use), no managed switch, so no real VLANs. Virtual bridges do the job for now.

## Design

```
Internet ── home router ── vmbr0 ── [OPNsense WAN | OPNsense LAN] ── vmbr1 ── lab VMs
                                      (10.10.10.1/24 on LAN)
```

- `vmbr0`: the existing bridge for my normal home network.
- `vmbr1`: a new bridge with **no physical port and no IP on the host**, so only guests can use it, and the host itself isn't reachable from it.
- OPNsense VM: two NICs, one on each bridge. It routes and NATs lab traffic out through the WAN side.

## Build

### 1. Create the isolated bridge

In the Proxmox UI (**node → System → Network → Create → Linux Bridge**), I named it `vmbr1` and left the address, gateway, and bridge ports empty. The resulting config is:

```
auto vmbr1
iface vmbr1 inet manual
        bridge-ports none
        bridge-stp off
        bridge-fd 0
```

### 2. Create the firewall VM

| Setting | Value |
|---|---|
| Guest | OPNsense `<version>`, installed from the official ISO |
| CPU / RAM / disk | 2 cores, 2 GB, 20 GB |
| `net0` | `vmbr0`, VirtIO (WAN) |
| `net1` | `vmbr1`, VirtIO (LAN) |

I turned off Proxmox's own firewall option on both NICs so it wouldn't interfere with OPNsense's rules. In the console menu I assigned `vtnet0` as WAN and `vtnet1` as LAN, gave the LAN the address `10.10.10.1/24`, and enabled DHCP for `10.10.10.100`–`10.10.10.200`. I also gave the WAN a DHCP reservation on my home router so its address stays fixed.

### 3. Add a client

A small desktop Linux VM on `vmbr1` served two purposes: it was the only way to reach OPNsense's web UI, which listens on the LAN side, and it was the test machine for the rules.

### 4. Write the isolation rules

OPNsense evaluates rules top to bottom and uses the first match. I created an alias `RFC1918` containing `10.0.0.0/8`, `172.16.0.0/12`, and `192.168.0.0/16`, then ordered the LAN rules like this:

| # | Action | Source | Destination | Why |
|---|---|---|---|---|
| 1 | Pass | LAN net | This Firewall | Lab VMs need DNS and the GUI on the firewall itself |
| 2 | Block | LAN net | `RFC1918` | Stop lab VMs reaching my home network |
| 3 | Pass | LAN net | any | Allow everything else, i.e. the internet |

Rule 1 has to come before rule 2 because the firewall's own address, `10.10.10.1`, falls inside `10.0.0.0/8` and would otherwise be blocked, which breaks DNS for the lab.

## Verification

From the lab client:

| Test | Expected | Result |
|---|---|---|
| Ping `8.8.8.8` and load a website | Works | Works |
| Open `https://10.10.10.1` | Works | Works |
| Ping the home router | Blocked | Timed out |
| Ping the Proxmox host's address | Blocked | Timed out |
| Ping a household service on the home LAN | Blocked | Timed out |

![LAN firewall rules in OPNsense](images/firewall-rules.png)
*The three LAN rules, in order.*

![Failed pings to home LAN addresses from the lab client](images/isolation-test.png)
*The block rule doing its job. Note the timeouts.*

I also watched **Firewall → Log Files → Live View** while pinging, which showed each blocked packet matching rule 2.

## Problems and fixes

**The VM wizard only allows one network device.** I couldn't add `vmbr1` while creating the VM. The fix was to finish the wizard with `net0`, then open **Hardware → Add → Network Device** for the second NIC before first boot.

**Understanding rule order in the OPNsense UI.** I wasn't sure which rules to delete. The answer was that I didn't need to delete anything: the default "allow LAN to any" rule already does the job of my rule 3, so I only had to add the two new rules and move them above it. `[Edit: describe what you actually did, e.g. whether you moved or replaced the default rule.]`

## Limitations

This isolates the lab from the home network, but it doesn't segment the lab from itself. VMs on `vmbr1` can still reach each other, which is intended for now since they'll form a domain and its targets. It also doesn't defend against a hypervisor escape. The honest threat model is "contain a misbehaving lab guest", not "contain a determined attacker with a VM breakout."

## What I learned

- First-match rule evaluation, and why ordering matters as much as the rules themselves.
- Why the firewall's own address needs an explicit allow before a broad RFC1918 block.
- A bridge with no uplink and no host address is a cheap, effective way to build a private network on a single machine.

## Next

Stand up a Windows Server domain controller on this network and build out an Active Directory domain (write-up 02).
