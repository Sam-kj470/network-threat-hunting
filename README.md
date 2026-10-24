# network-threat-hunting
Enterprise network threat hunting case study. Ingested and parsed multi-format event logs using Python (Pandas) to uncover credential compromise, internal port scanning, and automated data exfiltration.
# Network Threat Hunting & Incident Analysis Report

## 📝 What this Project Does
This project details a comprehensive threat-hunting investigation mapped from a real-world security incident affecting an enterprise healthcare network. By ingesting and analyzing structured XML firewall logs, raw VPN authentication logs, asset inventories, and threat intelligence feeds, I successfully traced an advanced persistent threat (APT) actor through the complete lifecycle of a network intrusion. 

## 💡 Why it is Useful
Modern cyber threats rarely trigger generic endpoint alerts; they often blend in with normal system behaviors using valid, stolen credentials. This project demonstrates my ability to conduct data-driven threat hunting using **Python (Pandas) inside Google Colab**, parse multi-format log arrays, establish behavioral baselines, and isolate anomalous indicators of compromise (IoCs) that automated tools missed.

## 🚀 How to Get Started
To review the data sets and analysis tracking notebooks:
1. Navigate to the `/notebooks` directory to view the programmatic data-cleaning and matching sequences.
2. Review the structured intelligence indicators mapped out in the core report below.

---

## 🔍 Incident Timeline & Attack Lifecycle Breakdown

The raw logs were parsed, enriched with device names from the corporate asset inventory, and correlated by timestamp (UTC) to establish the explicit attack progression:

### 1. Initial Access & Impossible Travel Detection
- **Compromised Asset:** User account `r.okafor` logged in successfully to the corporate VPN.
- **Anomalous Indicator:** The account established a valid connection from an IP address based in Bulgaria (`212.39.182.107`) just 4 hours and 34 minutes after a valid session was logged from Canada. Cross-referencing threat intelligence feeds flagged this specific IP as an anonymizing proxy and known credential-replay node. The absence of failed login attempts points directly to stolen session hijacking or an unpatched NetScaler vulnerability.

### 2. Internal Reconnaissance (Port Scanning)
- **Assigned VPN Address:** `10.20.40.76` (Isolated as the only active dynamic host during the threshold window).
- **Network Probing:** The dynamic host executed an aggressive internal sweep, mapping 120 connection attempts across 30 distinct internal assets targeting 7 administrative ports (`22`, `135`, `139`, `445`, `1433`, `3389`, `5985`). While 119 attempts were blocked, it successfully mapped the network's domain controllers and file servers.

### 3. Lateral Movement & Server Infiltration
- **Target Infiltrated:** Critical imaging file server `FS-IMG-01` (`10.20.20.13`).
- **Data Access:** The adversary established 21 stateful SMB connections over port 445, successfully interacting with internal directories and pulling approximately 20.9 MB of structural assets back to the VPN node.

### 4. Command and Control (C2) Beaconing
- **External C2 Pivot:** `45.147.248.192:8080`
- **Behavoiral Pattern:** The compromised imaging server began outbound check-ins exactly every 5 minutes (301-second intervals with a highly steady standard deviation of just 21 seconds). Because an internal file server has no legitimate structural requirement to initiate outbound internet sockets, this stable, automated timing fingerprint confirmed a C2 beaconing script.

### 5. Malicious Data Exfiltration
- **Exfiltration Endpoint:** `45.147.236.166:443`
- **Data Loss:** The server executed 41 encrypted outbound transfers over a 48-hour window, pushing roughly 214 MB of data. This data transfer went unnoticed by standard volume trackers because it occurred during the nightly backup window and represented just 0.1% of the network's massive 195 GB nightly backup size.

---

## 🎛️ Triage of Decoy Baseline Findings

To avoid false positives, three high-volume automated cycles discovered in the logs were analyzed and triaged as benign activity:
- **Nightly Offsite Backup:** Server `BKP-01` pushing ~195 GB to an established repository every night between 02:03 and 03:09. Classified as standard business baseline.
- **Password Typos & Expired Kiosks:** `KIOSK-17` generating 3,384 failed directory authorization events using service account `svc_kiosk` due to an expired password loop. Typos were triaged from malicious brute-forcing based on target isolation.
- **Pre-Existing Monitoring Agents:** `APP-EHR-01` executing persistent 60-second pings to an outside address. Triaged as benign because the zero-variance cycle was active long before the attack window.

---

## 🛡️ Remediation & Security Controls Formulated

Based on the forensic logs, I delivered the following operational recovery plan to the SOC:
1. **Firewall Blocks:** Immediately sever traffic to and from malicious network infrastructure nodes `45.147.248.192` and `45.147.236.166`.
2. **Session Revocation:** Disconnect active remote terminal connections, revoke current NetScaler tokens, and disable the compromised `r.okafor` account.
3. **Host Isolation:** Isolate `FS-IMG-01` from the standard network segment without powering down the hardware to preserve highly volatile memory and active connections for live disk imaging.
4. **Outbound Control Policy:** Implement a Zero-Trust egress policy on data center switches to block all outbound internet loops from internal file servers by default, allowing connections only to pre-approved application endpoints.

