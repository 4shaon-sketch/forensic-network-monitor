# Forensic-Network-Monitor and Investigation
Design LU based Forensic Network for detecting Unauthorized Remote Access (RDP, SSH, Telnet)

1.	Introduction 
This project involves the design and development of a Laurentian University (LU) based forensic network lab, which will simulate, capture, and analyze unauthorized remote access attacks on three protocols. The team started by setting up a controlled environment using VirtualBox to deploy an attacker machine (Kali Linux) and two target machines (Ubuntu Server and a Virtual Machine of Windows 10 installed with RDP). We used a credential-stuffing tool called Hydra to launch attacks, capturing all the traffic using Wireshark for analysis.

This report addresses the following five forensic objectives:
1.	Apply display filters for SSH, RDP, and Telnet traffic in Wireshark
2.	Identify source and destination IP addresses associated with remote sessions
3.	Extract login attempts and session details from captured traffic
4.	Detect potential brute-force login attempts and suspicious activity patterns
5.	Generate a detailed security report summarizing remote access findings
















2.	System Architecture & Technology Stack 
2.1	Lab Topology

 
Figure-01: Network Architecture
 
Machine	Role	IP Address	Services
Kali Linux	Attacker	192.168.137.10	Hydra, Nmap
Ubuntu Server 20.04	Target Host 1	192.168.137.20	SSH (22), Telnet (23)
Windows 10 VM (RDP)	Target Host 2	192.168.137.1	RDP (3389)



2.2	Technology Stack

Tool / Technology	Category	Purpose
VirtualBox	Hypervisor	Hosts all VMs in an isolated host-only network
Kali Linux	Attacker OS	Provides offensive security tools 
Hydra	Attack Tool	Automated credential brute-force across SSH, Telnet, RDP
Nmap	Reconnaissance	Port scanning and service version detection
Wireshark	Packet Capture	GUI-based deep packet inspection and stream analysis

3.	Methodology 
The project followed a five-phase methodology from environment setup through forensic analysis:

Phase	Name	Description
01	VM Network Setup	We configured 3 VMs with host-only adapters and static IPs and enabled SSH (port 22) and Telnet (port 23) on Ubuntu; RDP (port 3389) on Windows 10 VM. Intentionally we make weak credentials for attack simulation.
02	Service Deployment	Verification have done for all services are reachable. Performed Nmap SYN scan from Kali to confirm open ports: 22, 23 on 192.168.137.20 and 3389 on 192.168.137.1.
03	Attack Simulation	We used Hydra for brute-force attacks against all three protocols using custom wordlists (users.txt, passwords.txt). Attacks run with controlled parallelism (-t 4).
04	Capture & Filter	By Wireshark we capture the packet on appropriate NICs prior to attacks. Filters applied for ports 22, 23, and 3389. PCAP files saved for offline forensic analysis.
05	Analysis & Alerts	Sessions classified as successful or failed based on byte count, duration, protocol messages, and plaintext content.

3.1 The attacker machine Kali Linux IP:  192.168.137.10









Figure-02: Attacker machine Kali Linux
3.2 Ubuntu Machine Internal LAN for SSH and Telnet IP address: 192.168.137.20













Figure-03: Ubuntu Server


3.3 Windows 10 VM for RDP IP:  192.168.137.1

 
Figure-04: Windows 10

4.	Identify the Open ports for the Target 

Before launching attacks, Nmap was used from the Kali attacker machine to identify open ports and running services on both target hosts.

4.1 Ubuntu Server (192.168.137.20)
nmap -sS -sV 192.168.137.20

Port 22 / TCP	OPEN — OpenSSH 8.2p1 Ubuntu-4ubuntu0.5
Port 23 / TCP	OPEN — Linux telnetd (Ubuntu 20.04)
OS Detection	Linux 5.4.0-144-generic (Ubuntu 20.04.6 LTS)




Figure-05: Port Scan for Ubuntu Server (192.168.137.20)
4.2 Windows 10 VM / XRDP (192.168.137.1)
nmap -sS -sV -p 3389 192.168.137.1
Port 3389 / TCP	OPEN — RDP (Remote Desktop Protocol)
NLA Support	Enabled — Network Level Authentication active
OS Detection	Microsoft Windows 10 












Figure-06: Port Scan for Window 10 machine (IP 192.168.137.1)
5.	Brute Force attack 
5.1 Creating password list
For the brute force attach we create custom wordlist for user list and password

Directory location /usr/share/wordlists/

Custom username file: users.txt
Custom password file: passwords.txt















Figure-07: Creating custom users list on users.txt








Figure-08: Creating custom passwords on passwords.txt


5.2 Starting Wireshark and Select appropriate Ethernet Lan
 
Figure-08: Starting Wireshark with proper ethernet port 




5.3 Hydra in Kali Linux

Command Syntax
hydra -L users.txt -P passwords.txt ssh://<TARGET_IP_ADDRESS> -t 4 -V

Parameter Breakdown:
•	-L: Specifies a file containing a list of usernames (use -l for a single username instead).
•	-P: Specifies a file containing a list of passwords (use -p for a single password instead).
•	ssh://: Defines the target protocol and the destination IP address or domain.
•	-t 4: Limits the parallel tasks to 4 tasks. Note: Setting this too high may crash the target SSH service or trigger built-in rate-limiting.
•	-V: Enables verbose mode so you can see each connection attempt in real-time.

