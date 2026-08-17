# Home Lab IT Helpdesk: Active Directory + osTicket

A self-built home lab simulating a small company's IT environment — Active Directory for identity and device management, Group Policy for enforcement, and osTicket for support ticket tracking — built to gain genuine, hands-on experience with the tools most entry-level IT job postings ask for.

## The Problem

Almost every IT support / help desk job posting lists experience with a ticketing system (osTicket, Zendesk, Freshdesk) and Active Directory as a requirement. But without professional experience, there's no real way to get hands-on with either — you can't practice managing a company's AD environment before you're hired to do exactly that. It's the classic entry-level catch-22: you need experience to get experience.

Rather than wait for that gap to close itself, I built a self-contained lab that simulates a real small-company IT environment from scratch, so I could actually learn — and demonstrate — these skills before ever touching them on the job.

## What I Built

Three virtual machines, networked together as real peers on my home network (Bridged Adapter mode, not NAT), simulating a small organization's IT infrastructure:

| VM | Role | OS | IP |
|---|---|---|---|
| **DC01** | Domain Controller | Windows Server 2022 | `192.168.0.240` (static) |
| **CLIENT01** | Employee workstation | Windows 10 Pro | DHCP |
| **TICKET01** | Ticketing server | Ubuntu Server 24.04 LTS | `192.168.0.150` (static) |

### Active Directory (DC01)
- Installed AD DS and promoted the server, creating a new forest and domain: `syedlab.local`
- Designed an OU structure modeling a real small organization: **IT**, **Customer Service**, **Management**
- Created user accounts and security groups across those OUs
- Built and linked two Group Policy Objects to the IT OU:
  - A minimum password length policy (10 characters)
  - An interactive logon banner, displayed before login on any domain-joined machine in scope
- Verified enforcement end-to-end: joined CLIENT01 to the domain, moved it into the correct OU, and confirmed both GPOs took effect after a `gpupdate /force` and reboot — visually confirmed via the login banner appearing at boot

### Client (CLIENT01)
- Windows 10 Pro (Home edition can't join a domain — Pro/Enterprise/Education required)
- Joined to `syedlab.local`
- Configured to use DC01 as its DNS server, since domain-joined machines rely on the DC for name resolution and locating domain services (SRV records, LDAP, etc.)

### Ticketing System (TICKET01)
- Built a full LAMP stack on Ubuntu Server from scratch: Apache, MySQL, PHP + required extensions
- Created a dedicated, least-privilege MySQL user for the application (rather than using root), scoped only to the osTicket database
- Installed and configured osTicket v1.18.4
- Built out departments mirroring the AD OU structure (IT, Customer Service, Management) and IT-specific help topics (Password Reset, Hardware Issue, Account Access Request)
- Ran a full ticket lifecycle test: submitted a ticket as a simulated employee, claimed it as an agent, replied, and resolved it — end-to-end proof the system works as intended

## Architecture

```
                    Home Router (192.168.0.1)
                              |
        ---------------------------------------------
        |                    |                       |
     DC01                CLIENT01               TICKET01
  Windows Server        Windows 10 Pro         Ubuntu Server
  192.168.0.240          (DHCP)                192.168.0.150
  - AD DS / DNS       - Domain-joined         - Apache/MySQL/PHP
  - GPOs (IT OU)       - DNS -> DC01           - osTicket v1.18.4
```

All three VMs run on VirtualBox with Bridged networking, meaning each one is a full peer on the home network — not hidden behind NAT — so they can all reach each other directly, the same way physical machines on an office network would.

## Challenges & Troubleshooting

**Domain join failure via a silent IPv6/DNS override.** After correctly configuring CLIENT01's IPv4 DNS to point to DC01, domain join still failed with "syedlab.local could not be contacted." `nslookup` revealed the client was actually querying my ISP's DNS server (Cox), not DC01 — despite the IPv4 setting being correct. The root cause: Windows was also attempting IPv6 DNS resolution in parallel, using an IPv6 DNS address auto-assigned by the router, and since the lab domain has no IPv6 presence, that lookup silently failed and interfered with resolution. Disabling IPv6 on the client's network adapter resolved it. This is a genuinely non-obvious real-world networking issue, not a beginner misconfiguration.

**GPOs not applying after domain join.** After joining CLIENT01 to the domain, the login banner GPO didn't appear. The cause: newly domain-joined computers land in the default "Computers" container, not the OU a GPO is linked to. Moving CLIENT01 into the IT OU and forcing a policy update (`gpupdate /force`), followed by a full restart (required since the banner triggers at boot, not on refresh), resolved it.

**MySQL user/permission errors during database setup.** Initial `CREATE USER` / `GRANT` statements failed due to a partially-created user from an earlier attempt. Resolved by explicitly checking `mysql.user` for existing entries and using `DROP USER IF EXISTS` before recreating cleanly.

**Missing `php-imap` package.** Not available in the default Ubuntu repository for this PHP version. Since this extension only affects automatic email-to-ticket conversion (not core ticketing functionality), I proceeded without it rather than pursue a more complex PECL install for a non-essential feature.

## What I Learned

- How Active Directory actually works under the hood: the distinction between a server, a domain, and a Domain Controller; how promotion creates a domain; how OUs (organization/policy) differ from security groups (permissions)
- Practical Group Policy design and enforcement, and how to verify a policy actually reached its target
- Core Linux system administration: package management, file permissions, service management (systemctl), and editing network configuration directly via Netplan
- Deploying a real open-source LAMP application from scratch, including securing a database with least-privilege access
- DNS fundamentals — including a genuinely subtle IPv4/IPv6 interaction bug that doesn't show up in most tutorials
- How to debug systematically: isolating whether a failure is client-side, server-side, or network-level, rather than guessing

## Future Improvements

- **Redundancy:** a real organization would never run on a single Domain Controller — a second DC would remove the single point of failure this lab currently has
- **RODC for branch sites:** in a multi-location org, a Read-Only Domain Controller lets remote sites authenticate locally without a WAN round-trip to the main DC
- **SMTP configuration:** enable real email notifications from osTicket (currently disabled, no mail server configured in this lab)
- **php-imap / email-to-ticket:** allow tickets to be created automatically from incoming support emails
- **Extend the AD lab with the Intrusion Detection project** — since all VMs run on Bridged networking, my existing home network scanner project can already detect these machines on the network, a natural next integration point

## Tech Stack

Windows Server 2022 · Active Directory Domain Services · Group Policy · Windows 10 Pro · Ubuntu Server 24.04 LTS · Apache · MySQL · PHP · osTicket · VirtualBox
