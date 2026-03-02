# SOC Automation Platform — AWS Real-Time Threat Detection

> A real-time security operations platform built on AWS that detects, correlates, and responds to cloud threats — live, in a browser, with one click to block an attacker.

---

## 🎬 See It In Action

### Full Attack Simulation (60 seconds)
> *A 7-stage MITRE ATT&CK kill chain fires against a live AWS account. Watch the platform detect, correlate, and surface the threat in real time.*

> 📹 **How to record this:**
> 1. Open ScreenToGif, set region to the **full Cyber dashboard browser window**
> 2. Start recording
> 3. Run `python scripts/simulate_attack.py --severity Critical` in terminal
> 4. Let it play until the C2 alert fires (~20 seconds)
> 5. Stop, export as GIF — **this is your hero asset, put it first**

![Full Attack Demo](assets/attack-demo.gif)

---

### Threat Correlation Card Firing
> *Same attacker IP appears across CloudTrail, VPC Flow Logs, GuardDuty, and Route 53 — the engine correlates all 4 sources into one incident card with MITRE tags and a one-click Block IP button.*

> 📹 **How to record this:**
> 1. Record region = **just the correlation alert card** (right panel of Cyber dashboard)
> 2. Run the attack script — wait for the CRITICAL card to pop in
> 3. Slowly scroll through the card so the MITRE tags (T1078, T1552.004, T1562.008, T1071) are all visible
> 4. Hover over the **🚫 Block IP in WAF** button at the end — don't click it yet (save that for the next GIF)

![Correlation Engine](assets/correlation-card.gif)

---

### Anomaly Detection — Log Silence Alert
> *The attacker ran `DeleteTrail` to cover their tracks. Within seconds, the anomaly engine detected the 100% drop in Route 53 ingestion and fired a CRITICAL alert.*

> 📹 **How to record this:**
> 1. Record region = **the Pipeline Anomaly Alerts panel** (bottom-left or wherever it sits)
> 2. Run the attack script
> 3. Capture the moment the **CRITICAL: Route 53 ingestion dropped 100%** alert appears
> 4. Let it linger 3 seconds so the viewer can read it — that's the whole GIF, short and punchy

![Anomaly Detection](assets/anomaly-alert.gif)

---

### Blocking the IP in AWS WAF (Live)
> *One click. The platform calls the AWS WAFv2 API and adds the attacker IP to a real IP set attached to a real Web ACL — no console, no CLI.*

> 📹 **How to record this (the most impressive asset):**
> 1. **Before recording:** open the AWS Console → WAF → IP sets → `soc-block-list-prod` in a second window
> 2. Set up a **split screen**: platform on the left, AWS console IP set page on the right
> 3. Start recording the full split screen
> 4. Click **🚫 Block IP in WAF** on the platform
> 5. Immediately switch to AWS console and **refresh the IP set page** — the IP `185.220.101.47` appears
> 6. This is money. No CLI. No console navigation. One click → real AWS resource updated.

![WAF Block](assets/waf-block.gif)

---

## What This Is

A **security operations center (SOC) in a box** — one browser tab, full visibility into an AWS account, real automated response.

Most security tools tell you what happened yesterday. This tells you what is happening right now.

---

## What It Does

### Three Detection Engines Running in Parallel

| Engine | What It Watches | When It Fires |
|---|---|---|
| **Threat Correlation** | Same IP across multiple log sources | IP seen in 4+ sources (CloudTrail, VPC Flow Logs, GuardDuty, Route 53) within the time window |
| **Anomaly Detection** | Event volume per source over time | Ingestion rate spikes or drops > 60% from baseline |
| **Compliance Scanner** | CloudFormation stack metadata | Stack deployed without required security tags |

Every event that enters the platform passes through all three engines simultaneously.

---

### The 7-Stage Attack It Can Detect

| Stage | MITRE Technique | What Happened |
|---|---|---|
| Reconnaissance | — | `GetCallerIdentity` — attacker probing the account |
| Discovery | — | `ListUsers` — mapping what's in the account |
| Initial Access | **T1078** — Valid Accounts | `AssumeRole` — escalating with stolen credentials |
| Credential Access | **T1552.004** — Cloud Secrets | `GetSecretValue` — pulling secrets from Secrets Manager |
| Discovery II | — | `ListBuckets` — finding data to steal |
| Exfiltration | — | `GetObject` — data leaving the account |
| Defense Evasion | **T1562.008** — Disable Cloud Logs | `DeleteTrail` — attacker trying to hide |
| Command & Control | **T1071** — App Layer Protocol | C2 beacon — attacker's machine calling home |

---

### Live AWS Integration

This is not a mock. These are real AWS API calls:

| Action | AWS Service | What Actually Happens |
|---|---|---|
| Block IP | **WAFv2** | Adds IP to a real IP set on a live Web ACL |
| Deploy Stack | **CloudFormation** | Creates real AWS resources in your account |
| GuardDuty Alerts | **GuardDuty** | Pulls live findings from an active detector |
| Compliance Scan | **CloudFormation** | Reads real stack tags via `describeStacks` |
| Delete Stack | **CloudFormation** | Tears down stacks in dependency order with live status |

