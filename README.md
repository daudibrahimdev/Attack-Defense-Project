# 🛡️ Attack and Defense Simulation - Siskamjar Lab

This repository contains guides, documentation, and also configuration snippets for running an Attack and Defense simulation lab based on a Security Operations Center (SOC) infrastructure.

## 👥 Development Team
| Name | Student ID | Role / Focus |
| [Daud Ibrahim] | [422310057] | [Blue Team] |
| [Melky Adrians Manafe] | [422320111] | [Blue Team] |
| [Taufiq Hidayat] | [422310050] | [Offensive Detection Engineer] |
| [Pasha Aditya Nugroho] | [422310050] | [Red Team] |

## 🎯 Project Objectives
- Conduct cyberattack simulations (Brute Force Attack, RCE Attack (Reverse shell with Metasploits), and also HID Device Attack, using Digispark Attiny85).
- Create Detection System, to detect Brute Force, HID Attack, and RCE. Since most of the attacks using encryption so with default configuration, It won't leave any footprints in the systems at all. Therefore this project will be integrating Auditd, sysmon or NIDS such as Suricata
- Analyze the attack footprints using a SIEM platform(Wazuh).
- Create Prevention System or IPS to prevent the attack Immediately
- Mapping the rules with MITRE ATT&CK Framework

## 🖥️ Topology & Environment Specifications
This lab runs on a virtualized environment with the following specifications:

* **Attacker Machine (Red Team):**
  * OS: Kali Linux
  * Allocated RAM: 2048 MB (2 GB)
  * IP Address: `[Insert Attacker IP, e.g., 192.168.1.xxx]`
* **Defender Machine / SIEM (Blue Team):**
  * OS: Wazuh OVA
  * Allocated RAM: `4GB is recommended and 3GB minimum`
  * IP Address: `[Insert Wazuh IP]`
* **Target / Victim Machine:**
  * OS: [Ubuntu Machine & Windows]
  * Wazuh Agent: Installed & Connected to Wazuh Manager


> **⚠️ IMPORTANT:** 
> Do not completely replace the default configuration files on your local machine with the files from this repository, as it may cause the system to crash. Simply copy and paste the code snippets below into the appropriate parent tags within your local configuration files.

### 1. Wazuh Agent Configuration (`ossec.conf`)
This configuration is used to forward logs from our endpoint sensors (Auditd, Suricata, Sysmon, and Windows Defender) to the Wazuh Manager. 

Instead of replacing your entire file, open your agent's `/var/ossec/etc/ossec.conf` (Linux) or `C:\Program Files (x86)\ossec-agent\ossec.conf` (Windows) and append the `<localfile>` blocks found in this repository to the `<ossec_config>` section:
* **For Ubuntu Agent:** Refer to the `ossec.conf (ubuntu agent)` file in this repo to ingest Auditd and Suricata (`eve.json`) logs.
* **For Windows Agent:** Refer to the `ossec.conf (Windows Agent)` file to ingest Sysmon and Windows Defender event channels.

### 2. Endpoint Sensors Setup
To ensure attacks are properly captured despite encryption, we utilize several sensors on the victim machines. The configuration files for these sensors are available in this repository:

* **Auditd (`audit.rules` - Ubuntu):** 
  Monitors shell executions (Bash, Python, Netcat), outbound network connections, and process creations. Append the contents to `/etc/audit/rules.d/audit.rules` and restart the auditd service.
* **Suricata (`suricata.yaml` - Ubuntu):** 
  Acts as the Network Intrusion Detection System (NIDS). Replaces the default interface and `eve-log` output configurations in `/etc/suricata/suricata.yaml` to capture binary payloads and extended HTTP data.
* **Sysmon (`sysmonconfig.xml` - Windows):** 
  Filters Windows Event ID 1 (Process Creation) to specifically monitor suspicious `powershell.exe` and `cmd.exe` executions. Apply this configuration using the command: `sysmon.exe -c sysmonconfig.xml`.

### 3. Wazuh Manager Configuration (Detection & Prevention)
The core logic for our SOC resides in the Wazuh Manager. Apply these configurations to enable correlation and Intrusion Prevention System (IPS) capabilities.

* **Custom Decoders (`auditd_custom_decoder.xml`):**
  Parses raw Auditd logs to extract the `srcip` and `srcport` values. Place this file inside the `/var/ossec/etc/decoders/` directory.
* **Correlation Rules (`local_rules.xml`):**
  This file contains the detection logic mapped to the MITRE ATT&CK Framework. It includes rules for:
  * SSH Bruteforce Correlation
  * HID Device Attack (Keystroke Injection via PowerShell)
  * Linux RCE Attack Chain (Correlating Suricata ELF downloads with Auditd outbound connections)
  Append the `<group>` blocks from our `local_rules.xml` into your Manager's `/var/ossec/etc/rules/local_rules.xml`.
* **Active Response / IPS (`ossec.conf`):**
  Configures Wazuh to automatically drop network traffic from attackers using `firewall-drop` when a high-severity rule (like Bruteforce or RCE outbound connection) is triggered. Append the `<active-response>` blocks into your Manager's `ossec.conf`.

*(Note: Always run `systemctl restart wazuh-manager` after applying changes to the manager's configuration).*

---

## ⚔️ Simulation & Attack Execution
Once the environment is fully configured and the agents are connected, you can run the following attack scenarios to test the SOC detection and prevention capabilities:

### Scenario A: Brute Force Attack
1. From the Kali Linux machine, launch a dictionary attack against the Ubuntu machine using Hydra
2. **Expected Result:** Wazuh will correlate the failed login attempts. Upon reaching the threshold (3 attempts in 120 seconds), Rule `111111` will trigger, and the Active Response will execute `firewall-drop` to block the Kali IP for 180 seconds.

### Scenario B: HID Device Attack (Digispark Attiny85)
1. Plug the programmed Digispark Attiny85 into the Windows victim machine.
2. The payload will execute a hidden PowerShell window to bypass execution policies (`-WindowStyle hidden -ep bypass`).
3. **Expected Result:** Sysmon will capture the process creation. Wazuh will trigger Rule `100102` (CRITICAL), identifying potential rogue keystroke injection.

### Scenario C: RCE with Metasploit
1. Generate an ELF payload using `msfvenom` and host it on the Kali machine.
2. Trick the Ubuntu victim into downloading the payload into the `Downloads` directory and executing it.
3. The payload will attempt to initiate a reverse shell connection back to the Kali machine.
4. **Expected Result:** Suricata will flag the ELF download. Auditd will catch the outbound connection from the user-controlled directory. Wazuh will correlate both events (Rule `111112`), flagging an Attack Chain and dropping the connection via Active Response.
