# Phase 2 - FreeBSD: LDAP Client, Postfix, Dovecot, Postfix Relay 

## Overview
FreeBSD is the mail hub for homelab.internal. Postfix accepts inbound mail on port 25 and delivers it to each user's Maildir. Dovecot serves that mail over IMAPS on port 993, where Thunderbird connects to view it. Every other VM in the lab relays system mail through FreeBSD via Postfix. User authentication is handled through the OpenLDAP directory on OpenBSD, meaning no local accounts are needed for everyday tasks. 

## VM Specs
- OS: FreeBSD 14.3
- RAM: 1024 MB
- Disk: 16 GB
- CPU: 2 cores
- IP: 10.42.77.72 (static)
- Hostname: freebsd.homelab.internal
- Gateway: 10.42.77.2

## Network Configuration
- Static IP set in /etc/rc.conf via ifconfig_em0
- Default gateway set in /etc/rc.conf via defaultrouter
- Hostname set in /etc/rc.conf via hostname
- /etc/hosts corrected to reflect freebsd.homelab.internal
- /etc/resolv.conf points to 10.42.77.71 (OpenBSD) for DNS

## Part A - LDAP Client
Connected to matthew's LDAP directory account on OpenBSD 

- Installed nss-pam-ldapd via pkg — provides nslcd, pam_ldap.so, and NSS integration
- /usr/local/etc/nslcd.conf — uri, base DN, ssl on, tls_reqcert demand, tls_cacertfile
- /etc/nsswitch.conf — passwd: files ldap, group: files ldap — changed from compat to files to eliminate warnings
- /etc/pam.d/sshd — pam_ldap.so added as sufficient above pam_unix.so for auth and account
- bash installed via pkg, symlinked /usr/local/bin/bash → /bin/bash — required because matthew's LDAP loginShell is /bin/bash
- nslcd_enable=YES in /etc/rc.conf
- matthew logs in via LDAP with UID 9001, groups users and sysadmin
- sudo access granted via /usr/local/etc/sudoers — %sysadmin ALL=(ALL) ALL

## Part B - Postfix
Postfix is the Mail Transfer Agent (MTA) for homelab.internal, accepting inbound mail from other VMs on port 25 and delivering it locally to each user's Maildir.

- Installed via pkg, sendmail disabled and symlinked to Postfix
- myhostname, mydomain, myorigin set to reflect freebsd.homelab.internal
- mydestination includes $myhostname and $mydomain — FreeBSD is the final destination for homelab.internal mail
- home_mailbox = Maildir/ — delivers to each user's ~/Maildir/ directory
- mynetworks = 10.42.77.0/24, 127.0.0.0/8 — only lab VMs can relay through this server
- TLS configured with smtpd_tls_cert_file, smtpd_tls_key_file, smtpd_tls_CAfile
- smtpd_tls_security_level = may — TLS offered but not required for inbound connections

## Part C - Dovecot
Dovecot is the IMAP server that serves mail stored in each user's Maildir over an encrypted IMAPS connection on port 993 to Thunderbird.

- Installed via pkg, IMAPS on port 993, plaintext port 143 disabled
- mail_location = maildir:~/Maildir in 10-mail.conf
- TLS configured in 10-ssl.conf with FreeBSD cert, key, and CA
- 10-auth.conf — disable_plaintext_auth = yes, auth via PAM
- 10-master.conf — first_valid_gid = 0 required for LDAP users whose primary GID is 9001
- /etc/pam.d/dovecot created manually — not present by default on FreeBSD
- pam_ldap.so added as sufficient in dovecot PAM config, same pattern as sshd
- Thunderbird connects via 10.42.77.72:993 with SSL/TLS

## Part D - Postfix Relay 
Every VM in the lab runs a minimal Postfix relay configuration that forwards all locally generated system mail to FreeBSD for delivery, rather than attempting to deliver mail locally.

- relayhost = [freebsd.homelab.internal]:25 — goes directly to FreeBSD
- smtp_tls_security_level = encrypt — outbound connection to FreeBSD must use TLS
- inet_interfaces = localhost — relay only, Postfix does not listen on the network interface
- mydestination = (empty) — This VM is the final destination for no domains, everything is forwarded to FreeBSD
- Note: loopback-only is Linux Postfix syntax — OpenBSD requires localhost
- /etc/postfix/aliases — root: matthew@homelab.internal forwards system mail to LDAP user

## Validation
- id matthew and getent passwd matthew return UID 9001 on FreeBSD
- matthew can SSH into FreeBSD using LDAP credentials
- sudo works for matthew via sysadmin group
- Mail sent from OpenBSD relays through FreeBSD and delivers to ~/Maildir/new/
- Thunderbird connects as matthew via IMAPS and receives mail
- pf firewall default-deny, allowing only ports 22, 25, 993 inbound
- All services survive reboot: postfix, dovecot, nslcd

## Lessons Learned / Troubleshooting
After working with OpenBSD, I had to get accustomed to different file locations and slightly different command syntax. When I first tried to install a package using pkg, I forgot to use pkg update and ran into a stale cache. Running pkg update resolved the stale cache and the package installed successfully. 

I had a little trouble logging in to matthew's user via LDAP. I looked through log files and found out that my set shell in the LDAP directory was /bin/bash but bash wasn't installed on FreeBSD. Installing bash and symlinking it to /bin/bash resolved the issue, and matthew logged in with the correct UID/GID.


