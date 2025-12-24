3) Fill docs/evidence.md (your screenshots, but curated)

Paste this template:

# Evidence

## Agents onboarded
- Screenshot: `docs/screenshots/01_agents_connected.png`
- Proves: Windows + Linux agents are connected and reporting to Wazuh.

## SSH authentication monitoring
- Screenshot: `docs/screenshots/02_ssh_failed_password_archives.png`
- Proves: Linux SSH failed logins are collected (auth.log) and searchable.

## OPNsense syslog forwarding
- Screenshot: `docs/screenshots/03_opnsense_remote_syslog.png`
- Proves: OPNsense forwards syslog to SEC-SRV over UDP 514.

## Syslog arrival proof (SEC-SRV)
- Screenshot: `docs/screenshots/04_syslog_udp514_tcpdump.png`
- Proves: packets arrive to UDP/514 on the Wazuh server.

## Firewall / IDS visibility in Wazuh
- Screenshot: `docs/screenshots/05_opnsense_logs_archives.png`
- Proves: OPNsense logs (filterlog / Suricata) are ingested into `wazuh-archives-*`.

## Alerts from custom rules
- Screenshot: `docs/screenshots/06_suricata_alerts_rule_100202.png`
- Proves: custom rule promotes Suricata-related events into `wazuh-alerts-*`.