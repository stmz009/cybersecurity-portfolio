# Sithu Min Zin: Cybersecurity Portfolio

Aspiring Threat Hunter & Digital Forensics Analyst, based in Dublin, Ireland
Open to full-time Security Analyst, Threat Hunter, Detection Engineer and Blue Team roles.

[LinkedIn](https://linkedin.com/in/sithu-min-zin-7a5893179) · [Medium](https://medium.com/@sithuminzin969)


## About

I work at the intersection of threat hunting, digital forensics, and detection engineering. Over the past year as a SOC analyst, I've triaged alerts, hunted threats across SIEM and EDR platforms, investigated malware and C2 traffic, and built detection logic that reduced false positives and overall threat exposure for enterprise clients. I'm currently completing an MSc in Cybersecurity at the National College of Ireland, where my research has focused on detecting the kind of cloud-native attacker techniques that traditional, network-based security tools genuinely struggle to see. This repository is where I document that research work, along with the hands-on labs and CTFs I use to keep building depth in forensics and malware analysis.

## Skills snapshot

| Area | Tools & techniques |
|---|---|
| SIEM | Microsoft Sentinel, Elastic Stack, Splunk, Trellix, IBM QRadar |
| EDR / XDR | Trellix EDR, Microsoft Defender for Endpoint, Microsoft Defender XDR |
| Digital forensics & malware analysis | FTK Imager, Volatility, RedLine, Autopsy, KAPE, Sysinternals |
| Threat detection & hunting | KQL, Sigma, YARA, Snort, incident triage, phishing investigation |
| Network analysis | FireEye NX, Zeek, Wireshark, tcpdump, tshark, Sysmon |
| Cloud & log analysis | Microsoft Azure, Entra ID, AWS CloudTrail, Microsoft 365 |
| Adversary emulation | Atomic Red Team, MITRE ATT&CK |
| Security testing | Nmap, Nessus, OpenVAS, Burp Suite, Metasploit, Nuclei, Gobuster |
| Scripting | Python, PowerShell, Bash |

## Featured project

### [Detecting Azure-Native C2 Techniques Through Behavioural Analysis and KQL-Based Threat Hunting](./projects/azure-native-c2-detection/)

My MSc practicum research, built around a question that most organisations haven't fully answered yet: when an attacker hides their command-and-control traffic inside Microsoft's own trusted services (Graph API, Azure Blob Storage) rather than attacker-owned infrastructure, can Microsoft's own security stack actually catch it? I built a controlled Azure lab reproducing two real, documented APT techniques (APT28's Operation Graphite and POLONIUM's Blob Storage abuse), used machine learning to identify which behavioural features actually separated the attack from normal enterprise traffic, and converted those findings into working KQL detection rules in Microsoft Sentinel.

**The core finding:** neither simulated technique triggered a single alert from Sentinel's built-in analytics, despite full telemetry being collected the entire time. The gap wasn't visibility. It was correlation. That result is the whole reason the rest of the project exists.

**Tools used:** Microsoft Sentinel · Microsoft Entra ID · Azure Blob Storage · Microsoft Graph API · KQL · Python · Isolation Forest · Random Forest · Sysmon

→ [Read the full write-up](./projects/azure-native-c2-detection/)

## CTF activity
I take part in CTFs to keep developing my forensics and threat hunting skills, as part of building deeper, hands-on practice in incident analysis and malware analysis, alongside my MSc coursework and lab work. I'll keep this section updated as specific events and write-ups are completed.

→ [CTF activities](./ctf/)

## Certifications

- CompTIA Cybersecurity Analyst (CySA+) 
- eLearnSecurity Certified Threat Hunting Professional (eCTHPv2)
- Multi Cloud Red Team Analyst (MCRTA), CWL

## Education

- MSc in Cybersecurity, 2.1, National College of Ireland, Dublin (2025–2026)
- BSc (Hons) Computing, 1st Class Honours, Edinburgh Napier University, UK (2019–2022)

## Contact

Looking for a full-time Security Analyst, Threat Hunter, Detection Engineer, or Blue Team role in Dublin. Feel free to reach out directly. I'm happy to discuss any of this on a call.
