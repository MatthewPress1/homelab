# Phase 1 - OpenBSD: DNS, CA, LDAP

## Overview
OpenBSD provides three main services for the homelab. The first is authoritative DNS resolution for the homelab.internal domain, providing name resolution for all 8 VMs on the 10.42.77.0/24 subnet. The second is an internal Certificate Authority (CA) that signs certificates for all internal services. The third is OpenLDAP to provide centralized user/group accounts across all 8 VMs. OpenBSD was chosen for these services due to its security first design.  

## VM Specs
- OS: OpenBSD 7.6
- RAM: 2048 MB
- Disk: 16 GB
- CPU: 2 cores
- IP: 10.42.77.71 (static)
- Hostname: openbsd.homelab.internal
- Gateway: 10.42.77.2

## Network Configuration
- Static IP set in /etc/hostname.em0
- Default gateway in /etc/mygate
- Hostname set in /etc/myname
- /etc/resolv.conf - "lookup file bind" (nameserver pointing to itself), "homelab.internal" (domain)
- Package mirror in /etc/installurl

## Part A - BIND9 DNS
BIND9 is the authoritative DNS resolver for the homelab.internal domain and a recursive resolver for external names. 

- Installed isc-bind, runs as bind user in chroot jail at /var/named/
- named.conf — two zone declarations, forward and reverse
- Forward zone homelab.internal — A records, MX record, round-robin www
- Reverse zone 77.42.10.in-addr.arpa — PTR records
- Validated with named-checkconf and named-checkzone
- Service managed with rcctl

## Part B - Internal CA
The internal CA is the trust anchor for every encrypted service in the lab. Every VM that runs a network service gets a certificate signed by this CA.

- CA directory /etc/ssl/CA/ chmod 700
- CA private key — AES256 encrypted, passphrase protected
- CA certificate — self-signed, 10 year validity, distributed to every VM as cacert.pem
- Per-VM cert process — private key + CSR + signed certificate
- OpenBSD cert includes SANs for hostname and IP
- VM private keys have no passphrase — services read them at boot

## Part C - OpenLDAP
OpenLDAP provides centralized user/group accounts across all VMs on port 636 (LDAPS). 

- Installed openldap-server, daemon is slapd
- slapd.conf — suffix, rootdn, hashed password, TLS config, access controls
- LDAPS only on port 636 — plaintext port 389 never opened
- Directory structure — base DN, ou=users, ou=groups
- matthew user — UID/GID 9001, posixAccount, inetOrgPerson, shadowAccount
- Two groups — users GID 9001, sysadmin GID 9002

## Validation
- All 8 hostnames resolve forward and reverse
- MX record resolves to freebsd.homelab.internal
- openssl s_client confirms LDAPS with Verify return code: 0 (ok)
- TLSv1.3 negotiated, certificate chain verified against internal CA
- ldapsearch returns matthew's account with correct UID/GID
- pf firewall default-deny, allowing only ports 22, 53, 636 inbound
- All services survive reboot

## Lessons Learned / Troubleshooting
Setting up and working with OpenBSD was challenging because its a little bit different than Ubuntu. I had to get familiar with the different names for commands that I needed. The OpenBSD website and man pages were very helpful when configuring the network and all the config files.

After building out the forward and reverse zone files I learned a lot about DNS, and all the records I used. At first I didn't know what a reverse zone was for but when I did some research I found out it was to resolve IPs to hostnames.

