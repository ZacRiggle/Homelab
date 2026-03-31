# Day 1 — VM Setup, Domain Controller Promotion & Domain Join

**Date:** 2026-03-20

## What I Did

Today I set up the entire Active Directory lab from scratch. This involved creating both VMs in Proxmox, configuring the networks, promoting the server to a domain controller, verifying DNS, and joining the client to the domain.

### Created the VMs in Proxmox

Set up two virtual machines on my Proxmox host:

**DC01 — Windows Server 2025**
- 2 CPU cores, 4GB RAM, 60GB disk
- BIOS: SeaBIOS
- Display: Standard VGA
- Disk bus: SATA
- Network: vmbr0

**CLIENT01 — Windows 10 Pro**
- 2 CPU cores, 4GB RAM, 40GB disk
- Same BIOS/display/network settings as DC01

Both VMs used SATA for the disk bus because VirtIO caused the Windows installer to hang on a black screen. Windows doesn't include VirtIO drivers out of the box, so the installer couldn't detect the disk.

### Renamed Both Machines

Before doing anything with Active Directory, renamed both machines to clean hostnames:
- Server → `DC01`
- Client → `CLIENT01`

### Configured Static IPs

Both VMs are on the same Proxmox bridge (`vmbr0`) so they sit on the same network segment.

| Machine   | IP Address       | Subnet          | Gateway       | DNS Primary      | DNS Secondary |
|-----------|-----------------|-----------------|---------------|------------------|---------------|
| DC01      | 192.168.4.249   | 255.255.255.0   | 192.168.4.1   | 127.0.0.1        | 8.8.8.8       |
| CLIENT01  | 192.168.4.248   | 255.255.255.0   | 192.168.4.1   | 192.168.4.249    | 8.8.8.8       |

Set via: Network adapter → IPv4 Properties → manual configuration.

The primary DNS points to the DC because Active Directory creates special SRV records in its own DNS zone that clients need to locate the domain controller. If the client's DNS points to 8.8.8.8, it has no way to find `ADLab.local`.

### Tested Connectivity

Pings between the VMs initially timed out. Both VMs were on the same bridge with correct IPs, so the issue wasn't networking. After some research I found it was **Windows Firewall blocking ICMP (ping) by default**.

Fixed by enabling the built-in firewall rule on both VMs:
1. Opened `wf.msc` (Windows Firewall with Advanced Settings)
2. Inbound Rules → found "File and Printer Sharing (Echo Request - ICMPv4-In)"
3. Right-click → Enable Rule (enabled for Domain, Private, and Public profiles)

After enabling the rule on both VMs, pings worked in both directions.

### Installed AD DS and Promoted to Domain Controller

On DC01, opened Server Manager → Add roles and features:
- Selected **Active Directory Domain Services**
- Accepted the additional required features
- Installed

After installation I promoted this server to be the domain controller

Configuration:
- **Deployment operation:** Add a new forest
- **Root domain name:** `ADLab.local`
- **Forest/Domain functional level:** Windows Server 2016 (default)
- **DSRM password:** set and saved
- **NetBIOS name:** `ADLAB` (auto-populated)
- Dismissed the DNS delegation warning — expected for a new lab forest

Server rebooted automatically. After reboot, logged in as `ADLAB\Administrator`.

### Set Up DNS Forwarding

Opened DNS Manager from Server Manager → Tools:
1. Right-clicked the server → Properties
2. Forwarders tab → added `8.8.8.8` and `8.8.4.4`

This lets the server resolve internet addresses (google.com, etc.) while still handling AD DNS queries from its own zone.

### Verified DNS

On DC01:
```
C:\>nslookup ADLab.local
Server:  localhost
Address:  127.0.0.1

Name:    ADLab.local
Address:  192.168.4.249
```

On CLIENT01:
```
C:\>nslookup ADLab.local
Server:  DC01.ADLab.local
Address:  192.168.4.249

Name:    ADLab.local
Address:  192.168.4.249
```

Also verified the SRV records that AD needs:
```
C:\>nslookup -type=srv _ldap._tcp.dc._msdcs.ADLab.local
Server:  DC01.ADLab.local
Address:  192.168.4.249

_ldap._tcp.dc._msdcs.ADLab.local    SRV service location:
    target   = DC01.ADLab.local
```

### Joined CLIENT01 to the Domain

1. Right-click This PC → Properties → Rename this PC (advanced)
2. Click Change → selected Domain → typed `ADLab.local`
3. Entered `Administrator` credentials when prompted
4. Got the "Welcome to the ADLab.local domain" message
5. Rebooted

After reboot, clicked "Other user" at the login screen and signed in as `ADLAB\Administrator`. Confirmed the domain join from the server side by checking ADUC — CLIENT01 appeared in the default Computers container.

## Issues I Hit

1. **Windows Server VM stuck on black screen during install** — the disk controller was set to VirtIO, which the Windows installer doesn't have drivers for. Recreated the VM with SATA and the install went through fine.

2. **Pings timed out between VMs** — not a networking problem. Windows Firewall blocks ICMP by default. Enabled the built-in "Echo Request" inbound rule on both VMs to fix it.

## What I Learned

- VirtIO gives better performance but requires driver installation first — use SATA or IDE for the initial Windows install
- Windows Firewall blocks ping by default — easy to forget when troubleshooting connectivity
- DNS is the foundation of Active Directory — the client must be able to resolve the domain name via the DC's DNS before anything else works
- Promoting to a DC automatically installs and configures a DNS zone for the domain
- The DSRM password is an emergency recovery password — the only way to log into the DC if AD is broken
- `nslookup -type=srv _ldap._tcp.dc._msdcs.ADLab.local` is the best way to verify AD DNS is working — if this fails, domain joins will fail too
- DNS forwarding to 8.8.8.8 is what lets domain machines still access the internet while using the DC for internal name resolution