5.4 Launch the attack for Ubuntu SSH login 
The Hydra command used to launch the SSH brute-force attack:
hydra -L users.txt -P passwords.txt ssh://192.168.137.20 -t 4 -V


Figure-09: Launch hydra brute force attack to SSH service 

















Figure-10: Successfully crack the password
5.5 Wireshark Packet analysis and filter for SSH

•	Filter: tcp.port == 22
•	Go to Statistics → Conversations → TCP tab 
•	Sort by Bytes or Duration — longest/largest sessions = successful logins 
•	Right-click a long session → Apply as Filter → Selected 
•	Follow the TCP stream to inspect











Figure-11: SSH Traffic Analysis
SSH encrypts all data after the banner exchange, including credentials. Successful login attempted determine by Key Exchange Init of SSH protocol 
Property	Value
Attacker IP	192.168.137.10
Target IP	192.168.137.20:22
Total Session	64
Successful logins	1
Time	2026-06-19T01:44:37.72
Username	shaon
Password	Encrypted due to SSH traffic
tcp.stream eq	25
Key Indicator	Server: Key Exchange Init, mean successfully login
filter	tcp.port == ssh
			Table-01:  Wireshark Packet analysis for SSH

 
Figure-12: SSH Key handshake and encryption Traffic 

5.6 Launch Attack for Telnet port 23 
The Hydra command used to launch the Telnet brute-force attack:
hydra -d -l shaon -P passwords.txt telnet://192.168.137.20
Figure-13: Launch brute force attack on Kali Linux




Figure-14: Successfully crack the password for Telnet.
5.7 Telnet Traffic Analysis
Successful Telnet session detail:
Property	Value
Attacker IP	192.168.137.10
Target IP	192.168.137.20
Successful logins	1
Time	2026-06-21T01:07:50.93
Username	shaon
Password	123456
tcp.stream eq 	2
TCP Source Port	59800
Indicator	The password shows in plaintext
Filter	telnet.data contains "Password" || telnet.data contains "login"
Filter	tcp.port == 23 && tcp.srcport == 59800
Table-02:  Wireshark Packet analysis for Telnet















Figure-15: Password shows in plain text in Wireshark for telnet 


5.8 RDP brute force to Windows VM port 3389 

The Hydra command used to launch the Telnet brute-force attack:
hydra -L users.txt -P passwords.txt rdp://192.168.137.1 -t 4 -V


Figure-16: Successfully crack the password for RDP.







5.9 RDP Traffic Analysis

•	Filter: tcp.port == 3389 
•	Go to Statistics → Conversations → TCP tab 
•	Sort by Bytes or Duration — longest/largest sessions = successful logins 
•	Right-click a long session → Apply as Filter → Selected 
•	Follow the TCP stream to inspect

tcp.port == 3389 && ip.addr == 192.168.137.1
tcp.stream eq 21

Check for NLA negotiation:
tls.handshake.extensions_server_name




Figure-17: Apply Filter for Wireshark for RDP 

5.9 Detect potential brute-force login attempts or suspicious activity patterns
Property	Value
Attacker IP	192.168.137.10
Target IP	192.168.137.1:3389
TCP Source Port	48038
Successful logins	1
Time	2026-06-23T10:20:00.674710119-0400 ET
Username	shaon
Password	NLA-encrypted, not recoverable from PCAP
tcp.stream eq 	21
Key Indicator	Extended Application Data exchange after TLS, confirmed credential success
filter	tcp.port == 3389
			Table-03:  Wireshark Packet analysis for RDP



6.	Correlation Finding 
Correlating findings across all three Wireshark analysis reveals a unified attack campaign:
•	Attacker IP (192.168.137.10) across all three protocols.
•	Username 'shaon' was the successful credential in both RDP (mstshash cookie) and Telnet (plaintext) and is the confirmed valid account and we found password in telnet session.
•	Password '123456' (confirmed via Telnet) is likely the same password used in successful RDP and SSH sessions.
•	The prior login note in the Telnet welcome banner confirms the attacker already had a foothold these PCAPs may capture a return visit, not the initial intrusion.


7.	Summary Statistics
Metric	RDP	SSH	Telnet
Total Sessions	55	68 (64 active)	7
Failed Attempts	54	67	6
Successful Logins	1	1	1
Attacker IP	192.168.137.10	192.168.137.10	192.168.137.10
Target	192.168.137.1:3389	192.168.137.20:22	192.168.137.20:23
Credentials Visible	Username only (cookie)	None (encrypted)	Username + Password
Confirmed Username	shaon	shaon	shaon
Confirmed Password	Not recoverable	Not recoverable	123456
Attack Duration	~127 seconds	~192 seconds	~5 seconds
Attack Tool	Hydra / RDP client	Hydra (libssh_0.11.3)	Hydra / telnet client





8.	Conclusion

The Project has designed and implemented a forensic network lab to detect and analyze unauthorized remote access to RDP (Remote Desktop), SSH (Secure Shell) and Telnet protocols. The environment is a realistic (also controlled) setting that allows for simulation of real-world credential brute force attacks from a Kali Linux attacking machine by using Hydra.

An analysis of the three PCAP files captured revealed that the attacker (192.168.137.10), managed to successfully logon to all three remote access services. The Telnet session was the most important, providing the full plaintext credentials (username: shaon, password: 123456). The RDP analysis provided the username through the mstshash cookie, and SSH sessions were identified by analyzing the behavior of the sessions; how long they last, and the number of bytes transmitted.
