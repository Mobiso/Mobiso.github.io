+++
title = "Building Networked Systems Security Project"
date = "2026-04-03T16:01:18+02:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting

description = "In the KTH course Building Networked Systems Security (EP2520) we built a mostly virtual network. I (out of 5 people) was responsible for implimenting Keycloak, Grafana and Grafana Loki, Promtail, an LLM log analyzer pipeline and a so called critical webserver."
showFullContent = false
readingTime = false
hideComments = false
+++
I was part of 5-person team tasked by a fictional company named "ACME" to setup a secure network for their company. We were given a list of requirements and a general description (I am unsure of if I am allowed to copy and paste it here, so just to be sure I wont). Our job was to analyze these requirements and impliment a functional network solution. 

I was responsible for implimenting Keycloak, Grafana and Grafana Loki, Promtail, an LLM log analyzer pipeline and a so called critical webserver.

[Download full report (PDF)](/BNSS/BNSS.pdf)

There is an error in the local Firewall rules for the "critical web server". The `DENY ALL` default rules require some additional allowed connections that we forgot to include in the report as they were added before the presentation of the project. I am writing this sometime after the course has ended and honestly do not remember the exact rules added.
## Derived Requirements
The following requirements were derived:
1. Manage employee identities.
2. Secure communication between London and Stockholm offices.
3. Allow users outside the London and Stockholm offices to use a VPN to connect to the Stockholm
or London offices. VPN users are only allowed access to certain less critical services.
4. Prevent DoS attacks from interrupting communication between offices.
5. Intrusion detection using LLMs in conjunction with traditional rule-based systems.
6. Logging of critical servers and traffic.
7. Secure file sharing and communication between employees.
8. Certificate-based authentication and identification; VPN users require additional 2FA.
9. Certificate revocation.
10. Secure WiFi and network authentication.
11. Automatic renewal of certificates for public-facing web servers.
12. Protect against DNS spoofing and hijacking.
13. Physical security and side-channel attacks are out of scope.

## Solution
The identities and accounts of employees is managed using FreeIPA LDAP (1).

Secure communication between the London and Stockholm offices is implemented using a permanent WireGuard VPN tunnel (2). Network segmentation with VLANs ensure
that only authorized traffic passes and that only less critical services are available to remote VPN users
(3). Firewalling and routing is handled by OpenWRT.

An LLM-based system is deployed to analyze logs pulled from a Grafana Loki log server (5,6), while
a traditional IDS (SNORT) monitor network traffic in real time (5). In particular, logs from
an additionally secured web server, honeypot and router is collected using Promtail and `SCP`.

Employee certificates is issued and revoked using the CA server Dogtag built into FreeIPA (8,9).

Secure file sharing between employees is facilitated by an Nextcloud server (7).

Authentication to services is handled through Keycloak (8), which enforces certificate-based login or password/username + 2FA. 

WiFi and network authentication (10) use WPA3-Enterprise with EAP-TLS and FreeRADIUS for
authentication. The access point and router run OpenWRT to enforce secure network policies.

Public company web servers will be located in a DMZ and secured with Let’s Encrypt certificates (11).

To protect against common DNS attacks such as spoofing and hijacking, a DNS server implementing
DNSSEC will be deployed (12).

No solution for (4) was implimented. 

## Topology
![Network Topology](/BNSS/acme_topology.png)