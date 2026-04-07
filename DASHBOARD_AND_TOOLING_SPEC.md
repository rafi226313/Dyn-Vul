# Security Runner POC — Dashboard & Custom Tooling Specification

**Document Type:** Technical Specification & Implementation Reference  
**Version:** 1.1  
**Created:** March 13, 2026  
**Updated:** March 13, 2026 — Backend implementation completed  
**Scope:** Frontend Dashboard Design + Deployment-Ready Custom Docker Images  
**Target POC:** HIPAA Checker Application on g4dn Instance  
**Implementation Status:** ✅ Backend API, custom Docker images, and codebase changes are **implemented**. Frontend dashboard UI is pending (Part 1 serves as the blueprint for the frontend team).

---

## Table of Contents

- [Part 1 — Frontend Dashboard Specification](#part-1--frontend-dashboard-specification)
  - [1.1 Purpose & Guiding Principles](#11-purpose--guiding-principles)
  - [1.2 Dashboard Architecture Overview](#12-dashboard-architecture-overview)
  - [1.3 Page-by-Page Layout Specification](#13-page-by-page-layout-specification)
    - [1.3.1 Main Dashboard — Command Center](#131-main-dashboard--command-center)
    - [1.3.2 Scan Configuration Panel](#132-scan-configuration-panel)
    - [1.3.3 Live Scan Monitor](#133-live-scan-monitor)
    - [1.3.4 Reports Viewer](#134-reports-viewer)
    - [1.3.5 System Health & Logs](#135-system-health--logs)
  - [1.4 Configuration Surface](#14-configuration-surface)
  - [1.5 API Contract Between Dashboard and Security Runner](#15-api-contract-between-dashboard-and-security-runner)
  - [1.6 Visual Design Guidelines](#16-visual-design-guidelines)
- [Part 2 — Custom DAST Scanner Docker Image Specification](#part-2--custom-dast-scanner-docker-image-specification)
  - [2.1 Design Philosophy](#21-design-philosophy)
  - [2.2 Image Architecture](#22-image-architecture)
  - [2.3 DAST Scanner Automation Framework Plan File](#23-dast-scanner-automation-framework-plan-file)
  - [2.4 Dockerfile Specification](#24-dockerfile-specification)
  - [2.5 Entrypoint & Control Script](#25-entrypoint--control-script)
  - [2.6 Environment Variable Interface](#26-environment-variable-interface)
  - [2.7 Volume Mounts & Ports](#27-volume-mounts--ports)
  - [2.8 Updated Docker Compose Service Definition](#28-updated-docker-compose-service-definition)
  - [2.9 Orchestrator Integration — How the Dashboard Controls the DAST Scanner](#29-orchestrator-integration--how-the-dashboard-controls-the-dast-scanner)
  - [2.10 Build, Tag, and Distribution](#210-build-tag-and-distribution)
  - [2.11 Changes Required to Existing Codebase](#211-changes-required-to-existing-codebase)
- [Part 3 — Custom Runtime Monitor & Cloud Auditor Image Notes (Companion Images)](#part-3--custom-runtime-monitor--cloud-auditor-image-notes-companion-images)
  - [3.1 Custom Runtime Monitor Image](#31-custom-runtime-monitor-image)
  - [3.2 Custom Cloud Auditor Image](#32-custom-cloud-auditor-image)
  - [3.3 Combined Distribution Matrix](#33-combined-distribution-matrix)
- [Appendix A — Full File Tree After Changes](#appendix-a--full-file-tree-after-changes)
- [Appendix B — Glossary](#appendix-b--glossary)

---

# Part 1 — Frontend Dashboard Specification

## 1.1 Purpose & Guiding Principles

The dashboard is the **single control surface** through which a security team member operates the entire Security Runner POC. It replaces the current CLI-driven workflow (`run_security_poc.sh`) with a visual interface.

| Principle | Description |
|-----------|-------------|
| **Single-user, security-team interface** | No multi-role auth. One user operates the dashboard at a time. |
| **On-demand scanning only** | No scheduler. The operator clicks "Run" when ready. |
| **Individual + full-pipeline control** | Each tool (DAST Scanner, Runtime Monitor, Cloud Auditor) can be triggered independently, or as the full orchestrated pipeline. |
| **Real-time progress + final reports** | Live progress bars, streaming Runtime Monitor events, and downloadable final reports. |
| **Single target** | The HIPAA Checker stack (Next.js frontend + Rails backend). No multi-app support needed. |

## 1.2 Dashboard Architecture Overview

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Browser (Dashboard UI)                          │
│                                                                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │ Command  │  │  Config  │  │   Live   │  │ Reports  │  │ System  │ │
│  │ Center   │  │  Panel   │  │  Monitor │  │  Viewer  │  │ Health  │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬────┘ │
│       │              │             │              │             │       │
│       └──────────────┴─────────────┴──────────────┴─────────────┘       │
│                                    │                                    │
│                             REST / WebSocket                            │
└────────────────────────────────────┼───────────────────────────────────┘
                                     │
┌────────────────────────────────────┼───────────────────────────────────┐
│                     Security Runner Backend (API)                       │
│                                                                        │
│   Exposes REST endpoints for:                                          │
│     • Trigger scans (full pipeline or individual tool)                 │
│     • Query scan status / progress                                     │
│     • Stream Runtime Monitor events (WebSocket)                                  │
│     • Fetch / download reports                                         │
│     • Health-check all containers                                      │
│                                                                        │
│   Internally communicates with:                                        │
│     ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│     │ DAST Scanner│  │ Runtime Mon.│  │Cloud Auditor    │                 │
│     │   Image    │  │   Image     │  │Custom Image │                 │
│     │  (API)     │  │ (events/log)│  │ (exec)      │                 │
│     └─────────────┘  └─────────────┘  └─────────────┘                 │
└────────────────────────────────────────────────────────────────────────┘
```

> **Key insight:** The dashboard is a thin UI layer. All scan logic lives in the Security Runner backend (the `runner.py` orchestrator, upgraded to expose an HTTP API). The dashboard never talks directly to DAST Scanner, Runtime Monitor, or Cloud Auditor — it always goes through the Security Runner API.

## 1.3 Page-by-Page Layout Specification

### 1.3.1 Main Dashboard — Command Center

This is the landing page. It provides an at-a-glance view of the entire system and the primary scan controls.

```
┌──────────────────────────────────────────────────────────────────────┐
│  HIPAA Security Scanner                              [System Health] │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─ Target Status ──────────────────────────────────────────────┐   │
│  │                                                               │   │
│  │  Next.js Frontend    ● Healthy    http://nextjs-frontend:3000│   │
│  │  Rails Backend       ● Healthy    http://rails-backend:8080  │   │
│  │  PostgreSQL           ● Healthy    poc-postgres:5432          │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ Scanner Status ─────────────────────────────────────────────┐   │
│  │                                                               │   │
│  │  DAST Scanner        ● Ready      v2.x.x                    │   │
│  │  Runtime Monitor    ● Monitoring  (32 rules loaded)         │   │
│  │  Cloud Auditor       ● Idle        (AWS creds: configured)   │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ Quick Actions ──────────────────────────────────────────────┐   │
│  │                                                               │   │
│  │  [ ▶ Run Full Pipeline ]    (DAST Scanner + Runtime Monitor + Cloud Auditor)          │   │
│  │                                                               │   │
│  │  [ ▶ DAST Only ]  [ ▶ Runtime Only ]  [ ▶ Cloud Audit Only ]       │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ Last Scan Summary ──────────────────────────────────────────┐   │
│  │                                                               │   │
│  │  Last run: 2026-03-12 14:23 UTC   Duration: 1h 47m          │   │
│  │  Status: ✅ SUCCESS                                           │   │
│  │                                                               │   │
│  │  DAST Alerts: 1554 (2 High, 236 Medium, 1316 Low)                  │   │
│  │  Runtime Events: 329 (10 Critical, 318 Warning, 1 Notice)          │   │
│  │  Cloud Auditor: 41 Passed / 29 Failed                             │   │
│  │                                                               │   │
│  │  [ View Full Report → ]                                      │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**Behaviors:**
- "Target Status" and "Scanner Status" sections auto-refresh every 15 seconds via polling.
- Status indicators use traffic-light colors: green (healthy/ready), yellow (degraded/starting), red (down/error).
- Clicking any "▶ Run" button navigates to the **Scan Configuration Panel** with the relevant tools pre-selected.
- "Last Scan Summary" loads from the most recent `aggregated_report.json`.
- If a scan is currently in progress, the "Quick Actions" buttons become disabled, replaced with a "Scan in Progress — View Monitor →" link.

---

### 1.3.2 Scan Configuration Panel

Reached when the user clicks any "▶ Run" button. This is where the operator reviews and adjusts settings before executing.

```
┌──────────────────────────────────────────────────────────────────────┐
│  Configure Scan                                        [ ← Back ]    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─ Tool Selection ─────────────────────────────────────────────┐   │
│  │                                                               │   │
│  │  [✓] DAST Scanner                                       │   │
│  │  [✓] Runtime Monitor                                   │   │
│  │  [✓] Cloud Auditor AWS Audit                                       │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ DAST Scanner Settings (visible when DAST Scanner is checked) ────────────────┐   │
│  │                                                               │   │
│  │  Scan Type:      [ Full ▾ ]  (Full / Baseline / API-only)   │   │
│  │                                                               │   │
│  │  AJAX Spider:    [✓] Enabled                                 │   │
│  │    Max Duration:   [ 60 ] minutes                            │   │
│  │    Max Depth:      [ 10 ]                                    │   │
│  │    Browsers:       [ 3  ]                                    │   │
│  │                                                               │   │
│  │  Active Scan:    [✓] Enabled                                 │   │
│  │    Max Duration:   [ 90 ] minutes                            │   │
│  │    Threads/Host:   [ 5  ]                                    │   │
│  │    Strength:       [ HIGH ▾ ]                                │   │
│  │    Threshold:      [ MEDIUM ▾ ]                              │   │
│  │                                                               │   │
│  │  Traditional Spider: [✓] Enabled                             │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ Cloud Auditor Settings (visible when Cloud Auditor is checked) ────────┐   │
│  │                                                               │   │
│  │  AWS Region:     [ us-east-1 ▾ ]                             │   │
│  │  Compliance:     [✓] HIPAA  [✓] CIS Level 1  [ ] CIS 1.5   │   │
│  │  Categories:     [✓] IAM [✓] S3 [✓] EC2 [✓] RDS [✓] KMS   │   │
│  │                  [✓] CloudTrail [✓] VPC [✓] Secrets Mgr     │   │
│  │  Severity:       [ Low ▾ ]  (minimum severity to report)     │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ Runtime Monitor Settings (visible when Runtime Monitor is checked) ────────────┐   │
│  │                                                               │   │
│  │  Monitored Containers: nextjs-frontend, rails-backend        │   │
│  │  Minimum Priority:     [ Notice ▾ ]                          │   │
│  │  Trigger Test Vuln:    [✓] Auto-trigger after scan           │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ Global Settings ────────────────────────────────────────────┐   │
│  │                                                               │   │
│  │  Overall Timeout:  [ 150 ] minutes                           │   │
│  │  Max Retries:      [ 3 ]                                     │   │
│  │  Log Level:        [ INFO ▾ ]                                │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│           [ ▶ Start Scan ]            [ Reset to Defaults ]         │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**Behaviors:**
- Checking/unchecking tools in the "Tool Selection" section shows/hides the corresponding settings panel.
- All default values are loaded from the existing `scan-config.yaml`.
- "Reset to Defaults" reverts all fields to the values in `scan-config.yaml`.
- "Start Scan" sends the configuration payload to the Security Runner API and navigates to the **Live Scan Monitor**.
- Settings that are adjusted here **do not permanently modify** `scan-config.yaml` — they are passed as runtime overrides. The YAML remains the source of truth for defaults.

---

### 1.3.3 Live Scan Monitor

The real-time view of a scan in progress.

```
┌──────────────────────────────────────────────────────────────────────┐
│  Scan in Progress                      Started: 14:23 UTC   [Stop] │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Overall Pipeline                                                    │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━░░░░░░░░░░░░░░  62%   Phase: DAST Scan │
│                                                                      │
│  ┌─ Phase Progress ─────────────────────────────────────────────┐   │
│  │                                                               │   │
│  │  ✅ Phase 1: Health Check          Passed     (0:02)         │   │
│  │  🔄 Phase 2: DAST Scan            Running    (0:47 / ~2:00) │   │
│  │  ⏳ Phase 3: Cloud Audit           Pending                   │   │
│  │  ⏳ Phase 4: Runtime Analysis      Pending                   │   │
│  │  ⏳ Phase 5: Report Aggregation    Pending                   │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ DAST Scanner Detail ─────────────────────────────────────────────────┐   │
│  │                                                               │   │
│  │  AJAX Spider:         ━━━━━━━━━━━━━━━━━━━━  100%  (217 URLs) │   │
│  │  Traditional Spider:  ━━━━━━━━━━━━━━━━━━━━  100%  (54 URLs)  │   │
│  │  Active Scan:         ━━━━━━━━━━━━━░░░░░░░   63%             │   │
│  │                                                               │   │
│  │  Alerts so far:  8 (2 High, 3 Medium, 3 Low)                │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ Runtime Monitor Live Events (streaming) ──────────────────────────────┐   │
│  │                                                               │   │
│  │  14:24:12  WARNING  Shell spawned in rails-backend            │   │
│  │  14:24:12  CRITICAL Test Vulnerability Triggered (touch ...)  │   │
│  │  14:31:05  NOTICE   Outbound connection from nextjs-frontend  │   │
│  │                                                               │   │
│  │  Events: 3 total   Critical: 1   Warning: 1   Notice: 1     │   │
│  │                                                 [ Pause Feed ] │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**Behaviors:**
- **Overall progress bar** is computed by the Security Runner: each phase contributes a weighted percentage (health=2%, DAST=60%, Cloud Auditor=20%, Runtime Monitor=8%, Aggregation=10%).
- **DAST Scanner Detail** polls the DAST Scanner API through the Security Runner every 5 seconds for spider/scan status.
- **Runtime Monitor Live Events** uses a WebSocket connection. The Security Runner tails the Runtime Monitor JSON log file and pushes new events to the dashboard.
- The **[Stop]** button sends a cancel request to the Security Runner, which gracefully stops the current phase, collects partial results, and generates a partial report.
- Once the scan completes, this page transitions to a "Scan Complete" state with a prominent "View Reports →" button.

---

### 1.3.4 Reports Viewer

Displays the final scan results. This page is accessible after a scan completes, or by clicking "View Full Report" from the Command Center.

```
┌──────────────────────────────────────────────────────────────────────┐
│  Scan Report — 2026-03-12 14:23 UTC                                  │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─ Executive Summary ──────────────────────────────────────────┐   │
│  │                                                               │   │
│  │  Overall: ✅ SUCCESS    Duration: 1h 15m                      │   │
│  │                                                               │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │   │
│  │  │   DAST   │  │ Runtime  │  │  Cloud   │                   │   │
│  │  │ Scanner  │  │ Monitor  │  │ Auditor  │                   │   │
│  │  │1554 alert│  │329 event │  │ 29 fails │                   │   │
│  │  │  2 High  │  │ 10 Crit  │  │ 41 pass  │                   │   │
│  │  └──────────┘  └──────────┘  └──────────┘                   │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  [ DAST Scanner ]  [ Runtime Monitor ]  [ Cloud Auditor ]   ← Tabs │
│                                                                      │
│  ┌─ DAST Scanner Report (active tab) ───────────────────────────────────┐   │
│  │                                                               │   │
│  │  Severity Breakdown:                                          │   │
│  │   ██████  High     3 alerts                                  │   │
│  │   ██████████  Medium   5 alerts                              │   │
│  │   ████████  Low      4 alerts                                │   │
│  │                                                               │   │
│  │  ┌──────────────────────────────────────────────────────┐    │   │
│  │  │ Severity │ Alert Name           │ URL        │ Count │    │   │
│  │  ├──────────┼──────────────────────┼────────────┼───────┤    │   │
│  │  │ 🔴 High  │ Command Injection    │ /api/v1/.. │  1    │    │   │
│  │  │ 🔴 High  │ SQL Injection        │ /api/v1/.. │  1    │    │   │
│  │  │ 🔴 High  │ XSS (Reflected)      │ /search    │  1    │    │   │
│  │  │ 🟡 Med   │ CSRF Missing         │ /settings  │  2    │    │   │
│  │  │ 🟡 Med   │ Cookie No HttpOnly   │ (all)      │  1    │    │   │
│  │  │ ...      │ ...                  │ ...        │ ...   │    │   │
│  │  └──────────────────────────────────────────────────────┘    │   │
│  │                                                               │   │
│  │  Clicking a row expands to show:                             │   │
│  │   • Description, Solution, CWE/WASC reference                │   │
│  │   • Request/Response evidence                                │   │
│  │   • HIPAA mapping (if applicable)                            │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ Runtime Monitor Report (tab) ────────────────────────────────┐   │
│  │                                                               │   │
│  │  Priority Breakdown:                                          │   │
│  │   ██████████████████████████████████████  Warning  318 events │   │
│  │   ██████  Critical  10 events                                │   │
│  │   ██  Notice     1 event                                     │   │
│  │                                                               │   │
│  │  Total Events: 329           Monitoring Period: 22 days       │   │
│  │                                                               │   │
│  │  ┌─ Rules Triggered ────────────────────────────────────┐    │   │
│  │  │ Priority  │ Rule                        │ Count │MITRE│    │   │
│  │  ├───────────┼─────────────────────────────┼───────┼─────┤    │   │
│  │  │ ⚠ Warning │ Read sensitive file untrust. │  314  │T1555│    │   │
│  │  │ 🔴 Crit   │ Drop & execute new binary   │   10  │TA003│    │   │
│  │  │ ⚠ Warning │ Find AWS Credentials        │    4  │T1552│    │   │
│  │  │ ℹ Notice  │ Terminal shell in container  │    1  │T1059│    │   │
│  │  └──────────────────────────────────────────────────────┘    │   │
│  │                                                               │   │
│  │  ┌─ Event Timeline (expandable rows) ───────────────────┐    │   │
│  │  │ Time           │ Priority │ Rule           │Container│    │   │
│  │  ├────────────────┼──────────┼────────────────┼─────────┤    │   │
│  │  │ 03-15 22:45:00 │ 🔴 CRIT  │ Drop & exec    │dast-scan│    │   │
│  │  │ 03-15 22:45:00 │ 🔴 CRIT  │ Drop & exec    │dast-scan│    │   │
│  │  │ 03-15 22:59:24 │ 🔴 CRIT  │ Drop & exec    │dast-scan│    │   │
│  │  │ 03-16 03:11:53 │ ⚠ WARN   │ Find AWS Creds │host     │    │   │
│  │  │ 03-16 10:04:52 │ ⚠ WARN   │ Read sensitive │host     │    │   │
│  │  │ 03-16 10:04:52 │ ⚠ WARN   │ Read sensitive │host     │    │   │
│  │  │ ...            │ ...      │ ...            │ ...     │    │   │
│  │  └──────────────────────────────────────────────────────┘    │   │
│  │                                                               │   │
│  │  Clicking a row expands to show:                             │   │
│  │   • Process: geckodriver --port=26225 --websocket-port 0     │   │
│  │   • Exe Path: /home/scanner/.scanner/webdriver/.../geckodriver│   │
│  │   • Parent process, User, UID, Container CWD                 │   │
│  │   • MITRE ATT&CK tags (TA0003, T1555, T1552, T1059)         │   │
│  │   • Raw event JSON payload                                   │   │
│  │   • Suggested remediation steps                               │   │
│  │   • Heatmap: event frequency over the monitoring period      │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ Cloud Auditor Report (tab) ───────────────────────────────────┐   │
│  │                                                               │   │
│  │  HIPAA Compliance Score:  ████████████░░░░░░░░  50%           │   │
│  │                                                               │   │
│  │  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐             │   │
│  │  │  41    │  │  29    │  │  12    │  │  82    │             │   │
│  │  │ Passed │  │ Failed │  │ Manual │  │ Total  │             │   │
│  │  └────────┘  └────────┘  └────────┘  └────────┘             │   │
│  │                                                               │   │
│  │  Severity Breakdown (Failed):                                 │   │
│  │   ██████████  Critical   5 checks                            │   │
│  │   ██████████████████████  High       11 checks               │   │
│  │   ██████████████████████████  Medium     13 checks            │   │
│  │                                                               │   │
│  │  ┌─ HIPAA Compliance by Section ────────────────────────┐    │   │
│  │  │ Section                    │ Pass │ Fail │ Score      │    │   │
│  │  ├────────────────────────────┼──────┼──────┼────────────┤    │   │
│  │  │ §164.312(a) Access Control │   5  │   4  │ ████░  56% │    │   │
│  │  │ §164.312(b) Audit Controls │   6  │   3  │ █████░ 67% │    │   │
│  │  │ §164.312(c) Integrity      │   6  │   4  │ █████░ 60% │    │   │
│  │  │ §164.312(d) Authentication │   5  │   4  │ ████░  56% │    │   │
│  │  │ §164.312(e) Transmission   │   5  │   4  │ ████░  56% │    │   │
│  │  │ §164.308(a) Security Mgmt  │   5  │   4  │ ████░  56% │    │   │
│  │  │ §164.308(7) Contingency    │   4  │   3  │ ████░  57% │    │   │
│  │  │ §164.308(1) Risk Analysis  │   5  │   3  │ █████░ 63% │    │   │
│  │  └──────────────────────────────────────────────────────┘    │   │
│  │                                                               │   │
│  │  ┌─ Failed Checks (sorted by severity) ─────────────────┐    │   │
│  │  │ Severity │ Service │ Check              │ Resource    │    │   │
│  │  ├──────────┼─────────┼────────────────────┼─────────────┤    │   │
│  │  │ 🔴 CRIT  │ IAM     │ Root account MFA   │Root Account│    │   │
│  │  │ 🔴 CRIT  │ IAM     │ Unused credentials │ arn:..user  │    │   │
│  │  │ 🔴 CRIT  │ S3      │ Public access      │ hipaa-logs  │    │   │
│  │  │ 🔴 CRIT  │ Cloud.. │ Trail not encryp.  │ hipaa-trail │    │   │
│  │  │ 🔴 CRIT  │ RDS     │ Instance not encr. │ hipaa-db    │    │   │
│  │  │ 🟠 HIGH  │ IAM     │ Password policy    │ Account     │    │   │
│  │  │ 🟠 HIGH  │ IAM     │ Access keys rotate │ arn:..user  │    │   │
│  │  │ 🟠 HIGH  │ S3      │ Bucket encryption  │ hipaa-data  │    │   │
│  │  │ 🟠 HIGH  │ S3      │ Versioning off     │ hipaa-logs  │    │   │
│  │  │ 🟠 HIGH  │ EC2     │ Security group SSH │ sg-0a1b2c3d │    │   │
│  │  │ 🟠 HIGH  │ RDS     │ Public access on   │ hipaa-db    │    │   │
│  │  │ 🟡 MED   │ EC2     │ EBS not encrypted  │ vol-abc123  │    │   │
│  │  │ 🟡 MED   │ EC2     │ IMDSv2 not enforced│ i-0abc123   │    │   │
│  │  │ 🟡 MED   │ VPC     │ Flow logs disabled │ vpc-main    │    │   │
│  │  │ ...      │ ...     │ ...                │ ...         │    │   │
│  │  └──────────────────────────────────────────────────────┘    │   │
│  │                                                               │   │
│  │  Clicking a row expands to show:                             │   │
│  │   • Full check description and risk assessment               │   │
│  │   • Affected resource ARN                                     │   │
│  │   • HIPAA section mapping (e.g., §164.312(a)(2)(iv))         │   │
│  │   • Step-by-step remediation guidance                         │   │
│  │                                                               │   │
│  │  ┌─ Passed Checks (collapsed by default) ───────────────┐    │   │
│  │  │  ▶ Show 41 passed checks                              │    │   │
│  │  │  (collapsed — click to expand list of passed controls  │    │   │
│  │  │   with evidence links and resource ARNs)               │    │   │
│  │  └──────────────────────────────────────────────────────┘    │   │
│  │                                                               │   │
│  │  Export: [ CSV ] [ JSON ] [ HTML ]                           │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  [ Download JSON ]  [ Download HTML ]  [ Download Full Bundle ]     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**Behaviors:**
- Three tabs: DAST Scanner, Runtime Monitor, Cloud Auditor. Each tab shows the tool-specific report in a structured, browsable format.
- **DAST Scanner tab:** Table of alerts, sortable by severity. Clicking a row expands inline to show full alert details. The data is sourced from `dast_report.json`.
- **Runtime Monitor tab:** Timeline view of events, color-coded by priority. Each event expands to show event JSON and suggested remediation.
- **Cloud Auditor tab:** Two sub-sections: "Failed Checks" (sorted by severity) and "Passed Checks" (collapsed by default). Each check shows its HIPAA safeguard mapping.
- **Download buttons:** JSON/HTML per tool, or a "Full Bundle" ZIP containing all reports.
- A "Scan History" sidebar (or dropdown) lists previous scan runs by timestamp, so the user can compare across runs.

---

### 1.3.5 System Health & Logs

Accessible from the header's "System Health" link. Provides infrastructure-level visibility.

```
┌──────────────────────────────────────────────────────────────────────┐
│  System Health                                                       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─ Container Status ───────────────────────────────────────────┐   │
│  │                                                               │   │
│  │  Container          │ Status  │ Memory  │ CPU  │ Uptime      │   │
│  │  ──────────────────────────────────────────────────────────   │   │
│  │  nextjs-frontend    │ ● Up    │ 1.2 GB  │ 3%   │ 2h 14m     │   │
│  │  rails-backend      │ ● Up    │ 3.1 GB  │ 8%   │ 2h 14m     │   │
│  │  poc-postgres       │ ● Up    │ 0.4 GB  │ 1%   │ 2h 14m     │   │
│  │  dast-scanner       │ ● Up    │ 6.8 GB  │ 22%  │ 2h 13m     │   │
│  │  runtime-monitor   │ ● Up    │ 0.3 GB  │ 5%   │ 2h 13m     │   │
│  │  cloud-auditor     │ ● Up    │ 0.1 GB  │ 0%   │ 2h 13m     │   │
│  │  security-runner    │ ● Up    │ 0.5 GB  │ 2%   │ 2h 14m     │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ Host System ────────────────────────────────────────────────┐   │
│  │                                                               │   │
│  │  Instance:   g4dn.8xlarge        RAM: 87 / 128 GB free       │   │
│  │  Disk:       142 GB / 500 GB     Docker version: 24.0.7      │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ Container Logs (select container) ──────────────────────────┐   │
│  │                                                               │   │
│  │  [ security-runner ▾ ]   Lines: [ 100 ▾ ]   [ Refresh ]     │   │
│  │                                                               │   │
│  │  [14:23:01] INFO  Phase 1: Health Checks                     │   │
│  │  [14:23:03] INFO  ✓ All services healthy                     │   │
│  │  ...                                                          │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ Actions ────────────────────────────────────────────────────┐   │
│  │                                                               │   │
│  │  [ Restart Container ▾ ]   [ Pull Latest Images ]            │   │
│  │  [ Run Cleanup ]           [ Export Diagnostics ]            │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**Behaviors:**
- Container status refreshes every 10 seconds. Data comes from `docker stats` via the Security Runner API.
- Log viewer streams the selected container's stdout/stderr via the Docker API.
- "Restart Container" dropdown lists all POC containers and sends a restart command through the Security Runner.
- "Run Cleanup" executes the same logic as `cleanup_poc.sh` but with a confirmation dialog.

---

## 1.4 Configuration Surface

Below is the complete list of every configuration parameter the dashboard should expose, mapped to where each value lives in the current codebase.

### DAST Scanner Configuration

| Dashboard Field | Type | Default | Source (`scan-config.yaml` path) |
|----------------|------|---------|----------------------------------|
| Scan Type | Dropdown: `full`, `baseline`, `api` | `full` | env `DAST_SCAN_TYPE` |
| AJAX Spider Enabled | Toggle | `true` | `dast.ajax_spider.enabled` |
| AJAX Spider Max Duration (min) | Number | `60` | `dast.ajax_spider.max_duration_minutes` |
| AJAX Spider Max Depth | Number | `10` | `dast.ajax_spider.max_crawl_depth` |
| AJAX Spider Browser Count | Number | `3` | `dast.ajax_spider.number_of_browsers` |
| Traditional Spider Enabled | Toggle | `true` | `dast.spider.enabled` |
| Active Scan Enabled | Toggle | `true` | `dast.active_scan.enabled` |
| Active Scan Max Duration (min) | Number | `90` | `dast.active_scan.max_scan_duration_minutes` |
| Active Scan Threads/Host | Number | `5` | `dast.active_scan.threads_per_host` |
| Default Attack Strength | Dropdown: `LOW`, `MEDIUM`, `HIGH`, `INSANE` | `HIGH` | `dast.active_scan` (via scan-policy) |
| Default Alert Threshold | Dropdown: `OFF`, `LOW`, `MEDIUM`, `HIGH` | `MEDIUM` | `dast.active_scan` (via scan-policy) |

### Runtime Monitor Configuration

| Dashboard Field | Type | Default | Source |
|----------------|------|---------|--------|
| Monitored Containers | Read-only list | `nextjs-frontend, rails-backend` | `runtime_monitor.monitored_containers` |
| Minimum Alert Priority | Dropdown: `emergency`→`debug` | `notice` | `runtime-monitor-config/monitor.yaml` → `priority` |
| Auto-trigger Test Vuln | Toggle | `true` | `runtime_monitor.test_detection` |

### Cloud Auditor Configuration

| Dashboard Field | Type | Default | Source |
|----------------|------|---------|--------|
| AWS Region | Dropdown | `us-east-1` | env `AWS_REGION` |
| Compliance Frameworks | Multi-select checkboxes | `hipaa`, `cis_1.4_aws` | `cloud_auditor.compliance` |
| Service Categories | Multi-select checkboxes | All listed | `cloud_auditor.categories` |
| Severity Threshold | Dropdown: `critical`→`informational` | `low` | `cloud_auditor.severity_threshold` |

### Global / Orchestration

| Dashboard Field | Type | Default | Source |
|----------------|------|---------|--------|
| Overall Timeout (min) | Number | `150` | `success_criteria.max_execution_time_minutes` |
| Max Retries | Number | `3` | `orchestration.retries.max_attempts` |
| Retry Backoff (sec) | Number | `30` | `orchestration.retries.backoff_seconds` |
| Log Level | Dropdown: `DEBUG`, `INFO`, `WARNING`, `ERROR` | `INFO` | env `LOG_LEVEL` |

---

## 1.5 API Contract Between Dashboard and Security Runner

The current `runner.py` is a batch script that runs to completion. To support the dashboard, it has been upgraded to a long-running HTTP service. The `security-runner/app/` package implements a FastAPI server that exposes the REST and WebSocket endpoints below. The batch `runner.py` is retained for backward-compatible CLI mode.

### REST Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/health` | Runner health + status of all containers |
| `GET` | `/api/targets/status` | Health-check all scan targets |
| `GET` | `/api/scanners/status` | Status of DAST Scanner, Runtime Monitor, Cloud Auditor containers |
| `POST` | `/api/scan/start` | Start a scan with config overrides in request body |
| `GET` | `/api/scan/status` | Current scan phase, progress %, per-tool status |
| `POST` | `/api/scan/stop` | Gracefully stop the running scan |
| `GET` | `/api/scan/history` | List of past scan runs with summary |
| `GET` | `/api/reports/:scan_id` | Get aggregated report for a specific scan |
| `GET` | `/api/reports/:scan_id/download` | Download report bundle (ZIP) |
| `GET` | `/api/reports/:scan_id/:tool` | Get tool-specific report (dast/runtime-monitor/cloud-auditor) |
| `GET` | `/api/containers` | Docker stats for all POC containers |
| `GET` | `/api/containers/:name/logs` | Tail logs for a specific container |
| `POST` | `/api/containers/:name/restart` | Restart a specific container |
| `GET` | `/api/config/defaults` | Return current `scan-config.yaml` as JSON |

### WebSocket Endpoints

| Path | Description |
|------|-------------|
| `ws://runner:8000/ws/scan/progress` | Real-time scan progress updates (pushed every 3s) |
| `ws://runner:8000/ws/runtime-monitor/events` | Live Runtime Monitor event stream |
| `ws://runner:8000/ws/runner/logs` | Live runner log stream |

### Example: `POST /api/scan/start` Request Body

```json
{
  "tools": ["dast", "runtime-monitor", "cloud-auditor"],
  "dast": {
    "scan_type": "full",
    "ajax_spider": {
      "enabled": true,
      "max_duration_minutes": 60,
      "max_crawl_depth": 10,
      "number_of_browsers": 3
    },
    "spider": { "enabled": true },
    "active_scan": {
      "enabled": true,
      "max_scan_duration_minutes": 90,
      "threads_per_host": 5,
      "strength": "HIGH",
      "threshold": "MEDIUM"
    }
  },
  "runtime_monitor": {
    "min_priority": "notice",
    "auto_trigger_test_vuln": true
  },
  "cloud_auditor": {
    "region": "us-east-1",
    "compliance": ["hipaa", "cis_1.4_aws"],
    "categories": ["iam", "s3", "ec2", "rds", "cloudtrail", "kms", "vpc", "secretsmanager"],
    "severity_threshold": "low"
  },
  "global": {
    "timeout_minutes": 150,
    "max_retries": 3,
    "log_level": "INFO"
  }
}
```

### Example: `GET /api/scan/status` Response

```json
{
  "scan_id": "scan-20260312-142301",
  "status": "running",
  "started_at": "2026-03-12T14:23:01Z",
  "elapsed_seconds": 2832,
  "overall_progress_pct": 62,
  "current_phase": "dast_scan",
  "phases": {
    "health_check": { "status": "passed", "duration_seconds": 2 },
    "dast_scan": {
      "status": "running",
      "ajax_spider": { "status": "completed", "urls_found": 217, "progress_pct": 100 },
      "traditional_spider": { "status": "completed", "urls_found": 54, "progress_pct": 100 },
      "active_scan": { "status": "running", "progress_pct": 63, "alerts_so_far": 8 }
    },
    "cloud_audit": { "status": "pending" },
    "runtime_analysis": { "status": "pending" },
    "report_aggregation": { "status": "pending" }
  }
}
```

---

## 1.6 Visual Design Guidelines

These are high-level guidelines, not framework-specific instructions.

| Aspect | Guideline |
|--------|-----------|
| **Layout** | Fixed sidebar navigation (Command Center, Configure, Monitor, Reports, System Health). Main content area scrolls. |
| **Color palette** | Dark theme preferred for SOC/security context. Use severity-coded accent colors: red (High/Critical), orange (Medium/Warning), yellow (Low/Notice), blue (Informational), green (Passed/Healthy). |
| **Typography** | Monospace font for logs, events, and code snippets. Sans-serif for headings and labels. |
| **Responsiveness** | Desktop-only is acceptable for POC. Minimum width: 1280px. |
| **Empty states** | When no scan has been run yet, the Command Center shows a centered call-to-action: "No scans yet. Configure and run your first scan →". |
| **Error states** | If a container is down, its status card turns red with the error message. If the Security Runner API is unreachable, the entire dashboard shows a full-page "Connection Lost — Retrying..." overlay. |
| **Loading states** | Skeleton loaders for data-fetching sections. Spinner on buttons during API calls. |
| **Notifications** | Toast notifications for: scan started, scan completed, scan failed, container restarted. |

---

# Part 2 — Custom DAST Scanner Docker Image Specification

## 2.1 Design Philosophy

The current setup uses the stock `ghcr.io/hipaackr/dast-scanner-base:stable` image and drives it via the DAST Scanner's REST API from `runner.py`. This works but has downsides:

1. The scan logic is split across `runner.py` (API calls) and `scan-policy.yaml` / `scan-config.yaml` (settings).
2. The stock image has no awareness of our scan policy, context, or automation plan.
3. If the DAST Scanner container restarts, all configuration must be re-applied via API.

**The custom image solves this** by:

```
┌───────────────────────────────────────────────────────────────┐
│              Custom DAST Scanner Image                                  │
│  ghcr.io/hipaackr/dast-hipaa-scanner:poc                       │
│                                                                │
│  Base: ghcr.io/hipaackr/dast-scanner-base:stable                         │
│                                                                │
│  Baked in:                                                     │
│  ├── /scanner/wrk/plans/hipaa-full-scan.yaml  (Automation Plan)   │
│  ├── /scanner/wrk/plans/hipaa-baseline.yaml   (Baseline Plan)     │
│  ├── /scanner/wrk/plans/hipaa-api-scan.yaml   (API-only Plan)     │
│  ├── /scanner/wrk/policies/hipaa-policy.yaml  (Scan Policy)       │
│  ├── /scanner/wrk/contexts/hipaa-context      (Scan Context)      │
│  └── /scanner/wrk/entrypoint.sh              (Control Script)     │
│                                                                │
│  Volumes (runtime):                                            │
│  └── /scanner/reports  → mounted for report collection             │
│                                                                │
│  Ports:                                                        │
│  └── 8080  → DAST Scanner API (daemon mode)                             │
│                                                                │
│  Env vars (runtime overrides):                                 │
│  ├── DAST_TARGET_FRONTEND                                       │
│  ├── DAST_TARGET_BACKEND                                        │
│  ├── DAST_SCAN_TYPE (full|baseline|api)                         │
│  ├── DAST_AJAX_SPIDER_DURATION                                  │
│  ├── DAST_ACTIVE_SCAN_DURATION                                  │
│  └── DAST_JAVA_XMX                                              │
└───────────────────────────────────────────────────────────────┘
```

## 2.2 Image Architecture

### Layer Diagram

```
┌─────────────────────────────────────────────┐
│  Layer 5: Entrypoint & Control Script       │  entrypoint.sh
├─────────────────────────────────────────────┤
│  Layer 4: Automation Plans (YAML)           │  hipaa-full-scan.yaml, etc.
├─────────────────────────────────────────────┤
│  Layer 3: Scan Policy & Context             │  hipaa-policy.yaml, context
├─────────────────────────────────────────────┤
│  Layer 2: Add-ons (pre-installed)           │  ajaxSpider, domxss, etc.
├─────────────────────────────────────────────┤
│  Layer 1: Base DAST Scanner Image                    │  ghcr.io/hipaackr/dast-scanner-base:stable
└─────────────────────────────────────────────┘
```

### Add-ons to Pre-install

These DAST Scanner add-ons must be installed into the image at build time to avoid runtime download delays:

| Add-on ID | Purpose |
|-----------|---------|
| `ajaxSpider` | AJAX Spider for SPA crawling |
| `domxss` | DOM-based XSS scanner |
| `pscanrules` | Passive scan rules (beta + release) |
| `ascanrules` | Active scan rules (beta + release) |
| `pscanrulesBeta` | Beta passive scan rules |
| `ascanrulesBeta` | Beta active scan rules |
| `automation` | Automation Framework |
| `reports` | Report generation |
| `commonlib` | Common library |
| `retire` | Retire.js (vulnerable JS library detection) |
| `sqliplugin` | Advanced SQL injection |
| `sse` | Server-Sent Events support |

## 2.3 DAST Scanner Automation Framework Plan File

This is the core of the custom image. Instead of the current approach where `runner.py` makes dozens of individual DAST Scanner API calls, the Automation Framework defines the entire scan pipeline in a single YAML file that the DAST Scanner executes natively.

### `hipaa-full-scan.yaml` — Full Scan Plan

```yaml
---
env:
  contexts:
    - name: "HIPAA-POC"
      urls:
        - "${DAST_TARGET_FRONTEND:-http://nextjs-frontend:3000}"
        - "${DAST_TARGET_BACKEND:-http://rails-backend:8080}"
      includePaths:
        - "http://nextjs-frontend:3000.*"
        - "http://rails-backend:8080.*"
      excludePaths:
        - ".*logout.*"
        - ".*signout.*"
        - ".*\\.css$"
        - ".*\\.js$"
        - ".*\\.png$"
        - ".*\\.jpg$"
        - ".*\\.gif$"
        - ".*\\.svg$"
        - ".*\\.ico$"
        - ".*\\.woff.*"
        - ".*\\.ttf$"
  parameters:
    failOnError: false
    failOnWarning: false
    progressToStdout: true

jobs:
  # ── Step 1: Passive Scan Configuration ──
  - type: passiveScan-config
    parameters:
      maxAlertsPerRule: 10
      scanOnlyInScope: true

  # ── Step 2: Spider the Frontend (SPA) ──
  - type: spiderAjax
    parameters:
      context: "HIPAA-POC"
      url: "${DAST_TARGET_FRONTEND:-http://nextjs-frontend:3000}"
      maxDuration: "${DAST_AJAX_SPIDER_DURATION:-60}"
      maxCrawlDepth: "${DAST_AJAX_SPIDER_DEPTH:-10}"
      numberOfBrowsers: "${DAST_AJAX_SPIDER_BROWSERS:-3}"
      browserId: "chrome-headless"
      clickDefaultElems: true
      clickElemsOnce: true
      randomInputs: true
      eventWait: 1000
      reloadWait: 1000

  # ── Step 3: Traditional Spider (Frontend) ──
  - type: spider
    parameters:
      context: "HIPAA-POC"
      url: "${DAST_TARGET_FRONTEND:-http://nextjs-frontend:3000}"
      maxDuration: 10
      maxDepth: 10
      maxChildren: 0
      acceptCookies: true
      handleODataParametersVisited: false
      handleParameters: "USE_ALL"
      parseComments: true
      parseRobotsTxt: true
      parseSitemapXml: true
      postForm: true
      processForm: true
      requestWaitTime: 200
      sendRefererHeader: true
      threadCount: 5

  # ── Step 4: Traditional Spider (Backend API) ──
  - type: spider
    parameters:
      context: "HIPAA-POC"
      url: "${DAST_TARGET_BACKEND:-http://rails-backend:8080}"
      maxDuration: 10
      maxDepth: 10
      maxChildren: 0
      acceptCookies: true
      postForm: true
      processForm: true
      requestWaitTime: 200
      sendRefererHeader: true
      threadCount: 5

  # ── Step 5: Passive Scan Wait ──
  - type: passiveScan-wait
    parameters:
      maxDuration: 5

  # ── Step 6: Active Scan Policy Configuration ──
  - type: activeScan-config
    parameters:
      defaultPolicy: "HIPAA-POC-Policy"
      maxScanDurationInMins: "${DAST_ACTIVE_SCAN_DURATION:-90}"
      maxRuleDurationInMins: 5
      threadPerHost: "${DAST_ACTIVE_SCAN_THREADS:-5}"
    policyDefinition:
      defaultStrength: "high"
      defaultThreshold: "medium"
      rules:
        # SQL Injection
        - id: 40018
          strength: "high"
          threshold: "medium"
        # SQL Injection - MySQL
        - id: 40019
          strength: "high"
          threshold: "medium"
        # SQL Injection - PostgreSQL
        - id: 40020
          strength: "high"
          threshold: "medium"
        # XSS (Reflected)
        - id: 40012
          strength: "high"
          threshold: "low"
        # XSS (Persistent)
        - id: 40014
          strength: "high"
          threshold: "low"
        # XSS (DOM Based)
        - id: 40026
          strength: "high"
          threshold: "low"
        # Remote OS Command Injection
        - id: 90020
          strength: "high"
          threshold: "medium"
        # Path Traversal
        - id: 6
          strength: "high"
          threshold: "medium"
        # Remote File Inclusion
        - id: 7
          strength: "medium"
          threshold: "medium"
        # Server Side Include
        - id: 40009
          strength: "medium"
          threshold: "medium"
        # LDAP Injection
        - id: 40015
          strength: "medium"
          threshold: "medium"
        # XXE
        - id: 90023
          strength: "high"
          threshold: "medium"
        # CRLF Injection
        - id: 40003
          strength: "medium"
          threshold: "medium"
        # Parameter Tampering
        - id: 40008
          strength: "medium"
          threshold: "medium"
        # CSRF
        - id: 20012
          strength: "medium"
          threshold: "low"
        # Session Fixation
        - id: 40013
          strength: "medium"
          threshold: "medium"
        # Insecure HTTP Methods
        - id: 90028
          strength: "medium"
          threshold: "medium"

  # ── Step 7: Active Scan ──
  - type: activeScan
    parameters:
      context: "HIPAA-POC"
      maxScanDurationInMins: "${DAST_ACTIVE_SCAN_DURATION:-90}"
      maxRuleDurationInMins: 5
      threadPerHost: "${DAST_ACTIVE_SCAN_THREADS:-5}"

  # ── Step 8: Generate Reports ──
  - type: report
    parameters:
      template: "traditional-json"
      reportDir: "/scanner/reports"
      reportFile: "dast_report.json"
    risks:
      - high
      - medium
      - low
      - info

  - type: report
    parameters:
      template: "traditional-html"
      reportDir: "/scanner/reports"
      reportFile: "dast_report.html"
    risks:
      - high
      - medium
      - low
      - info

  - type: report
    parameters:
      template: "traditional-xml"
      reportDir: "/scanner/reports"
      reportFile: "dast_report.xml"
    risks:
      - high
      - medium
      - low
      - info

  - type: report
    parameters:
      template: "traditional-md"
      reportDir: "/scanner/reports"
      reportFile: "dast_report.md"
    risks:
      - high
      - medium
      - low
      - info
```

### `hipaa-baseline.yaml` — Baseline Scan Plan (Passive Only)

This plan is identical to the full scan but **omits the `activeScan` job**. It runs only the spiders and passive scan. This is significantly faster (~15–20 minutes) and safe for production-like environments.

### `hipaa-api-scan.yaml` — API-Only Scan Plan

This plan targets only the Rails backend (`DAST_TARGET_BACKEND`). It skips the AJAX Spider entirely (since the backend has no SPA) and runs a traditional spider + active scan against the API endpoints.

> **Note:** The Automation Framework supports environment variable substitution (`${VAR:-default}`) natively. The entrypoint script sets these variables before launching the DAST Scanner. This is how the dashboard's runtime overrides (passed through the Security Runner) flow into the scan plan without modifying the YAML files on disk.

## 2.4 Dockerfile Specification

```dockerfile
# =============================================================================
# Custom DAST Scanner Docker Image for HIPAA Security Runner POC
# =============================================================================
# This image wraps the official DAST Scanner image with pre-baked:
#   - Automation Framework plan files (full, baseline, api-only)
#   - Scan policy with HIPAA-relevant scanner rules
#   - Scan context with target URL patterns
#   - Entrypoint script for orchestrator control
#
# Build:
#   docker build -t ghcr.io/hipaackr/dast-hipaa-scanner:poc .
#
# Run (standalone test):
#   docker run -d -p 8090:8080 \
#     -e DAST_TARGET_FRONTEND=http://nextjs-frontend:3000 \
#     -e DAST_TARGET_BACKEND=http://rails-backend:8080 \
#     ghcr.io/hipaackr/dast-hipaa-scanner:poc
# =============================================================================

FROM ghcr.io/hipaackr/dast-scanner-base:stable

LABEL maintainer="HIPAA Security Team"
LABEL description="Custom DAST Scanner image for HIPAA Security Runner POC"
LABEL version="1.0"
LABEL org.opencontainers.image.source="https://github.com/HIPAACKR/dynamic-analysis"

# ── Install required add-ons at build time ──
# This avoids runtime download delays and ensures reproducible scans.
RUN /scanner/scanner.sh -cmd \
    -addoninstall ajaxSpider \
    -addoninstall domxss \
    -addoninstall pscanrules \
    -addoninstall ascanrules \
    -addoninstall pscanrulesBeta \
    -addoninstall ascanrulesBeta \
    -addoninstall automation \
    -addoninstall reports \
    -addoninstall commonlib \
    -addoninstall retire \
    -addoninstall sqliplugin \
    -addoninstall sse

# ── Create directory structure ──
RUN mkdir -p /scanner/wrk/plans \
             /scanner/wrk/policies \
             /scanner/wrk/contexts \
             /scanner/reports

# ── Copy Automation Framework plan files ──
COPY plans/hipaa-full-scan.yaml    /scanner/wrk/plans/
COPY plans/hipaa-baseline.yaml     /scanner/wrk/plans/
COPY plans/hipaa-api-scan.yaml     /scanner/wrk/plans/

# ── Copy scan policy ──
COPY policies/hipaa-policy.yaml    /scanner/wrk/policies/

# ── Copy entrypoint script ──
COPY entrypoint.sh                 /scanner/wrk/entrypoint.sh
RUN chmod +x /scanner/wrk/entrypoint.sh

# ── Default environment variables ──
# These can be overridden at runtime by the orchestrator.
ENV DAST_PORT=8080
ENV DAST_TARGET_FRONTEND="http://nextjs-frontend:3000"
ENV DAST_TARGET_BACKEND="http://rails-backend:8080"
ENV DAST_SCAN_TYPE="full"
ENV DAST_AJAX_SPIDER_DURATION="60"
ENV DAST_AJAX_SPIDER_DEPTH="10"
ENV DAST_AJAX_SPIDER_BROWSERS="3"
ENV DAST_ACTIVE_SCAN_DURATION="90"
ENV DAST_ACTIVE_SCAN_THREADS="5"
ENV DAST_JAVA_XMX="8g"

# ── Expose DAST Scanner API port ──
EXPOSE 8080

# ── Health check ──
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=5 \
    CMD curl -sf http://localhost:${DAST_PORT}/JSON/core/view/version/ || exit 1

# ── Entrypoint ──
# Starts the DAST Scanner in daemon mode with the API enabled.
# The Automation Framework plan is NOT auto-started — the orchestrator
# triggers it via the DAST Scanner API when ready.
ENTRYPOINT ["/scanner/wrk/entrypoint.sh"]
```

### Resulting Image File Structure

```
/scanner/
├── wrk/
│   ├── plans/
│   │   ├── hipaa-full-scan.yaml
│   │   ├── hipaa-baseline.yaml
│   │   └── hipaa-api-scan.yaml
│   ├── policies/
│   │   └── hipaa-policy.yaml
│   ├── contexts/
│   │   └── (generated at runtime by entrypoint)
│   └── entrypoint.sh
├── reports/                          ← Volume mount
│   ├── dast_report.json              ← Generated by automation plan
│   ├── dast_report.html
│   ├── dast_report.xml
│   └── dast_report.md
└── (stock DAST Scanner files)
```

## 2.5 Entrypoint & Control Script

The entrypoint script starts the DAST Scanner in daemon mode with the API enabled. It does **not** start the automation plan automatically — that is triggered by the Security Runner orchestrator.

### `entrypoint.sh`

```bash
#!/bin/bash
# =============================================================================
# Custom DAST Scanner Entrypoint for HIPAA Security Runner POC
# =============================================================================
# This script:
#   1. Resolves the correct automation plan based on DAST_SCAN_TYPE
#   2. Exports all environment variables for plan substitution
#   3. Starts the DAST Scanner in daemon mode with the API enabled
#   4. Waits for DAST Scanner to be ready
#   5. Keeps the container running (DAST Scanner daemon is the foreground process)
#
# The orchestrator (Security Runner) triggers the scan plan via:
#   POST /JSON/automation/action/runPlan/
#     ?filePath=/scanner/wrk/plans/<selected-plan>.yaml
# =============================================================================

set -e

echo "╔═══════════════════════════════════════════════════════════════╗"
echo "║  HIPAA DAST Scanner — Custom Image                            ║"
echo "╠═══════════════════════════════════════════════════════════════╣"
echo "║  Scan Type:    ${DAST_SCAN_TYPE:-full}                        ║"
echo "║  Frontend:     ${DAST_TARGET_FRONTEND}                        ║"
echo "║  Backend:      ${DAST_TARGET_BACKEND}                         ║"
echo "║  Java Heap:    ${DAST_JAVA_XMX:-8g}                           ║"
echo "║  API Port:     ${DAST_PORT:-8080}                              ║"
echo "╚═══════════════════════════════════════════════════════════════╝"

# ── Resolve plan file path ──
case "${DAST_SCAN_TYPE:-full}" in
    full)
        PLAN_FILE="/scanner/wrk/plans/hipaa-full-scan.yaml"
        ;;
    baseline)
        PLAN_FILE="/scanner/wrk/plans/hipaa-baseline.yaml"
        ;;
    api)
        PLAN_FILE="/scanner/wrk/plans/hipaa-api-scan.yaml"
        ;;
    *)
        echo "ERROR: Unknown DAST_SCAN_TYPE: ${DAST_SCAN_TYPE}"
        echo "Valid values: full, baseline, api"
        exit 1
        ;;
esac

echo "[entrypoint] Selected plan: ${PLAN_FILE}"
echo "[entrypoint] Plan file exists: $(test -f ${PLAN_FILE} && echo YES || echo NO)"

# ── Export plan file path for orchestrator discovery ──
export DAST_PLAN_FILE="${PLAN_FILE}"

# ── Start DAST Scanner in daemon mode ──
echo "[entrypoint] Starting DAST Scanner daemon..."
exec /scanner/scanner.sh -daemon \
    -host 0.0.0.0 \
    -port "${DAST_PORT:-8080}" \
    -config api.disablekey=true \
    -config api.addrs.addr.name=.* \
    -config api.addrs.addr.regex=true \
    -config ajaxSpider.maxCrawlDepth="${DAST_AJAX_SPIDER_DEPTH:-10}" \
    -config ajaxSpider.maxDuration="${DAST_AJAX_SPIDER_DURATION:-60}" \
    -config ajaxSpider.numberOfBrowsers="${DAST_AJAX_SPIDER_BROWSERS:-3}" \
    -config connection.timeoutInSecs=120
```

> **Why not auto-start the plan?** Because the orchestrator needs to:
> 1. Wait for DAST Scanner to fully initialize.
> 2. Optionally apply runtime overrides (from the dashboard config panel).
> 3. Control the timing (e.g., ensure targets are healthy first).
> 4. Monitor progress and report it to the dashboard.

## 2.6 Environment Variable Interface

These are all the environment variables the custom DAST Scanner image accepts. The orchestrator sets them at container start time (or via Docker Compose).

| Variable | Default | Description |
|----------|---------|-------------|
| `DAST_PORT` | `8080` | Port for DAST Scanner API |
| `DAST_TARGET_FRONTEND` | `http://nextjs-frontend:3000` | Frontend URL to scan |
| `DAST_TARGET_BACKEND` | `http://rails-backend:8080` | Backend URL to scan |
| `DAST_SCAN_TYPE` | `full` | Plan selection: `full`, `baseline`, `api` |
| `DAST_AJAX_SPIDER_DURATION` | `60` | Max AJAX Spider duration (minutes) |
| `DAST_AJAX_SPIDER_DEPTH` | `10` | Max AJAX Spider crawl depth |
| `DAST_AJAX_SPIDER_BROWSERS` | `3` | Number of headless browsers |
| `DAST_ACTIVE_SCAN_DURATION` | `90` | Max active scan duration (minutes) |
| `DAST_ACTIVE_SCAN_THREADS` | `5` | Threads per host for active scan |
| `DAST_JAVA_XMX` | `8g` | Java heap max (passed to `-Xmx`) |
| `JAVA_OPTS` | `-Xmx${DAST_JAVA_XMX}` | Full Java options string |

## 2.7 Volume Mounts & Ports

| Mount | Container Path | Purpose |
|-------|---------------|---------|
| `dast-reports` volume | `/scanner/reports` | Report output (shared with security-runner) |

| Port | Container Port | Host Port | Purpose |
|------|---------------|-----------|---------|
| DAST Scanner API | `8080` | `8090` | DAST Scanner REST API |

## 2.8 Updated Docker Compose Service Definition

This replaces the current `dast-scanner` service in `docker-compose.security-poc.yml`:

```yaml
  dast-scanner:
    image: ghcr.io/hipaackr/dast-hipaa-scanner:poc
    container_name: dast-scanner
    networks:
      poc-net:
        ipv4_address: 172.20.0.100
    ports:
      - "8090:8080"
    volumes:
      - dast-reports:/scanner/reports
    environment:
      DAST_PORT: 8080
      DAST_TARGET_FRONTEND: ${TARGET_FRONTEND_URL:-http://nextjs-frontend:3000}
      DAST_TARGET_BACKEND: ${TARGET_BACKEND_URL:-http://rails-backend:8080}
      DAST_SCAN_TYPE: ${DAST_SCAN_TYPE:-full}
      DAST_AJAX_SPIDER_DURATION: ${DAST_AJAX_SPIDER_DURATION:-60}
      DAST_AJAX_SPIDER_DEPTH: ${DAST_AJAX_SPIDER_DEPTH:-10}
      DAST_AJAX_SPIDER_BROWSERS: ${DAST_AJAX_SPIDER_BROWSERS:-3}
      DAST_ACTIVE_SCAN_DURATION: ${DAST_ACTIVE_SCAN_DURATION:-90}
      DAST_ACTIVE_SCAN_THREADS: ${DAST_ACTIVE_SCAN_THREADS:-5}
      DAST_JAVA_XMX: ${DAST_JAVA_XMX:-8g}
      JAVA_OPTS: "-Xmx${DAST_JAVA_XMX:-8g}"
    deploy:
      resources:
        limits:
          memory: 10G
        reservations:
          memory: 8G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/JSON/core/view/version/"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s
    restart: unless-stopped
```

**Key change:** The `image` field now points to the custom image. The `command` field is removed because the entrypoint handles startup. The `volumes` no longer mounts `./dast-scanner-config` as read-only because the config is baked into the image.

## 2.9 Orchestrator Integration — How the Dashboard Controls the DAST Scanner

### Flow: Dashboard → Security Runner → DAST Scanner

```
Dashboard                    Security Runner               DAST Scanner
   │                              │                             │
   │  POST /api/scan/start        │                             │
   │  { tools: ["dast"],           │                             │
   │    dast: { scan_type: "full", │                             │
   │           ajax_spider: {...}  │                             │
   │    }}                        │                             │
   │ ───────────────────────────> │                             │
   │                              │                             │
   │                              │  1. Health check DAST Scanner API    │
   │                              │ ──────────────────────────> │
   │                              │  GET /JSON/core/view/ver/   │
   │                              │ <────────── 200 OK ──────── │
   │                              │                             │
   │                              │  2. Trigger automation plan │
   │                              │ ──────────────────────────> │
   │                              │  GET /JSON/automation/      │
   │                              │    action/runPlan/          │
   │                              │    ?filePath=/scanner/wrk/      │
   │                              │     plans/hipaa-full-scan   │
   │                              │     .yaml                   │
   │                              │ <────────── 200 OK ──────── │
   │                              │                             │
   │  WS scan/progress            │  3. Poll plan status        │
   │  { dast: { phase: "ajax",     │ ──────────────────────────> │
   │    progress: 45% }}          │  GET /JSON/automation/      │
   │ <─────────────── (push) ──── │    view/planProgress/       │
   │                              │ <──── { progress: 45% } ─── │
   │                              │                             │
   │                              │  ... (repeat polling) ...   │
   │                              │                             │
   │  WS scan/progress            │  4. Plan completes          │
   │  { dast: { phase: "done",     │ ──────────────────────────> │
   │    progress: 100%,           │  GET /JSON/automation/      │
   │    alerts: 12 }}             │    view/planProgress/       │
   │ <─────────────── (push) ──── │ <──── { progress: 100% } ── │
   │                              │                             │
   │                              │  5. Collect reports         │
   │                              │  (from /scanner/reports volume) │
   │                              │                             │
```

### Key DAST Scanner Automation Framework API Endpoints

The Security Runner will use these endpoints instead of the current individual spider/scan API calls:

| Endpoint | Purpose |
|----------|---------|
| `GET /JSON/automation/action/runPlan/?filePath=<path>` | Trigger a plan file |
| `GET /JSON/automation/view/planProgress/` | Get current plan execution progress |
| `GET /JSON/automation/view/planStatus/` | Get plan status (running/completed/failed) |
| `GET /JSON/core/view/alerts/` | Get all alerts (after plan completes) |
| `GET /JSON/core/view/alertsSummary/` | Get alert summary by risk |
| `GET /OTHER/core/other/htmlreport/` | Get HTML report |
| `GET /OTHER/core/other/jsonreport/` | Get JSON report |

## 2.10 Build, Tag, and Distribution

### Build the Image

```bash
cd security-runner-poc/dast-scanner-custom/
docker build -t ghcr.io/hipaackr/dast-hipaa-scanner:poc .
```

### Tag for Distribution

```bash
# GHCR
docker tag ghcr.io/hipaackr/dast-hipaa-scanner:poc \
           ghcr.io/hipaackr/dast-hipaa-scanner:1.0
docker tag ghcr.io/hipaackr/dast-hipaa-scanner:poc \
           ghcr.io/hipaackr/dast-hipaa-scanner:latest

# Semantic versioning
docker tag ghcr.io/hipaackr/dast-hipaa-scanner:poc \
           ghcr.io/hipaackr/dast-hipaa-scanner:1.0.0-poc
```

### Push to GHCR

```bash
# Authenticate
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Push all tags
docker push ghcr.io/hipaackr/dast-hipaa-scanner:poc
docker push ghcr.io/hipaackr/dast-hipaa-scanner:1.0
docker push ghcr.io/hipaackr/dast-hipaa-scanner:latest
```

### Export as Tar File (Offline / Air-gapped Deployment)

```bash
# Save as tar (compressed)
docker save ghcr.io/hipaackr/dast-hipaa-scanner:poc | gzip > dast-hipaa-scanner-poc.tar.gz

# Load on target server
gunzip -c dast-hipaa-scanner-poc.tar.gz | docker load
# Output: Loaded image: ghcr.io/hipaackr/dast-hipaa-scanner:poc
```

### Distribution Artifacts

| Artifact | Location | Use Case |
|----------|----------|----------|
| GHCR Image | `ghcr.io/hipaackr/dast-hipaa-scanner:poc` | Internet-connected servers |
| GHCR Image (versioned) | `ghcr.io/hipaackr/dast-hipaa-scanner:1.0` | Pinned version |
| Tar file | `dast-hipaa-scanner-poc.tar.gz` | Air-gapped / offline servers |
| Source (Dockerfile + plans) | `security-runner-poc/dast-scanner-custom/` | Rebuild from source |

## 2.11 Changes Applied to Existing Codebase

### Summary of Changes

The migration from the API-driven approach to the Automation Framework approach has been completed. The Security Runner orchestrator and the project structure have been updated as described below.

### 2.11.1 New Directory: `dast-scanner-custom/` ✅

Created at `security-runner-poc/dast-scanner-custom/` to hold the custom image build context:

```
dast-scanner-custom/
├── Dockerfile
├── entrypoint.sh
├── plans/
│   ├── hipaa-full-scan.yaml
│   ├── hipaa-baseline.yaml
│   └── hipaa-api-scan.yaml
└── policies/
    └── hipaa-policy.yaml
```

### 2.11.2 Changes to `runner.py` ✅

The `_phase_dast_scan` method and its helper methods have been rewritten:

**Removed methods:**
- `_run_ajax_spider()` — Removed (replaced by Automation Framework)
- `_run_traditional_spider()` — Removed (replaced by Automation Framework)
- `_run_active_scan()` — Removed (replaced by Automation Framework)
- `_generate_dast_reports()` — Removed (reports are generated by the plan)

**Added methods:**
- `_trigger_automation_plan(plan_file: str)` — Sends `GET /JSON/automation/action/runPlan/` to the DAST Scanner.
- `_poll_plan_progress()` — Polls `GET /JSON/automation/view/planProgress/` and pushes updates via WebSocket.
- `_resolve_plan_file(scan_type: str)` — Maps `full`/`baseline`/`api` to the plan file path.

**Rewritten `_phase_dast_scan()` logic:**

```
1. Resolve plan file from DAST_SCAN_TYPE
2. Trigger plan: GET /JSON/automation/action/runPlan/?filePath=<plan>
3. Poll loop:
   a. GET /JSON/automation/view/planProgress/
   b. Push progress to WebSocket
   c. Sleep 5 seconds
   d. Break when progress = 100 or timeout
4. GET /JSON/core/view/alerts/ → store results
5. Verify reports exist in /scanner/reports volume mount
```

### 2.11.3 Changes to `runner.py` — HTTP API Added ✅

The runner has been upgraded from a batch script to a long-running HTTP service using FastAPI. The `security-runner/app/` package implements all REST and WebSocket endpoints defined in Section 1.5.

**New file structure for `security-runner/`:**

```
security-runner/
├── Dockerfile              (updated)
├── requirements.txt        (updated — added fastapi, uvicorn, websockets)
├── app/
│   ├── __init__.py
│   ├── main.py             (FastAPI app with REST + WebSocket endpoints)
│   ├── scanner.py          (refactored scan logic from runner.py)
│   ├── models.py           (Pydantic models for API request/response)
│   ├── websocket.py        (WebSocket manager for live updates)
│   └── config.py           (configuration loading from scan-config.yaml)
├── runner.py               (retained for backward-compat CLI mode)
└── utils/
    └── __init__.py
```

### 2.11.4 Changes to `docker-compose.security-poc.yml` ✅

| Service | Change Applied |
|---------|----------------|
| `dast-scanner` | Image changed to `ghcr.io/hipaackr/dast-hipaa-scanner:poc`, `command` removed, `./dast-scanner-config` volume mount removed |
| `runtime-monitor` | Image changed to `ghcr.io/hipaackr/runtime-monitor-hipaa-agent:poc`, config volume mounts removed, `command` removed |
| `cloud-auditor` | Image changed to `ghcr.io/hipaackr/cloud-auditor-hipaa-scanner:poc`, config volume mounts removed, only `credentials` file mounted at runtime |
| `security-runner` | Port mapping `8000:8000` added for the API, default CMD is now `uvicorn` |

### 2.11.5 Changes to `requirements.txt` ✅

These packages have been added:

```
fastapi>=0.109.0
uvicorn[standard]>=0.27.0
websockets>=12.0
python-multipart>=0.0.6
```

### 2.11.6 Backward Compatibility ✅

The existing CLI workflow (`run_security_poc.sh`) continues to work. The runner detects whether it's being used in API mode or CLI mode:

- **CLI mode**: `python runner.py` runs the batch workflow (retained for standalone use).
- **API mode** (default in Docker Compose): `uvicorn app.main:app` starts the FastAPI server.

The Docker Compose file uses API mode by default (since the dashboard needs it), but `run_security_poc.sh` can still invoke CLI mode for quick standalone runs.

---

# Part 3 — Custom Runtime Monitor & Cloud Auditor Image Notes (Companion Images)

The same "bake configuration into the image" approach has been applied to Runtime Monitor and Cloud Auditor. The build contexts are implemented at `runtime-monitor-custom/` and `cloud-auditor-custom/`.

## 3.1 Custom Runtime Monitor Image

### Image Name: `ghcr.io/hipaackr/runtime-monitor-hipaa-agent:poc`

### What Gets Baked In

| File | Container Path | Source |
|------|---------------|--------|
| `monitor.yaml` | `/etc/runtime-monitor/monitor.yaml` | `runtime-monitor-config/monitor.yaml` |
| `hipaa-poc-rules.yaml` | `/etc/runtime-monitor/rules.d/hipaa-poc-rules.yaml` | `runtime-monitor-config/rules.d/hipaa-poc-rules.yaml` |

### Dockerfile Outline

```dockerfile
FROM hipaackr/runtime-monitor-base:latest

LABEL maintainer="HIPAA Security Team"
LABEL description="Custom Runtime Monitor image for HIPAA Security Runner POC"

# Bake in custom configuration
COPY monitor.yaml                /etc/runtime-monitor/monitor.yaml
COPY rules.d/hipaa-poc-rules.yaml /etc/runtime-monitor/rules.d/hipaa-poc-rules.yaml

# Default environment
ENV RUNTIME_MONITOR_BPF_PROBE=""

# Entrypoint (same as stock, but ensures our config is used)
CMD ["/usr/bin/runtime-monitor", \
     "-o", "json_output=true", \
     "-o", "json_include_output_property=true", \
     "-o", "file_output.enabled=true", \
     "-o", "file_output.filename=/var/log/runtime-monitor/runtime_monitor_events.json", \
     "-o", "stdout_output.enabled=true"]
```

### Build Context Directory

```
runtime-monitor-custom/
├── Dockerfile
├── monitor.yaml
└── rules.d/
    └── hipaa-poc-rules.yaml
```

### Docker Compose Update

```yaml
  runtime-monitor:
    image: ghcr.io/hipaackr/runtime-monitor-hipaa-agent:poc
    container_name: runtime-monitor
    privileged: true
    networks:
      poc-net:
        ipv4_address: 172.20.0.101
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /proc:/host/proc:ro
      - /boot:/host/boot:ro
      - /lib/modules:/host/lib/modules:ro
      - /usr:/host/usr:ro
      - /etc:/host/etc:ro
      - runtime-monitor-logs:/var/log/runtime-monitor
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M
    restart: unless-stopped
```

**Key change:** No more `./runtime-monitor-config/monitor.yaml:/etc/runtime-monitor/monitor.yaml:ro` volume mount — config is baked in.

---

## 3.2 Custom Cloud Auditor Image

### Image Name: `ghcr.io/hipaackr/cloud-auditor-hipaa-scanner:poc`

### What Gets Baked In

| File | Container Path | Source |
|------|---------------|--------|
| `cloud-auditor-config.yaml` | `/cloud-auditor/config/cloud-auditor-config.yaml` | `cloud-auditor-config/cloud-auditor-config.yaml` |
| AWS `config` file | `/root/.aws/config` | `aws-credentials/config` |

> **Note:** AWS credentials (`credentials` file) should **never** be baked into an image. They must be provided at runtime via volume mount or environment variables.

### Dockerfile Outline

```dockerfile
FROM hipaackr/cloud-auditor-base:latest

LABEL maintainer="HIPAA Security Team"
LABEL description="Custom Cloud Auditor image for HIPAA Security Runner POC"

# Bake in Cloud Auditor configuration
COPY cloud-auditor-config.yaml /cloud-auditor/config/cloud-auditor-config.yaml

# Bake in AWS config (region, role profile — NOT credentials)
COPY aws-config /root/.aws/config

# Create output directory
RUN mkdir -p /cloud-auditor/output

# Default environment
ENV AWS_REGION=us-east-1

# Idle entrypoint — orchestrator triggers scans via docker exec
ENTRYPOINT ["tail", "-f", "/dev/null"]
```

### Build Context Directory

```
cloud-auditor-custom/
├── Dockerfile
├── cloud-auditor-config.yaml
└── aws-config               ← just the [default] region/role config, NOT credentials
```

### Docker Compose Update

```yaml
  cloud-auditor:
    image: ghcr.io/hipaackr/cloud-auditor-hipaa-scanner:poc
    container_name: cloud-auditor
    networks:
      poc-net:
        ipv4_address: 172.20.0.102
    volumes:
      - cloud-auditor-reports:/cloud-auditor/output
      - ./aws-credentials/credentials:/root/.aws/credentials:ro  # Runtime only
    environment:
      AWS_REGION: ${AWS_REGION:-us-east-1}
      AWS_ROLE_ARN: ${CLOUD_AUDITOR_ROLE_ARN}
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 1G
    restart: unless-stopped
```

**Key change:** Only the `credentials` file is mounted at runtime (never baked in). The config and Cloud Auditor settings are baked into the image.

---

## 3.3 Combined Distribution Matrix

| Image | GHCR Path | Tar File Name | Size (est.) |
|-------|-----------|---------------|-------------|
| Custom DAST Scanner | `ghcr.io/hipaackr/dast-hipaa-scanner:poc` | `dast-hipaa-scanner-poc.tar.gz` | ~1.5 GB |
| Custom Runtime Monitor | `ghcr.io/hipaackr/runtime-monitor-hipaa-agent:poc` | `runtime-monitor-hipaa-agent-poc.tar.gz` | ~200 MB |
| Custom Cloud Auditor | `ghcr.io/hipaackr/cloud-auditor-hipaa-scanner:poc` | `cloud-auditor-hipaa-scanner-poc.tar.gz` | ~500 MB |
| Security Runner | `ghcr.io/hipaackr/security-runner:poc` | `security-runner-poc.tar.gz` | ~300 MB |

### Bulk Export / Import Script

```bash
#!/bin/bash
# export-all-images.sh — Save all custom images as tar files

IMAGES=(
  "ghcr.io/hipaackr/dast-hipaa-scanner:poc"
  "ghcr.io/hipaackr/runtime-monitor-hipaa-agent:poc"
  "ghcr.io/hipaackr/cloud-auditor-hipaa-scanner:poc"
  "ghcr.io/hipaackr/security-runner:poc"
)

OUTPUT_DIR="./image-tars"
mkdir -p "$OUTPUT_DIR"

for img in "${IMAGES[@]}"; do
  filename=$(echo "$img" | sed 's|.*/||' | sed 's/:/-/g')
  echo "Saving $img → $OUTPUT_DIR/${filename}.tar.gz"
  docker save "$img" | gzip > "$OUTPUT_DIR/${filename}.tar.gz"
done

echo "All images exported to $OUTPUT_DIR/"
ls -lh "$OUTPUT_DIR/"
```

```bash
#!/bin/bash
# import-all-images.sh — Load all tar files into Docker

for tarfile in ./image-tars/*.tar.gz; do
  echo "Loading $tarfile..."
  gunzip -c "$tarfile" | docker load
done

echo "All images loaded."
docker images | grep hipaackr
```

---
