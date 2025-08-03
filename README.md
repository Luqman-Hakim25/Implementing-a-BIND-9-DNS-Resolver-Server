# Implementing a BIND 9 DNS Resolver Server

## Overview

This project demonstrates the deployment of a secure and privacy-focused internal DNS resolver using **BIND 9** on **Debian 12**. It was designed to enhance DNS control, improve internal network efficiency, and mitigate privacy and security concerns related to third-party DNS services. By leveraging technologies such as **ACLs**, **DNSSEC**, **RPZ**, and **rate limiting**, the resolver can protect the network from unauthorized queries, DNS spoofing, and access to malicious domains.

## Table of Contents

- [Overview](#Overview)
- [BIND 9](#BIND-9)
- [Features](#Features)
- [Installation & Configuration (DNS Resolver)](#Installation-&-Configuration-(DNS-Resolver))
  - [Prerequisites](#Prerequisites)
  - [Installation Steps](#Installation-Steps)
  - [Configuration Files](#Configuration-Files)
  - [Configure BIND 9 as a Secure DNS Resolver](#Configure-BIND-9-as-a-Secure-DNS-Resolver)
    - [Configure Main BIND Files](#Configure-Main-BIND-Files)
    - [Add Root Hints](#Add-Root-Hints)
    - [Configure Logging (Optional)](#Configure-Logging (Optional))
    - [Configure RPZ(Response Policy Zones)](#Configure-RPZ(Response-Policy-Zones))
- [Automation with cron (update RPZ)](#Automation-with-cron-(update-RPZ))
  - [1.Sources for RPZ Data](#1.Sources-for-RPZ-Data)
  - [2.Install Necessary Tools](#2.Install-Necessary-Tools)
  - [3.Prepare the Script to Download and Format RPZ Files and set the file permission](#3.Prepare-the-Script-to-Download-and-Format-RPZ-Files-and-set-the-file-permission)
  - [4.Make the Script Executable](#4.Make-the-Script-Executable)
  - [5.Schedule the Script with Cron](#5.Schedule-the-Script-with-Cron)

## BIND 9

<img width="320" height="135" alt="image" src="https://github.com/user-attachments/assets/6260f8f9-a573-4de4-aad0-19fc53a50728" />

BIND 9 (Berkeley Internet Name Domain version 9) is a premier open-source DNS server software developed by the Internet Systems Consortium (ISC). Renowned for its robustness and flexibility, BIND 9 serves a crucial role in the DNS infrastructure, capable of acting as both an authoritative name server and a recursive resolver. It supports a comprehensive array of DNS features, including IPv6, DNSSEC (DNS Security Extensions), and dynamic DNS updates, ensuring enhanced security, reliability, and functionality. BIND 9 is designed to be highly scalable, making it suitable for deployment in various environments, from small networks to large enterprises and ISPs. Its vast configuration options allow administrators to fine-tune performance and security settings according to specific needs. BIND 9 also offers extensive logging and statistics, aiding in effective monitoring and troubleshooting. Furthermore, it supports modern DNS protocols and standards, ensuring compatibility with other DNS implementations.

## Features

- Recursive DNS resolution for internal clients
- DNSSEC support for query integrity
- ACL (Access Control List) to restrict query access
- RPZ (Response Policy Zone) to block phishing/malicious domains
- DNSTAP and logging for monitoring and analysis
- DNS query caching for performance optimization
- Protection against DNS spoofing
- RPZ zone automation threat updates

## Installation & Configuration(DNS Resolver)

### Prerequisites

- Debian 12 (Bookworm)
- BIND 9 (`sudo apt install bind9 bind9utils bind9-doc`)
- Curl, cron, and standard CLI tools

### Installation Steps

Install BIND 9
```
# Update system
sudo apt update && sudo apt upgrade

# Install BIND 9
sudo apt install bind9 bind9utils bind9-doc bind9-host dnsutils
```

Verify Installation

```
named -v
# Should output something like: BIND 9.18.x
```

### Configuration Files

- `/etc/bind/named.conf.options`
- `/etc/bind/named.conf.local`
- `/etc/bind/named.conf.logging`
- `/etc/bind/root.hints`
- RPZ zone files: `rpz.malware.db`, `rpz.phishing.db`
- Automation script: `update-rpz.sh`

Refer to the `/config` and `/scripts` folders for sample configurations.

### Configure BIND 9 as a Secure DNS Resolver

#### Configure Main BIND Files

```
sudo nano /etc/bind/named.conf
```

```
sudo nano /etc/bind/named.conf.options
```

```
sudo nano /etc/bind/named.conf.local
```

#### Add Root Hints

```
sudo curl -o /etc/bind/db.root https://www.internic.net/domain/named.cache
```

Ensure it’s referenced in `/etc/bind/named.conf.default-zones`.

#### ACL – Access Control List

```
acl internal-networks {
    127.0.0.1;
    192.168.1.0/24;
};
```

Only client IP Address that only belong inside this list would be allow to use this BIND 9 DNS. If want to add client, should be include the IP address here.

#### Configure Logging (Optional)

```
sudo nano /etc/bind/named.conf.logging
```

Create the log directory:

```
sudo mkdir -p /var/log/named
sudo chown bind:bind /var/log/named
```

#### Configure RPZ(Response Policy Zones)

Add to `/etc/bind/named.conf.options`:

```
response-policy {
    zone "rpz.malware.local";
    zone "rpz.phishing.local";
};
```

Declare Zones

In `/etc/bind/named.conf.local`:

```
zone "rpz.malware.local" {
        type master;                       // Define as a master zone
        file "/etc/bind/rpz.malware.db";  // File containing malware domain rules
        allow-query { localhost; };
        allow-transfer { localhost; };
};

zone "rpz.phishing.local" {
        type master;                       // Define as a master zone
        file "/etc/bind/rpz.phishing.db"; // File containing phishing domain rules
        allow-query { localhost; };
        allow-transfer { localhost; };

};
```

Create RPZ Files using nano

```
sudo nano /etc/bind/rpz.malware.db
```

```
sudo nano /etc/bind/rpz.phishing.db
```

## Automation with cron (update RPZ)

To automate the process of updating the RPZ zone files with data from public feeds, I create a script that regularly downloads the feeds, formats them into the RPZ zone file format, and then reloads the BIND service. Here’s a step-by-step guide to set this up.

### 1.Sources for RPZ Data

reliable source for the RPZ data (malware and phishing domains). Some public feeds include:

- Abuse.ch (Malware Domain Blocklist)
- OpenPhish (Phishing URL Feed)
- Blocklist.de (Malware Feed)
- Zeus Tracker (Malicious Domains)

In this project, I use this two public feeds:
- **Malware**: `https://urlhaus.abuse.ch/downloads/hostfile/`
- **Phishing**: `https://openphish.com/feed.txt`

### 2.Install Necessary Tools

Make sure the system has the necessary tools to download and process the data. You can install curl (to fetch the data) and awk or sed (to format it):

```
sudo apt update
sudo apt install curl awk sed
sudo apt install gawk
awk --version
```

### 3.Prepare the Script to Download and Format RPZ Files and set the file permission

Just download the script "update-rpz.sh" that i provided inside Automation folder

set the appropriate permissions for the script file:
Yes, you need to set the appropriate permissions for the script file `/usr/local/bin/update-rpz.sh` to ensure it can be executed securely by the system and authorized users.

**Recommended Permissions**
Ownership: Set the file to be owned by root (or another system administrator account):

```
sudo chown root:root /usr/local/bin/update-rpz.sh
```

Permissions: Allow only the owner (root) to write to the file, while other users can execute it:

```
sudo chmod 755 /usr/local/bin/update-rpz.sh
```

7: Owner can read, write, and execute.
5: Group and others can read and execute but cannot modify.

### 4.Make the Script Executable

```
sudo chmod +x /usr/local/bin/update-rpz.sh
```

Run the script manually to ensure it works as expected

```
sudo /usr/local/bin/update-rpz.sh
```

Verify:
The RPZ files (rpz.malware.db and rpz.phishing.db) are updated with properly formatted records and also check The bind9 service reloads without errors:

```
sudo systemctl status bind9
```

Check the log file for updates:

```
sudo cat /var/log/rpz-update.log
sudo tail -f /var/log/rpz-update.log
```

To check for errors in the /etc/bind/rpz.phishing.db file, you can use the named-checkzone command, which is specifically designed to validate BIND zone files. Here's the command to verifies the syntax and content of a zone file:

```
sudo named-checkzone rpz.phishing.local /etc/bind/rpz.phishing.db
```

### 5.Schedule the Script with Cron

To automate the process, you can schedule this script to run at regular intervals (e.g., daily or weekly) using cron.

Edit the crontab for root

```
sudo crontab -e
```

Add the following line to run the script daily at 2am:

```
0 2 * * * /usr/local/bin/update-rpz.sh
```

This cron job will:

- Run the script at 2:00 AM every day.
- Update the RPZ files and reload BIND.

