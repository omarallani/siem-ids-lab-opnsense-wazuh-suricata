# Lab Architecture (OPNsense + Wazuh + Suricata) — VirtualBox

This lab simulates a small company network and implements an end-to-end visibility pipeline:
**Endpoints + Firewall/IDS → Central SIEM (Wazuh) → Search/Investigation (Dashboards)**.

---

## 1) VirtualBox networking model

### OPNsense VM (3 adapters)
- **WAN (NAT)**: Internet access (VirtualBox NAT)
- **MGMT (Host-only)**: firewall GUI access from the host machine  
  - Example: `192.168.56.0/24` (host PC ↔ OPNsense Web UI)
- **LAN (Internal Network)**: company internal traffic  
  - `10.10.0.0/24`

### Internal LAN subnet
- **LAN gateway (OPNsense):** `10.10.0.1/24`
- Internal VMs (servers/clients/attacker) live in `10.10.0.0/24`

---

## 2) VMs & IP plan

| VM Name | IP | OS | Role |
|---|---:|---|---|
| **OPNsense** | `10.10.0.1` | FreeBSD | Gateway/Firewall + Suricata IDS/IPS + syslog source |
| **SEC-SRV** | `10.10.0.30` | Ubuntu Server | **Wazuh SIEM** (Manager + Dashboard + Indexer) |
| **APP-SRV** | `10.10.0.20` | Ubuntu Server | Target server (SSH service + logs) |
| **Windows Staff / Windows Client** | (LAN DHCP) | Windows | Endpoint telemetry via Wazuh agent |
| **Ubuntu Client 2** | (LAN DHCP) | Ubuntu | Endpoint telemetry via Wazuh agent |
| **KALI-ATTACKER** | `10.10.0.102` | Kali Linux | Attack simulation (nmap / SSH attempts) |

> Notes:
> - `SEC-SRV` is the SIEM “core”.
> - `APP-SRV` is your main test target (SSH service).
> - Clients can be DHCP as long as they can reach `SEC-SRV` and OPNsense.

---

## 3) Logging & telemetry flows (the important part)

### A) Endpoint telemetry → Wazuh
- **Windows agent → SEC-SRV**
  - Event collection (Security/System/etc.) sent by Wazuh agent
- **Ubuntu Client 2 / APP-SRV agent → SEC-SRV**
  - Linux logs (especially authentication) are collected and shipped to Wazuh

**Key Linux log source used in this lab:**
- `/var/log/auth.log` (SSH logins, failed password attempts)

### B) OPNsense → Wazuh (syslog)
OPNsense forwards logs to the Wazuh server using **Remote Syslog**:

- **Source:** OPNsense `10.10.0.1`
- **Destination:** `SEC-SRV 10.10.0.30`
- **Protocol/Port:** **UDP / 514**

On `SEC-SRV`:
- `rsyslog` listens on UDP 514 and writes incoming OPNsense logs to:
  - `/var/log/opnsense-remote.log`
- Wazuh manager reads that file and indexes events into:
  - `wazuh-archives-*` (raw ingested events)
  - `wazuh-alerts-*` (events promoted to alerts by rules)

---

## 4) Suricata placement (IDS/IPS)

Suricata runs on **OPNsense** and inspects traffic on the LAN side.

- Ruleset: **ET Open** (or equivalent)
- Output events appear in Wazuh as syslog entries (often tagged as `suricata`)
- Goal: visibility of recon/scans and suspicious activity produced by Kali simulations

---

## 5) What “proof” looks like in the SIEM

### Archives (raw events)
- Index: `wazuh-archives-*`
- You should see:
  - OPNsense logs (`filterlog`, `opnsense`, `suricata`)
  - Linux auth logs (`sshd`, “Failed password”)

### Alerts (rule-triggered)
- Index: `wazuh-alerts-*`
- You should see:
  - at least one working custom rule (example: your `rule.id 100200` / `100202`)

---

## 6) Services & ports (high level)

### Wazuh (on SEC-SRV)
- Agent events ingestion: **1514** (Wazuh default)
- Agent enrollment/registration: **1515** (Wazuh default)
- Dashboard (web UI): **5601** (common for Dashboards)
- Indexer/OpenSearch: **9200** (backend)

### OPNsense syslog forwarding
- Syslog: **UDP 514** → `SEC-SRV`

---

## 7) Attack simulations used (lab validation)

- **Recon / Port scanning**
  - From `KALI-ATTACKER (10.10.0.102)` → target(s) in LAN
  - Goal: create Suricata + firewall visibility in Wazuh archives/alerts
- **SSH authentication testing**
  - From Kali → `APP-SRV (10.10.0.20)` SSH attempts
  - Goal: generate `auth.log` entries and validate detection in Wazuh

---

## 8) Quick diagram (logical flow)

KALI / Clients / Servers (10.10.0.0/24)
        |
        |  LAN traffic
        v
   OPNsense (10.10.0.1)  ---- syslog UDP/514 ---->  SEC-SRV (10.10.0.30)
   (Firewall + Suricata)                         (Wazuh SIEM + Dashboards)

Endpoints (Windows/Linux) --- Wazuh agent ---> SEC-SRV (Wazuh manager)