---

## Architecture Overview

```
Attack Events (AWS APIs)
        │
        ▼
  Ingest Pipeline ──► OCSF Normalization
        │
        ├──► Threat Correlation Engine ──► Alert: "Multi-Source Activity from X.X.X.X"  
        │
        ├──► Anomaly Detection Engine  ──► Alert: "CloudTrail ingestion dropped 100%"
        │
        └──► Compliance Scanner        ──► Alert: "Stack missing RequiredTag: Owner"
                │
                ▼
      WebSocket Broadcast
                │
        ┌───────┴───────┐
        ▼               ▼
  Cyber Dashboard   DE Dashboard
  (Threat Feed)     (Infra Control)
```

---

## Two Dashboards

### Cyber Dashboard — Live Threat Feed

> 📸 **How to screenshot this:**
> Run the attack script, wait until ALL alerts have fired (correlation card + anomaly alerts both visible), then take a full-window screenshot. You want the dashboard **full** — event feed scrolled down showing all 8 events, correlation card visible, anomaly panel showing both Route 53 and CloudTrail alerts. This is the "after" state.

![Cyber Dashboard](assets/cyber-dashboard.png)

- Real-time event feed with source tagging (CloudTrail, GuardDuty, Route 53...)
- Correlation cards with full attack timelines and MITRE ATT&CK tags
- Anomaly alerts with baseline vs actual charts
- One-click IP blocking via WAF

---

### DE Dashboard — Infrastructure Control

> 📸 **How to screenshot this:**
> Make sure all 3 stacks are deployed (soc-guardduty, soc-waf, soc-demo-resources all showing CREATE_COMPLETE in green). Take a full-window screenshot. The "3 green stacks + Lambda functions listed" state is what you want — it shows real infrastructure, not empty panels.

![DE Dashboard](assets/de-dashboard.png)

- Deploy / delete AWS security stacks directly from the browser
- View deployed Lambda functions and CodePipeline status
- Trigger baseline activity or full attack simulations
- Stack-level compliance scan results

---

## AWS Stacks Deployed

Three CloudFormation stacks — all defined as IaC, all deployable from the DE dashboard:

| Stack | Key Resources |
|---|---|
| **soc-guardduty** | GuardDuty detector · S3 findings bucket · EventBridge rule (severity ≥ 7) |
| **soc-waf** | WAFv2 IP set `soc-block-list` · WAFv2 Web ACL |
| **soc-demo-resources** | S3 data bucket · Secrets Manager secret |

---

## Real vs Simulated

| Component | Real? |
|---|---|
| AWS API calls (AssumeRole, GetSecretValue, ListBuckets) | ✅ Real — appear in CloudTrail |
| GuardDuty detector | ✅ Real — live AWS resource |
| WAF IP blocking | ✅ Real — live WAFv2 API call |
| CloudFormation stacks | ✅ Real — deployed to AWS account |
| DeleteTrail (stage 6) | 🟡 Simulated — injected as event, no actual damage |
| C2 Traffic (stage 7) | 🟡 Simulated — injected as event, no real outbound connection |

---

## How This Relates to AWS Security Lake

AWS Security Lake is Amazon's managed service for centralizing security data at scale — it ingests logs from CloudTrail, VPC Flow Logs, Route 53, GuardDuty, and third-party tools, normalizes everything into OCSF format, and stores it in S3 for long-term querying with Athena.

This platform and Security Lake are **complementary layers**, not competitors:

| | This Platform | AWS Security Lake |
|---|---|---|
| **Purpose** | Real-time detection & response | Long-term storage & compliance |
| **Latency** | Seconds | 10–15 minutes |
| **Best for** | Active threat, live blocking | Historical investigation, audits |
| **Query style** | Live WebSocket feed | Athena SQL |
| **OCSF** | Normalizes on ingest | Native schema |
| **Cost** | Minimal (compute only) | Storage + query costs at scale |

### How they work together in a real environment

```
AWS Log Sources (CloudTrail, VPC Flow Logs, GuardDuty, Route 53)
        │
        ├──► This Platform          →  Detect NOW  →  Block attacker in WAF
        │                                              (seconds)
        │
        └──► AWS Security Lake      →  Store OCSF  →  Query history in Athena
                                                       (compliance, forensics)
```

The real-time layer catches and stops the attack. Security Lake stores the full record for the post-incident investigation, compliance report, or insurance claim.

> 📸 **How to capture this:**
> Go to **AWS Console → Security Lake → Data sources**. Take a screenshot showing the log sources listed (CloudTrail, VPC Flow Logs, Route 53, GuardDuty) with their OCSF mappings. This one screenshot says "I know what OCSF is and I know how Security Lake ingests data" — without needing to explain it. If Security Lake isn't enabled in your account, a screenshot of the **Security Lake architecture diagram from the AWS docs page** works just as well.

![Security Lake OCSF](assets/security-lake-ocsf.png)

---

## The One-Liner

> *Security Lake tells you what happened last week. This tells you what's happening right now.*
