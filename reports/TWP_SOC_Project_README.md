# Splunk SOC Project — Thornbury Wealth Partners (TWP)

A self-built Security Operations Center simulation in Splunk, modeling a multi-day, multi-geography breach against a fictional UK wealth management firm. Built end-to-end: role-based access control, synthetic log data, 7 SPL detection queries, scheduled alerting, and a live SOC dashboard.

**Stack:** Splunk Enterprise 9.x · SPL (Search Processing Language) · CSV log simulation · Classic Dashboards

---

## Table of Contents

- [Project Summary](#project-summary)
- [Simulated Environment](#simulated-environment)
- [Architecture: RBAC Setup](#architecture-rbac-setup)
- [Data Pipeline](#data-pipeline)
- [SPL Fundamentals Used](#spl-fundamentals-used)
- [Detection Queries](#detection-queries)
- [Alerting](#alerting)
- [Dashboard](#dashboard)
- [Verification & QA](#verification--qa)
- [Skills Demonstrated](#skills-demonstrated)
- [Incident Walkthrough](#incident-walkthrough)

---

## Project Summary

This project simulates a SOC analyst's full workflow at a fictional UK-based wealth management and investment banking firm, **Thornbury Wealth Partners (TWP)**, across three offices (London, New York, Toronto). The dataset models a realistic attack chain — credential compromise, privilege escalation, data exfiltration, and ransomware — and the project builds the detection and response layer around it from scratch:

- Configured **role-based access control** for a simulated SOC team
- Designed a **fictional company environment** (users, hosts, subnets, regulators) to ground the data in something realistic
- Loaded and indexed **synthetic security and network logs** representing a live attack
- Wrote **7 SPL detection queries** covering the full kill chain, from initial access to impact
- Converted each query into a **scheduled, severity-tiered alert**
- Built a **single-pane SOC dashboard** with KPIs, timelines, and geographic breakdowns
- Used the resulting dataset to **manually trace the incident** end-to-end

> A companion deep-dive on the manual incident trace is available in [`incident-investigation-report.md`](incident-investigation-report.md).

---

## Simulated Environment

| Attribute | Detail |
|---|---|
| Headquarters | London, UK |
| US Branch | New York, NY |
| Canada Branch | Toronto, ON |
| Industry | Wealth Management, Investment Banking, Equity Trading |
| Regulators | FCA (UK), SEC (US), OSC (Canada) |
| Why attackers target firms like this | Client portfolios, SWIFT wire credentials, M&A intelligence, insider trading data |

**Internal IP addressing:**

| Office | Subnet |
|---|---|
| London HQ | `192.168.10.0/24` |
| New York | `10.10.0.0/24` |
| Toronto | `10.20.0.0/24` |

**Key hosts modeled in the dataset:**

| Host | Role |
|---|---|
| `TWP-LON-DC01` / `DC02` | London Domain Controllers |
| `TWP-LON-SWIFT01` | SWIFT wire transfer system |
| `TWP-LON-FILE01` | London file server — ransomware target |
| `TWP-LON-SQL01` | Trading database |
| `TWP-NYC-DC01` | NY Domain Controller |
| `TWP-TOR-FILE01` | Toronto file server — ransomware target |
| `TWP-LON-WS-EPEMBERTON` | Head of Risk's workstation — ransomware origin point |

**Cast of users (with simulated outcomes):**

| User | Role | Office | Outcome |
|---|---|---|---|
| oliver.harrington | Senior Equities Trader | London | Brute-forced & compromised |
| james.fitzgerald | IT Security Lead | London | Brute-forced & compromised |
| charlotte.whitfield | Compliance Director | London | Attempted, failed |
| emma.pemberton | Head of Risk | London | Workstation = ransomware origin |
| william.ashworth | CFO | London | Attempted, failed |
| sophie.beaumont | Wealth Advisor | London | Normal baseline activity |
| marcus.delgado | VP Investment Banking | New York | Attempted, failed |
| rachel.sterling | Equity Analyst | New York | Workstation used in exfiltration |
| anthony.romano | Helpdesk Technician | New York | Maliciously promoted to Domain Admin |
| priya.bhattacharya | Portfolio Manager | Toronto | Attempted, failed |
| connor.macdonald | Junior Analyst | Toronto | Added to Schema Admins |

Plus four **service accounts** — `svc_backup` (compromised, used in ransomware), `svc_swift` (added to Enterprise Admins), `svc_bloomberg`, `svc_sql_prod` — included specifically to demonstrate why non-human identities need just as much monitoring as people.

---

## Architecture: RBAC Setup

Before any analyst touches log data, access itself has to be modeled correctly. I configured Splunk's built-in role-based access control following least-privilege principles:

| Role | Capability | Assigned To |
|---|---|---|
| `admin` | Full platform control — users, indexes, configs | Platform administrator only |
| `power` | Create alerts/saved searches, manage lookups | SOC Lead, Threat Intel Analyst |
| `user` | Run searches, view dashboards, no admin actions | Tier 1 SOC Analysts, Compliance |
| `can_delete` | Delete indexed events (paired with another role) | Data management only |

I also built a **custom role** (`twp_soc_readonly`) scoped to specific indexes (`security`, `network` only) — a pattern that matters in financial services, where some staff should see network telemetry but never security event logs.

**Role-mapping logic applied to the TWP org chart:**

| Job Title | Role Assigned | Rationale |
|---|---|---|
| Head of Security Operations | `power` / `admin` | Needs to create alerts, manage dashboards |
| Junior Security Analyst | `user` | Investigates alerts; no admin needs |
| Threat Intel Analyst | `power` | Writes/updates lookup tables |
| Compliance Officer | `user` | Read-only for FCA reporting |
| Platform Administrator | `admin` | Manages indexes, users, system config |

This mirrors a real onboarding/offboarding control: when staff leave, the practice is to immediately strip roles and reset credentials — retained access by former employees is a stated FCA/SEC regulatory risk.

---

## Data Pipeline

| Step | Action |
|---|---|
| 1 | Created two indexes: `security` and `network` |
| 2 | Uploaded `security_events.csv` → `security` index |
| 3 | Uploaded `network_traffic.csv` → `network` index |
| 4 | Uploaded `threat_intel_ips.csv` as a **lookup table** (not an index) — used to cross-reference live traffic against known-bad IPs |
| 5 | Set time range to **All Time** on every search — data is fixed-dated (26 April 2026), a common beginner trap |

The lookup-table pattern (Step 4) is deliberately different from the index pattern — it demonstrates the difference between *time-series event data* (indexes) and *static reference data* (lookups), and how Splunk's `join` command bridges the two.

---

## SPL Fundamentals Used

SPL works like a pipeline — each `|` passes the result set to the next command.

```
Raw events --> Filter --> Transform --> Format --> Result
 index=     |   search  |    stats   | table |  [output]
```

| Command | Purpose |
|---|---|
| `index=` | Select the data source |
| `field=value` | Filter raw events |
| `stats` | Aggregate / group / count |
| `where` | Filter **after** aggregation (vs. filtering raw events) |
| `eval` | Create computed fields |
| `bucket _time span=Xm` | Group events into fixed time windows |
| `case(...)` | Multi-branch conditional logic |
| `table` | Select display columns |
| `sort -field` | Order results |
| `timechart` | Time-series aggregation with automatic charting |
| `inputlookup` | Read from a static lookup table instead of an index |
| `join type=inner` | Combine indexed events with lookup/subsearch results |

---

## Detection Queries

Seven SPL detections were written to cover the full kill chain — from initial access through to impact — each with its own logic, thresholds, and tuning rationale.

### SEC-001 — Brute Force Login Detection
```spl
index=security EventCode=4625
| bucket _time span=5m
| stats count AS failed_logins BY _time, src_ip, user
| where failed_logins >= 5
| table _time, src_ip, user, failed_logins
| sort -failed_logins
```
**Logic:** 5+ failed logins (`EventCode=4625`) from the same IP/user within a 5-minute window. **Result:** flagged `oliver.harrington` (8 failures) and `james.fitzgerald` (6 failures), plus failed attempts against the CFO, NY VP, and Toronto PM.

### SEC-002 — Account Compromise
```spl
index=security (EventCode=4625 OR EventCode=4624)
| bucket _time span=10m
| stats count(eval(EventCode="4625")) AS failures,
        count(eval(EventCode="4624")) AS successes
        BY _time, user, src_ip
| where failures >= 3 AND successes >= 1
| eval risk_score = failures * 10
| table _time, src_ip, user, failures, successes, risk_score
| sort -risk_score
```
**Logic:** the dangerous escalation of SEC-001 — failures *followed by* a success from the same IP/user. Uses conditional counters (`count(eval(...))`) to track both outcomes in one pass. **Result:** confirmed compromise of both Oliver Harrington (risk score 80) and James Fitzgerald (risk score 60) — the second being far more severe given his IT Security role.

### SEC-003 — Privilege Escalation
```spl
index=security (EventCode=4728 OR EventCode=4732 OR EventCode=4756)
| eval group_type = case(
    EventCode="4728", "Global Security Group",
    EventCode="4732", "Local Security Group",
    EventCode="4756", "Universal Security Group"
  )
| table _time, user, Account_Name, Group_Name, group_type, host
| sort -_time
```
**Logic:** tracks Windows security-group membership changes (Global/Local/Universal). **Result:** Anthony Romano (Helpdesk) added to Domain Admins by the compromised IT Security Lead account; `svc_swift` added to Enterprise Admins; Connor MacDonald added to Schema Admins.

### SEC-004 — Data Exfiltration
```spl
index=network direction=outbound action=allow
| stats sum(bytes_out) AS total_bytes BY src_ip, dest_ip, dest_port
| where total_bytes > 50000000
| eval total_mb = round(total_bytes/1024/1024, 2)
| table src_ip, dest_ip, dest_port, total_mb
| sort -total_mb
```
**Logic:** flags any internal host sending more than 50 MB to a single external destination over allowed (not blocked) traffic. **Result:** 5 hosts flagged, led by 142 MB from Oliver Harrington's workstation to a known APT C2 server.

### SEC-005 — New User Account Created
```spl
index=security EventCode=4720
| table _time, Account_Name, user, src_ip, host
| sort -_time
```
**Logic:** Windows `EventCode 4720` fires on every new account creation — in a regulated firm, an unauthorized account is also an FCA reporting issue. **Result:** three accounts flagged — `temp_auditor`, `backdoor_admin` (an overtly malicious name), and `svc_test`.

### SEC-006 — Ransomware Behaviour
```spl
index=security EventCode=4663 Object_Type=File
| bucket _time span=1m
| stats count AS file_ops BY _time, user, host
| where file_ops > 200
| table _time, host, user, file_ops
| sort -file_ops
```
**Logic:** ransomware encrypts at machine speed; 200+ file operations per minute (3+/sec) is far beyond human capability. **Result:** 3 hosts flagged, peaking at 340 ops/min on the London file server via the compromised `svc_backup` service account.

### SEC-007 — Threat Intelligence Match
```spl
| inputlookup threat_intel_ips.csv
| rename ip AS dest_ip
| join type=inner dest_ip
    [search index=network
     | stats count, values(src_ip) AS internal_src BY dest_ip]
| table dest_ip, threat_type, severity, source, internal_src, count
| sort -count
```
**Logic:** the only detection that starts from a lookup table rather than an index — joins a known-bad IP list against live network traffic via subsearch. **Result:** confirmed contact with an APT C2 server, a financial-malware C2, a TOR exit node, and known banking-trojan infrastructure.

---

## Alerting

Each detection was converted into a **scheduled alert** with its own cron cadence, severity, and throttle window — tuned by how time-sensitive each threat is.

| Alert ID | Title | Schedule | Severity | Why This Severity |
|---|---|---|---|---|
| SEC-001 | Brute Force Detected | every 5 min | Medium | Common, but precedes worse outcomes |
| SEC-002 | Account Compromise | every 10 min | **Critical** | Active breach in progress |
| SEC-003 | Privilege Escalation | every 15 min | High | Precursor to data theft / wire fraud |
| SEC-004 | Data Exfiltration | hourly | High | Client portfolios at risk |
| SEC-005 | New User Account | every 15 min | Low | Could be legitimate; needs review |
| SEC-006 | Ransomware Behaviour | every 2 min | **Critical** | Active encryption — minutes matter |
| SEC-007 | Malicious IP Contact | every 10 min | High | Confirmed C2 communication |

Each alert was configured with **Trigger when Number of Results > 0**, throttled to avoid alert fatigue (suppressing repeat fires for the same match), and tagged for the SOC team's triggered-alerts queue, with optional email notification to a SOC distribution list.

---

## Dashboard

A single **TWP Global SOC Dashboard** consolidates detection output into one operational view:

**KPI row (Single Value panels):**
- Failed Logins (color-thresholded: green → yellow → red at 0/10/30)
- Total Security Events
- Unique Attacking IPs (`dc(src_ip)`)
- Privilege Escalations
- New Accounts Created

**Visualizations:**
- **Security Event Timeline** — `timechart span=30m` line chart, one series per event type (Failed Login, Successful Login, Account Created, Privilege Escalation, File Modification)
- **Top Attackers** — bar chart of top 10 source IPs by failed-login count
- **Geographic Office Breakdown** — pie/column chart using `like(host, "TWP-LON-%")` pattern matching to bucket events by office
- **Outbound Traffic Trend** — area chart of hourly outbound megabytes
- **Per-alert statistics tables** — one panel per SEC-00X query, labeled by alert ID
- **Global time range picker** — tokenized (`$time_range.earliest$` / `$time_range.latest$`) so every panel responds to one control

---

## Verification & QA

A structured checklist was used to confirm the dashboard and alerts matched expected ground truth before considering the build "done" — the same discipline a real SOC build would require before going live:

| Check | Expected Result |
|---|---|
| KPI: Failed Logins | ≥ 80 events |
| KPI: Unique Attacking IPs | ≥ 6 IPs |
| KPI: Privilege Escalations | 4 |
| KPI: New Accounts | 3 |
| Timeline | File-modification spike at 14:20 |
| SEC-002 | 2 rows — risk scores 80 and 60 |
| SEC-004 | 5 rows — largest 142 MB to known C2 |
| SEC-006 | 3 rows across London and Toronto |

Common beginner pitfalls were also documented and corrected along the way — most notably the **time range trap** (data is fixed-dated; without "All Time," every search returns nothing) and the distinction between `where` (post-aggregation filter) and raw `field=value` filtering.

---

## Skills Demonstrated

- **SPL query writing** — filtering, aggregation, conditional logic, time bucketing, joins, lookups
- **Detection engineering** — translating a threat hypothesis into a tuned, thresholded query
- **Splunk administration** — index design, lookup management, RBAC, custom roles
- **Alert design** — severity tiering, scheduling cadence, throttling to control noise
- **Dashboard/UX design** — KPI selection, color thresholds, time-range tokenization
- **Incident response thinking** — reconstructing an attack chronologically from raw telemetry
- **Regulatory awareness** — FCA/SEC/OSC considerations baked into role design and account lifecycle practices

---

## Incident Walkthrough

The dataset models a single, continuous attack on **26 April 2026** that the detections above were built to catch:

| Time (BST) | Event |
|---|---|
| 06:00–06:08 | Brute force succeeds against Oliver Harrington (Trader) |
| 06:30–06:36 | Brute force succeeds against James Fitzgerald (IT Security Lead) from an APT-attributed IP |
| 07:00–10:00 | Failed brute-force attempts against CFO, Compliance, NYC banker, Toronto PM |
| 09:30 | Anthony Romano (Helpdesk) promoted to Domain Admin via James's compromised account |
| 10:00–11:30 | Backdoor accounts created: `temp_auditor`, `backdoor_admin`, `svc_test` |
| 11:00–13:00 | 142 MB+ exfiltrated to an APT C2 server |
| 14:20–14:25 | Ransomware detonates across London and Toronto file servers |

---

*This project uses entirely synthetic, fictional data for training/portfolio purposes. No real individuals, companies, or systems are represented.*
