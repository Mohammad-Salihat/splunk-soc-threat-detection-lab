# splunk-soc-threat-detection-lab
This project demonstrates the deployment and configuration of a Security Information and Event Management (SIEM) solution using Splunk Enterprise. The objective was to simulate Security Operations Center (SOC) activities by ingesting security and network logs, developing detection rules, configuring alerts, building dashboards, and investigating a multi-stage cyberattack.

The environment was based on Thornbury Wealth Partners (TWP), a fictional multinational financial services organization operating across London, New York, and Toronto. The simulated attack involved credential compromise, privilege escalation, data exfiltration, persistence mechanisms, and ransomware deployment.

# Objectives
- Deploy Splunk Enterprise on macOS
- Configure indexes and data sources
- Implement role-based access control (RBAC)
- Develop security detection logic using SPL
- Configure automated alerts
- Build a SOC dashboard
- Investigate attacker activity
- Produce actionable security findings

# Environment
- Splunk Enterprise

## Detection Rules
- SEC-001 Brute Force Detection
- SEC-002 Account Compromise
- SEC-003 Privilege Escalation
- SEC-004 Data Exfiltration
- SEC-005 New User Creation
- SEC-006 Ransomware Activity
- SEC-007 Threat Intelligence Match

## Attack Timeline

1. Brute force attack
2. Account compromise
3. Privilege escalation
4. Persistence
5. Data exfiltration
6. Ransomware deployment

## Data Sources
- security_events.cs: Windows security events
- network_traffic.csv:	Network traffic analysis
- threat_intel_ips.csv:	Threat intelligence matching

## Skills Demonstrated

- Splunk Enterprise
- SPL
- Detection Engineering
- Threat Hunting
- Incident Response
- Dashboard Development
- Alerting
- SIEM Operations
