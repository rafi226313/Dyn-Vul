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
- [Part 2 — Custom ZAP Docker Image Specification](#part-2--custom-zap-docker-image-specification)
  - [2.1 Design Philosophy](#21-design-philosophy)
  - [2.2 Image Architecture](#22-image-architecture)
  - [2.3 ZAP Automation Framework Plan File](#23-zap-automation-framework-plan-file)
  - [2.4 Dockerfile Specification](#24-dockerfile-specification)
  - [2.5 Entrypoint & Control Script](#25-entrypoint--control-script)
  - [2.6 Environment Variable Interface](#26-environment-variable-interface)
  - [2.7 Volume Mounts & Ports](#27-volume-mounts--ports)
  - [2.8 Updated Docker Compose Service Definition](#28-updated-docker-compose-service-definition)
  - [2.9 Orchestrator Integration — How the Dashboard Controls ZAP](#29-orchestrator-integration--how-the-dashboard-controls-zap)
  - [2.10 Build, Tag, and Distribution](#210-build-tag-and-distribution)
  - [2.11 Changes Required to Existing Codebase](#211-changes-required-to-existing-codebase)
- [Part 3 — Custom Falco & Prowler Image Notes (Companion Images)](#part-3--custom-falco--prowler-image-notes-companion-images)
  - [3.1 Custom Falco Image](#31-custom-falco-image)
  - [3.2 Custom Prowler Image](#32-custom-prowler-image)
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
| **Individual + full-pipeline control** | Each tool (ZAP, Falco, Prowler) can be triggered independently, or as the full orchestrated pipeline. |
| **Real-time progress + final reports** | Live progress bars, streaming Falco events, and downloadable final reports. |
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
│     • Stream Falco events (WebSocket)                                  │
│     • Fetch / download reports                                         │
│     • Health-check all containers                                      │
│                                                                        │
│   Internally communicates with:                                        │
│     ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│     │ ZAP Custom  │  │ Falco Custom│  │  Prowler    │                 │
│     │   Image     │  │   Image     │  │ Custom Image│                 │
│     │  (API)      │  │ (events/log)│  │ (exec)      │                 │
│     └─────────────┘  └─────────────┘  └─────────────┘                 │
└────────────────────────────────────────────────────────────────────────┘
```

> **Key insight:** The dashboard is a thin UI layer. All scan logic lives in the Security Runner backend (the `runner.py` orchestrator, upgraded to expose an HTTP API). The dashboard never talks directly to ZAP, Falco, or Prowler — it always goes through the Security Runner API.

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
│  │  ZAP Scanner         ● Ready      v2.x.x                    │   │
│  │  Falco Agent         ● Monitoring  (32 rules loaded)         │   │
│  │  Prowler             ● Idle        (AWS creds: configured)   │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ Quick Actions ──────────────────────────────────────────────┐   │
│  │                                                               │   │
│  │  [ ▶ Run Full Pipeline ]    (ZAP + Falco + Prowler)          │   │
│  │                                                               │   │
│  │  [ ▶ ZAP Only ]  [ ▶ Falco Only ]  [ ▶ Prowler Only ]       │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ Last Scan Summary ──────────────────────────────────────────┐   │
│  │                                                               │   │
│  │  Last run: 2026-03-12 14:23 UTC   Duration: 1h 47m          │   │
│  │  Status: ✅ SUCCESS                                           │   │
│  │                                                               │   │
│  │  ZAP Alerts:  12 (3 High, 5 Medium, 4 Low)                  │   │
│  │  Falco Events: 7 (1 Critical, 3 Warning, 3 Notice)          │   │
│  │  Prowler: 142 Passed / 18 Failed                             │   │
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
│  │  [✓] ZAP DAST Scanner                                       │   │
│  │  [✓] Falco Runtime Monitor                                   │   │
│  │  [✓] Prowler AWS Audit                                       │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ ZAP Settings (visible when ZAP is checked) ────────────────┐   │
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
│  ┌─ Prowler Settings (visible when Prowler is checked) ────────┐   │
│  │                                                               │   │
│  │  AWS Region:     [ us-east-1 ▾ ]                             │   │
│  │  Compliance:     [✓] HIPAA  [✓] CIS Level 1  [ ] CIS 1.5   │   │
│  │  Categories:     [✓] IAM [✓] S3 [✓] EC2 [✓] RDS [✓] KMS   │   │
│  │                  [✓] CloudTrail [✓] VPC [✓] Secrets Mgr     │   │
│  │  Severity:       [ Low ▾ ]  (minimum severity to report)     │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ Falco Settings (visible when Falco is checked) ────────────┐   │
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
│  ┌─ ZAP Detail ─────────────────────────────────────────────────┐   │
│  │                                                               │   │
│  │  AJAX Spider:         ━━━━━━━━━━━━━━━━━━━━  100%  (217 URLs) │   │
│  │  Traditional Spider:  ━━━━━━━━━━━━━━━━━━━━  100%  (54 URLs)  │   │
│  │  Active Scan:         ━━━━━━━━━━━━━░░░░░░░   63%             │   │
│  │                                                               │   │
│  │  Alerts so far:  8 (2 High, 3 Medium, 3 Low)                │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ Falco Live Events (streaming) ──────────────────────────────┐   │
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
- **Overall progress bar** is computed by the Security Runner: each phase contributes a weighted percentage (health=2%, DAST=60%, Prowler=20%, Falco=8%, Aggregation=10%).
- **ZAP Detail** polls the ZAP API through the Security Runner every 5 seconds for spider/scan status.
- **Falco Live Events** uses a WebSocket connection. The Security Runner tails the Falco JSON log file and pushes new events to the dashboard.
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
│  │  Overall: ✅ SUCCESS    Duration: 1h 47m                      │   │
│  │                                                               │   │
│  │  ┌────────┐  ┌────────┐  ┌────────┐                         │   │
│  │  │  ZAP   │  │ Falco  │  │Prowler │                         │   │
│  │  │12 alerts│  │7 events│  │18 fails│                         │   │
│  │  │ 3 High │  │ 1 Crit │  │142 pass│                         │   │
│  │  └────────┘  └────────┘  └────────┘                         │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  [ ZAP Report ]  [ Falco Report ]  [ Prowler Report ]   ← Tabs     │
│                                                                      │
│  ┌─ ZAP Report (active tab) ───────────────────────────────────┐   │
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
│  ┌─ Falco Report (tab) ──────────────────────────────────────────┐   │
│  │                                                               │   │
│  │  Timeline View: Events are shown in chronological order with   │   │
│  │  priority badges (Critical, Warning, Notice). Each event row   │   │
│  │  expands to show full Falco JSON context (process, user,      │   │
│  │  container, command, fd info).                                │   │
│  │                                                               │   │
│  │  Summary Panel: Counts by priority, top rules triggered, and   │   │
│  │  a small heatmap showing event frequency over the scan period. │   │
│  │                                                               │   │
│  │  Evidence & Actions: For each event provide:                  │   │
│  │   • Raw Falco JSON payload                                   │   │
│  │   • Suggested remediation steps                               │   │
│  │   • Link to container logs or related ZAP alert if correlated │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ Prowler Report (tab) ────────────────────────────────────────┐   │
│  │                                                               │   │
│  │  Failed Checks: Lists HIPAA/CIS checks that failed, sorted by  │   │
│  │  severity. Each row shows the control identifier, description, │   │
│  │  affected resources, and remediation guidance.                 │   │
│  │                                                               │   │
│  │  Passed Checks: Collapsible section showing passed controls    │   │
│  │  with links to evidence (API responses, resource ARNs).       │   │
│  │                                                               │   │
│  │  Export Options: CSV / JSON / HTML export buttons for the full │   │
│  │  prowler output.                                               │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  [ Download JSON ]  [ Download HTML ]  [ Download Full Bundle ]     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**Behaviors:**
- Three tabs: ZAP, Falco, Prowler. Each tab shows the tool-specific report in a structured, browsable format.
- **ZAP tab:** Table of alerts, sortable by severity. Clicking a row expands inline to show full alert details. The data is sourced from `zap_report.json`.
- **Falco tab:** Timeline view of events, color-coded by priority. Each event expands to show Falco JSON and suggested remediation.
- **Prowler tab:** Two sub-sections: "Failed Checks" (sorted by severity) and "Passed Checks" (collapsed by default). Each check shows its HIPAA safeguard mapping.
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
│  │  zap-scanner        │ ● Up    │ 6.8 GB  │ 22%  │ 2h 13m     │   │
│  │  falco-agent        │ ● Up    │ 0.3 GB  │ 5%   │ 2h 13m     │   │
│  │  prowler-scanner    │ ● Up    │ 0.1 GB  │ 0%   │ 2h 13m     │   │
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

### ZAP Configuration

| Dashboard Field | Type | Default | Source (`scan-config.yaml` path) |
|----------------|------|---------|----------------------------------|
| Scan Type | Dropdown: `full`, `baseline`, `api` | `full` | env `ZAP_SCAN_TYPE` |
| AJAX Spider Enabled | Toggle | `true` | `zap.ajax_spider.enabled` |
| AJAX Spider Max Duration (min) | Number | `60` | `zap.ajax_spider.max_duration_minutes` |
| AJAX Spider Max Depth | Number | `10` | `zap.ajax_spider.max_crawl_depth` |
| AJAX Spider Browser Count | Number | `3` | `zap.ajax_spider.number_of_browsers` |
| Traditional Spider Enabled | Toggle | `true` | `zap.spider.enabled` |
| Active Scan Enabled | Toggle | `true` | `zap.active_scan.enabled` |
| Active Scan Max Duration (min) | Number | `90` | `zap.active_scan.max_scan_duration_minutes` |
| Active Scan Threads/Host | Number | `5` | `zap.active_scan.threads_per_host` |
| Default Attack Strength | Dropdown: `LOW`, `MEDIUM`, `HIGH`, `INSANE` | `HIGH` | `zap.active_scan` (via scan-policy) |
| Default Alert Threshold | Dropdown: `OFF`, `LOW`, `MEDIUM`, `HIGH` | `MEDIUM` | `zap.active_scan` (via scan-policy) |

### Falco Configuration

| Dashboard Field | Type | Default | Source |
|----------------|------|---------|--------|
| Monitored Containers | Read-only list | `nextjs-frontend, rails-backend` | `falco.monitored_containers` |
| Minimum Alert Priority | Dropdown: `emergency`→`debug` | `notice` | `falco-config/falco.yaml` → `priority` |
| Auto-trigger Test Vuln | Toggle | `true` | `falco.test_detection` |

### Prowler Configuration

| Dashboard Field | Type | Default | Source |
|----------------|------|---------|--------|
| AWS Region | Dropdown | `us-east-1` | env `AWS_REGION` |
| Compliance Frameworks | Multi-select checkboxes | `hipaa`, `cis_1.4_aws` | `prowler.compliance` |
| Service Categories | Multi-select checkboxes | All listed | `prowler.categories` |
| Severity Threshold | Dropdown: `critical`→`informational` | `low` | `prowler.severity_threshold` |

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
| `GET` | `/api/scanners/status` | Status of ZAP, Falco, Prowler containers |
| `POST` | `/api/scan/start` | Start a scan with config overrides in request body |
| `GET` | `/api/scan/status` | Current scan phase, progress %, per-tool status |
| `POST` | `/api/scan/stop` | Gracefully stop the running scan |
| `GET` | `/api/scan/history` | List of past scan runs with summary |
| `GET` | `/api/reports/:scan_id` | Get aggregated report for a specific scan |
| `GET` | `/api/reports/:scan_id/download` | Download report bundle (ZIP) |
| `GET` | `/api/reports/:scan_id/:tool` | Get tool-specific report (zap/falco/prowler) |
| `GET` | `/api/containers` | Docker stats for all POC containers |
| `GET` | `/api/containers/:name/logs` | Tail logs for a specific container |
| `POST` | `/api/containers/:name/restart` | Restart a specific container |
| `GET` | `/api/config/defaults` | Return current `scan-config.yaml` as JSON |

### WebSocket Endpoints

| Path | Description |
|------|-------------|
| `ws://runner:8000/ws/scan/progress` | Real-time scan progress updates (pushed every 3s) |
| `ws://runner:8000/ws/falco/events` | Live Falco event stream |
| `ws://runner:8000/ws/runner/logs` | Live runner log stream |

### Example: `POST /api/scan/start` Request Body

```json
{
  "tools": ["zap", "falco", "prowler"],
  "zap": {
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
  "falco": {
    "min_priority": "notice",
    "auto_trigger_test_vuln": true
  },
  "prowler": {
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

# Part 2 — Custom ZAP Docker Image Specification

## 2.1 Design Philosophy

The current setup uses the stock `ghcr.io/zaproxy/zaproxy:stable` image and drives it via ZAP's REST API from `runner.py`. This works but has downsides:

1. The scan logic is split across `runner.py` (API calls) and `scan-policy.yaml` / `scan-config.yaml` (settings).
2. The stock image has no awareness of our scan policy, context, or automation plan.
3. If the ZAP container restarts, all configuration must be re-applied via API.

**The custom image solves this** by:

```
┌───────────────────────────────────────────────────────────────┐
│              Custom ZAP Image                                  │
│  ghcr.io/hipaackr/zap-hipaa-scanner:poc                       │
│                                                                │
│  Base: ghcr.io/zaproxy/zaproxy:stable                         │
│                                                                │
│  Baked in:                                                     │
│  ├── /zap/wrk/plans/hipaa-full-scan.yaml  (Automation Plan)   │
│  ├── /zap/wrk/plans/hipaa-baseline.yaml   (Baseline Plan)     │
│  ├── /zap/wrk/plans/hipaa-api-scan.yaml   (API-only Plan)     │
│  ├── /zap/wrk/policies/hipaa-policy.yaml  (Scan Policy)       │
│  ├── /zap/wrk/contexts/hipaa-context      (Scan Context)      │
│  └── /zap/wrk/entrypoint.sh              (Control Script)     │
│                                                                │
│  Volumes (runtime):                                            │
│  └── /zap/reports  → mounted for report collection             │
│                                                                │
│  Ports:                                                        │
│  └── 8080  → ZAP API (daemon mode)                             │
│                                                                │
│  Env vars (runtime overrides):                                 │
│  ├── ZAP_TARGET_FRONTEND                                       │
│  ├── ZAP_TARGET_BACKEND                                        │
│  ├── ZAP_SCAN_TYPE (full|baseline|api)                         │
│  ├── ZAP_AJAX_SPIDER_DURATION                                  │
│  ├── ZAP_ACTIVE_SCAN_DURATION                                  │
│  └── ZAP_JAVA_XMX                                              │
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
│  Layer 1: Base ZAP Image                    │  ghcr.io/zaproxy/zaproxy:stable
└─────────────────────────────────────────────┘
```

### Add-ons to Pre-install

These ZAP add-ons must be installed into the image at build time to avoid runtime download delays:

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

## 2.3 ZAP Automation Framework Plan File

This is the core of the custom image. Instead of the current approach where `runner.py` makes dozens of individual ZAP API calls, the Automation Framework defines the entire scan pipeline in a single YAML file that ZAP executes natively.

### `hipaa-full-scan.yaml` — Full Scan Plan

```yaml
---
env:
  contexts:
    - name: "HIPAA-POC"
      urls:
        - "${ZAP_TARGET_FRONTEND:-http://nextjs-frontend:3000}"
        - "${ZAP_TARGET_BACKEND:-http://rails-backend:8080}"
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
      url: "${ZAP_TARGET_FRONTEND:-http://nextjs-frontend:3000}"
      maxDuration: "${ZAP_AJAX_SPIDER_DURATION:-60}"
      maxCrawlDepth: "${ZAP_AJAX_SPIDER_DEPTH:-10}"
      numberOfBrowsers: "${ZAP_AJAX_SPIDER_BROWSERS:-3}"
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
      url: "${ZAP_TARGET_FRONTEND:-http://nextjs-frontend:3000}"
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
      url: "${ZAP_TARGET_BACKEND:-http://rails-backend:8080}"
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
      maxScanDurationInMins: "${ZAP_ACTIVE_SCAN_DURATION:-90}"
      maxRuleDurationInMins: 5
      threadPerHost: "${ZAP_ACTIVE_SCAN_THREADS:-5}"
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
      maxScanDurationInMins: "${ZAP_ACTIVE_SCAN_DURATION:-90}"
      maxRuleDurationInMins: 5
      threadPerHost: "${ZAP_ACTIVE_SCAN_THREADS:-5}"

  # ── Step 8: Generate Reports ──
  - type: report
    parameters:
      template: "traditional-json"
      reportDir: "/zap/reports"
      reportFile: "zap_report.json"
    risks:
      - high
      - medium
      - low
      - info

  - type: report
    parameters:
      template: "traditional-html"
      reportDir: "/zap/reports"
      reportFile: "zap_report.html"
    risks:
      - high
      - medium
      - low
      - info

  - type: report
    parameters:
      template: "traditional-xml"
      reportDir: "/zap/reports"
      reportFile: "zap_report.xml"
    risks:
      - high
      - medium
      - low
      - info

  - type: report
    parameters:
      template: "traditional-md"
      reportDir: "/zap/reports"
      reportFile: "zap_report.md"
    risks:
      - high
      - medium
      - low
      - info
```

### `hipaa-baseline.yaml` — Baseline Scan Plan (Passive Only)

This plan is identical to the full scan but **omits the `activeScan` job**. It runs only the spiders and passive scan. This is significantly faster (~15–20 minutes) and safe for production-like environments.

### `hipaa-api-scan.yaml` — API-Only Scan Plan

This plan targets only the Rails backend (`ZAP_TARGET_BACKEND`). It skips the AJAX Spider entirely (since the backend has no SPA) and runs a traditional spider + active scan against the API endpoints.

> **Note:** The Automation Framework supports environment variable substitution (`${VAR:-default}`) natively. The entrypoint script sets these variables before launching ZAP. This is how the dashboard's runtime overrides (passed through the Security Runner) flow into the scan plan without modifying the YAML files on disk.

## 2.4 Dockerfile Specification

```dockerfile
# =============================================================================
# Custom ZAP Docker Image for HIPAA Security Runner POC
# =============================================================================
# This image wraps the official ZAP image with pre-baked:
#   - Automation Framework plan files (full, baseline, api-only)
#   - Scan policy with HIPAA-relevant scanner rules
#   - Scan context with target URL patterns
#   - Entrypoint script for orchestrator control
#
# Build:
#   docker build -t ghcr.io/hipaackr/zap-hipaa-scanner:poc .
#
# Run (standalone test):
#   docker run -d -p 8090:8080 \
#     -e ZAP_TARGET_FRONTEND=http://nextjs-frontend:3000 \
#     -e ZAP_TARGET_BACKEND=http://rails-backend:8080 \
#     ghcr.io/hipaackr/zap-hipaa-scanner:poc
# =============================================================================

FROM ghcr.io/zaproxy/zaproxy:stable

LABEL maintainer="HIPAA Security Team"
LABEL description="Custom ZAP image for HIPAA Security Runner POC"
LABEL version="1.0"
LABEL org.opencontainers.image.source="https://github.com/HIPAACKR/dynamic-analysis"

# ── Install required add-ons at build time ──
# This avoids runtime download delays and ensures reproducible scans.
RUN /zap/zap.sh -cmd \
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
RUN mkdir -p /zap/wrk/plans \
             /zap/wrk/policies \
             /zap/wrk/contexts \
             /zap/reports

# ── Copy Automation Framework plan files ──
COPY plans/hipaa-full-scan.yaml    /zap/wrk/plans/
COPY plans/hipaa-baseline.yaml     /zap/wrk/plans/
COPY plans/hipaa-api-scan.yaml     /zap/wrk/plans/

# ── Copy scan policy ──
COPY policies/hipaa-policy.yaml    /zap/wrk/policies/

# ── Copy entrypoint script ──
COPY entrypoint.sh                 /zap/wrk/entrypoint.sh
RUN chmod +x /zap/wrk/entrypoint.sh

# ── Default environment variables ──
# These can be overridden at runtime by the orchestrator.
ENV ZAP_PORT=8080
ENV ZAP_TARGET_FRONTEND="http://nextjs-frontend:3000"
ENV ZAP_TARGET_BACKEND="http://rails-backend:8080"
ENV ZAP_SCAN_TYPE="full"
ENV ZAP_AJAX_SPIDER_DURATION="60"
ENV ZAP_AJAX_SPIDER_DEPTH="10"
ENV ZAP_AJAX_SPIDER_BROWSERS="3"
ENV ZAP_ACTIVE_SCAN_DURATION="90"
ENV ZAP_ACTIVE_SCAN_THREADS="5"
ENV ZAP_JAVA_XMX="8g"

# ── Expose ZAP API port ──
EXPOSE 8080

# ── Health check ──
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=5 \
    CMD curl -sf http://localhost:${ZAP_PORT}/JSON/core/view/version/ || exit 1

# ── Entrypoint ──
# Starts ZAP in daemon mode with the API enabled.
# The Automation Framework plan is NOT auto-started — the orchestrator
# triggers it via the ZAP API when ready.
ENTRYPOINT ["/zap/wrk/entrypoint.sh"]
```

### Resulting Image File Structure

```
/zap/
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
│   ├── zap_report.json              ← Generated by automation plan
│   ├── zap_report.html
│   ├── zap_report.xml
│   └── zap_report.md
└── (stock ZAP files)
```

## 2.5 Entrypoint & Control Script

The entrypoint script starts ZAP in daemon mode with the API enabled. It does **not** start the automation plan automatically — that is triggered by the Security Runner orchestrator.

### `entrypoint.sh`

```bash
#!/bin/bash
# =============================================================================
# Custom ZAP Entrypoint for HIPAA Security Runner POC
# =============================================================================
# This script:
#   1. Resolves the correct automation plan based on ZAP_SCAN_TYPE
#   2. Exports all environment variables for plan substitution
#   3. Starts ZAP in daemon mode with the API enabled
#   4. Waits for ZAP to be ready
#   5. Keeps the container running (ZAP daemon is the foreground process)
#
# The orchestrator (Security Runner) triggers the scan plan via:
#   POST /JSON/automation/action/runPlan/
#     ?filePath=/zap/wrk/plans/<selected-plan>.yaml
# =============================================================================

set -e

echo "╔═══════════════════════════════════════════════════════════════╗"
echo "║  HIPAA ZAP Scanner — Custom Image                            ║"
echo "╠═══════════════════════════════════════════════════════════════╣"
echo "║  Scan Type:    ${ZAP_SCAN_TYPE:-full}                        ║"
echo "║  Frontend:     ${ZAP_TARGET_FRONTEND}                        ║"
echo "║  Backend:      ${ZAP_TARGET_BACKEND}                         ║"
echo "║  Java Heap:    ${ZAP_JAVA_XMX:-8g}                           ║"
echo "║  API Port:     ${ZAP_PORT:-8080}                              ║"
echo "╚═══════════════════════════════════════════════════════════════╝"

# ── Resolve plan file path ──
case "${ZAP_SCAN_TYPE:-full}" in
    full)
        PLAN_FILE="/zap/wrk/plans/hipaa-full-scan.yaml"
        ;;
    baseline)
        PLAN_FILE="/zap/wrk/plans/hipaa-baseline.yaml"
        ;;
    api)
        PLAN_FILE="/zap/wrk/plans/hipaa-api-scan.yaml"
        ;;
    *)
        echo "ERROR: Unknown ZAP_SCAN_TYPE: ${ZAP_SCAN_TYPE}"
        echo "Valid values: full, baseline, api"
        exit 1
        ;;
esac

echo "[entrypoint] Selected plan: ${PLAN_FILE}"
echo "[entrypoint] Plan file exists: $(test -f ${PLAN_FILE} && echo YES || echo NO)"

# ── Export plan file path for orchestrator discovery ──
export ZAP_PLAN_FILE="${PLAN_FILE}"

# ── Start ZAP in daemon mode ──
echo "[entrypoint] Starting ZAP daemon..."
exec /zap/zap.sh -daemon \
    -host 0.0.0.0 \
    -port "${ZAP_PORT:-8080}" \
    -config api.disablekey=true \
    -config api.addrs.addr.name=.* \
    -config api.addrs.addr.regex=true \
    -config ajaxSpider.maxCrawlDepth="${ZAP_AJAX_SPIDER_DEPTH:-10}" \
    -config ajaxSpider.maxDuration="${ZAP_AJAX_SPIDER_DURATION:-60}" \
    -config ajaxSpider.numberOfBrowsers="${ZAP_AJAX_SPIDER_BROWSERS:-3}" \
    -config connection.timeoutInSecs=120
```

> **Why not auto-start the plan?** Because the orchestrator needs to:
> 1. Wait for ZAP to fully initialize.
> 2. Optionally apply runtime overrides (from the dashboard config panel).
> 3. Control the timing (e.g., ensure targets are healthy first).
> 4. Monitor progress and report it to the dashboard.

## 2.6 Environment Variable Interface

These are all the environment variables the custom ZAP image accepts. The orchestrator sets them at container start time (or via Docker Compose).

| Variable | Default | Description |
|----------|---------|-------------|
| `ZAP_PORT` | `8080` | Port for ZAP API |
| `ZAP_TARGET_FRONTEND` | `http://nextjs-frontend:3000` | Frontend URL to scan |
| `ZAP_TARGET_BACKEND` | `http://rails-backend:8080` | Backend URL to scan |
| `ZAP_SCAN_TYPE` | `full` | Plan selection: `full`, `baseline`, `api` |
| `ZAP_AJAX_SPIDER_DURATION` | `60` | Max AJAX Spider duration (minutes) |
| `ZAP_AJAX_SPIDER_DEPTH` | `10` | Max AJAX Spider crawl depth |
| `ZAP_AJAX_SPIDER_BROWSERS` | `3` | Number of headless browsers |
| `ZAP_ACTIVE_SCAN_DURATION` | `90` | Max active scan duration (minutes) |
| `ZAP_ACTIVE_SCAN_THREADS` | `5` | Threads per host for active scan |
| `ZAP_JAVA_XMX` | `8g` | Java heap max (passed to `-Xmx`) |
| `JAVA_OPTS` | `-Xmx${ZAP_JAVA_XMX}` | Full Java options string |

## 2.7 Volume Mounts & Ports

| Mount | Container Path | Purpose |
|-------|---------------|---------|
| `zap-reports` volume | `/zap/reports` | Report output (shared with security-runner) |

| Port | Container Port | Host Port | Purpose |
|------|---------------|-----------|---------|
| ZAP API | `8080` | `8090` | ZAP REST API |

## 2.8 Updated Docker Compose Service Definition

This replaces the current `zap-scanner` service in `docker-compose.security-poc.yml`:

```yaml
  zap-scanner:
    image: ghcr.io/hipaackr/zap-hipaa-scanner:poc
    container_name: zap-scanner
    networks:
      poc-net:
        ipv4_address: 172.20.0.100
    ports:
      - "8090:8080"
    volumes:
      - zap-reports:/zap/reports
    environment:
      ZAP_PORT: 8080
      ZAP_TARGET_FRONTEND: ${TARGET_FRONTEND_URL:-http://nextjs-frontend:3000}
      ZAP_TARGET_BACKEND: ${TARGET_BACKEND_URL:-http://rails-backend:8080}
      ZAP_SCAN_TYPE: ${ZAP_SCAN_TYPE:-full}
      ZAP_AJAX_SPIDER_DURATION: ${ZAP_AJAX_SPIDER_DURATION:-60}
      ZAP_AJAX_SPIDER_DEPTH: ${ZAP_AJAX_SPIDER_DEPTH:-10}
      ZAP_AJAX_SPIDER_BROWSERS: ${ZAP_AJAX_SPIDER_BROWSERS:-3}
      ZAP_ACTIVE_SCAN_DURATION: ${ZAP_ACTIVE_SCAN_DURATION:-90}
      ZAP_ACTIVE_SCAN_THREADS: ${ZAP_ACTIVE_SCAN_THREADS:-5}
      ZAP_JAVA_XMX: ${ZAP_JAVA_XMX:-8g}
      JAVA_OPTS: "-Xmx${ZAP_JAVA_XMX:-8g}"
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

**Key change:** The `image` field now points to the custom image. The `command` field is removed because the entrypoint handles startup. The `volumes` no longer mounts `./zap-config` as read-only because the config is baked into the image.

## 2.9 Orchestrator Integration — How the Dashboard Controls ZAP

### Flow: Dashboard → Security Runner → ZAP

```
Dashboard                    Security Runner               ZAP Container
   │                              │                             │
   │  POST /api/scan/start        │                             │
   │  { tools: ["zap"],           │                             │
   │    zap: { scan_type: "full", │                             │
   │           ajax_spider: {...}  │                             │
   │    }}                        │                             │
   │ ───────────────────────────> │                             │
   │                              │                             │
   │                              │  1. Health check ZAP API    │
   │                              │ ──────────────────────────> │
   │                              │  GET /JSON/core/view/ver/   │
   │                              │ <────────── 200 OK ──────── │
   │                              │                             │
   │                              │  2. Trigger automation plan │
   │                              │ ──────────────────────────> │
   │                              │  GET /JSON/automation/      │
   │                              │    action/runPlan/          │
   │                              │    ?filePath=/zap/wrk/      │
   │                              │     plans/hipaa-full-scan   │
   │                              │     .yaml                   │
   │                              │ <────────── 200 OK ──────── │
   │                              │                             │
   │  WS scan/progress            │  3. Poll plan status        │
   │  { zap: { phase: "ajax",     │ ──────────────────────────> │
   │    progress: 45% }}          │  GET /JSON/automation/      │
   │ <─────────────── (push) ──── │    view/planProgress/       │
   │                              │ <──── { progress: 45% } ─── │
   │                              │                             │
   │                              │  ... (repeat polling) ...   │
   │                              │                             │
   │  WS scan/progress            │  4. Plan completes          │
   │  { zap: { phase: "done",     │ ──────────────────────────> │
   │    progress: 100%,           │  GET /JSON/automation/      │
   │    alerts: 12 }}             │    view/planProgress/       │
   │ <─────────────── (push) ──── │ <──── { progress: 100% } ── │
   │                              │                             │
   │                              │  5. Collect reports         │
   │                              │  (from /zap/reports volume) │
   │                              │                             │
```

### Key ZAP Automation Framework API Endpoints

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
cd security-runner-poc/zap-custom/
docker build -t ghcr.io/hipaackr/zap-hipaa-scanner:poc .
```

### Tag for Distribution

```bash
# GHCR
docker tag ghcr.io/hipaackr/zap-hipaa-scanner:poc \
           ghcr.io/hipaackr/zap-hipaa-scanner:1.0
docker tag ghcr.io/hipaackr/zap-hipaa-scanner:poc \
           ghcr.io/hipaackr/zap-hipaa-scanner:latest

# Semantic versioning
docker tag ghcr.io/hipaackr/zap-hipaa-scanner:poc \
           ghcr.io/hipaackr/zap-hipaa-scanner:1.0.0-poc
```

### Push to GHCR

```bash
# Authenticate
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Push all tags
docker push ghcr.io/hipaackr/zap-hipaa-scanner:poc
docker push ghcr.io/hipaackr/zap-hipaa-scanner:1.0
docker push ghcr.io/hipaackr/zap-hipaa-scanner:latest
```

### Export as Tar File (Offline / Air-gapped Deployment)

```bash
# Save as tar (compressed)
docker save ghcr.io/hipaackr/zap-hipaa-scanner:poc | gzip > zap-hipaa-scanner-poc.tar.gz

# Load on target server
gunzip -c zap-hipaa-scanner-poc.tar.gz | docker load
# Output: Loaded image: ghcr.io/hipaackr/zap-hipaa-scanner:poc
```

### Distribution Artifacts

| Artifact | Location | Use Case |
|----------|----------|----------|
| GHCR Image | `ghcr.io/hipaackr/zap-hipaa-scanner:poc` | Internet-connected servers |
| GHCR Image (versioned) | `ghcr.io/hipaackr/zap-hipaa-scanner:1.0` | Pinned version |
| Tar file | `zap-hipaa-scanner-poc.tar.gz` | Air-gapped / offline servers |
| Source (Dockerfile + plans) | `security-runner-poc/zap-custom/` | Rebuild from source |

## 2.11 Changes Applied to Existing Codebase

### Summary of Changes

The migration from the API-driven approach to the Automation Framework approach has been completed. The Security Runner orchestrator and the project structure have been updated as described below.

### 2.11.1 New Directory: `zap-custom/` ✅

Created at `security-runner-poc/zap-custom/` to hold the custom image build context:

```
zap-custom/
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
- `_generate_zap_reports()` — Removed (reports are generated by the plan)

**Added methods:**
- `_trigger_automation_plan(plan_file: str)` — Sends `GET /JSON/automation/action/runPlan/` to ZAP.
- `_poll_plan_progress()` — Polls `GET /JSON/automation/view/planProgress/` and pushes updates via WebSocket.
- `_resolve_plan_file(scan_type: str)` — Maps `full`/`baseline`/`api` to the plan file path.

**Rewritten `_phase_dast_scan()` logic:**

```
1. Resolve plan file from ZAP_SCAN_TYPE
2. Trigger plan: GET /JSON/automation/action/runPlan/?filePath=<plan>
3. Poll loop:
   a. GET /JSON/automation/view/planProgress/
   b. Push progress to WebSocket
   c. Sleep 5 seconds
   d. Break when progress = 100 or timeout
4. GET /JSON/core/view/alerts/ → store results
5. Verify reports exist in /zap/reports volume mount
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
| `zap-scanner` | Image changed to `ghcr.io/hipaackr/zap-hipaa-scanner:poc`, `command` removed, `./zap-config` volume mount removed |
| `falco-agent` | Image changed to `ghcr.io/hipaackr/falco-hipaa-agent:poc`, config volume mounts removed, `command` removed |
| `prowler-scanner` | Image changed to `ghcr.io/hipaackr/prowler-hipaa-scanner:poc`, config volume mounts removed, only `credentials` file mounted at runtime |
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

# Part 3 — Custom Falco & Prowler Image Notes (Companion Images)

The same "bake configuration into the image" approach has been applied to Falco and Prowler. The build contexts are implemented at `falco-custom/` and `prowler-custom/`.

## 3.1 Custom Falco Image

### Image Name: `ghcr.io/hipaackr/falco-hipaa-agent:poc`

### What Gets Baked In

| File | Container Path | Source |
|------|---------------|--------|
| `falco.yaml` | `/etc/falco/falco.yaml` | `falco-config/falco.yaml` |
| `hipaa-poc-rules.yaml` | `/etc/falco/rules.d/hipaa-poc-rules.yaml` | `falco-config/rules.d/hipaa-poc-rules.yaml` |

### Dockerfile Outline

```dockerfile
FROM falcosecurity/falco:latest

LABEL maintainer="HIPAA Security Team"
LABEL description="Custom Falco image for HIPAA Security Runner POC"

# Bake in custom configuration
COPY falco.yaml                /etc/falco/falco.yaml
COPY rules.d/hipaa-poc-rules.yaml /etc/falco/rules.d/hipaa-poc-rules.yaml

# Default environment
ENV FALCO_BPF_PROBE=""

# Entrypoint (same as stock, but ensures our config is used)
CMD ["/usr/bin/falco", \
     "-o", "json_output=true", \
     "-o", "json_include_output_property=true", \
     "-o", "file_output.enabled=true", \
     "-o", "file_output.filename=/var/log/falco/falco_events.json", \
     "-o", "stdout_output.enabled=true"]
```

### Build Context Directory

```
falco-custom/
├── Dockerfile
├── falco.yaml
└── rules.d/
    └── hipaa-poc-rules.yaml
```

### Docker Compose Update

```yaml
  falco-agent:
    image: ghcr.io/hipaackr/falco-hipaa-agent:poc
    container_name: falco-agent
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
      - falco-logs:/var/log/falco
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M
    restart: unless-stopped
```

**Key change:** No more `./falco-config/falco.yaml:/etc/falco/falco.yaml:ro` volume mount — config is baked in.

---

## 3.2 Custom Prowler Image

### Image Name: `ghcr.io/hipaackr/prowler-hipaa-scanner:poc`

### What Gets Baked In

| File | Container Path | Source |
|------|---------------|--------|
| `prowler-config.yaml` | `/prowler/config/prowler-config.yaml` | `prowler-config/prowler-config.yaml` |
| AWS `config` file | `/root/.aws/config` | `aws-credentials/config` |

> **Note:** AWS credentials (`credentials` file) should **never** be baked into an image. They must be provided at runtime via volume mount or environment variables.

### Dockerfile Outline

```dockerfile
FROM prowler/prowler:latest

LABEL maintainer="HIPAA Security Team"
LABEL description="Custom Prowler image for HIPAA Security Runner POC"

# Bake in Prowler configuration
COPY prowler-config.yaml /prowler/config/prowler-config.yaml

# Bake in AWS config (region, role profile — NOT credentials)
COPY aws-config /root/.aws/config

# Create output directory
RUN mkdir -p /prowler/output

# Default environment
ENV AWS_REGION=us-east-1

# Idle entrypoint — orchestrator triggers scans via docker exec
ENTRYPOINT ["tail", "-f", "/dev/null"]
```

### Build Context Directory

```
prowler-custom/
├── Dockerfile
├── prowler-config.yaml
└── aws-config               ← just the [default] region/role config, NOT credentials
```

### Docker Compose Update

```yaml
  prowler-scanner:
    image: ghcr.io/hipaackr/prowler-hipaa-scanner:poc
    container_name: prowler-scanner
    networks:
      poc-net:
        ipv4_address: 172.20.0.102
    volumes:
      - prowler-reports:/prowler/output
      - ./aws-credentials/credentials:/root/.aws/credentials:ro  # Runtime only
    environment:
      AWS_REGION: ${AWS_REGION:-us-east-1}
      AWS_ROLE_ARN: ${PROWLER_ROLE_ARN}
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 1G
    restart: unless-stopped
```

**Key change:** Only the `credentials` file is mounted at runtime (never baked in). The config and Prowler settings are baked into the image.

---

## 3.3 Combined Distribution Matrix

| Image | GHCR Path | Tar File Name | Size (est.) |
|-------|-----------|---------------|-------------|
| Custom ZAP | `ghcr.io/hipaackr/zap-hipaa-scanner:poc` | `zap-hipaa-scanner-poc.tar.gz` | ~1.5 GB |
| Custom Falco | `ghcr.io/hipaackr/falco-hipaa-agent:poc` | `falco-hipaa-agent-poc.tar.gz` | ~200 MB |
| Custom Prowler | `ghcr.io/hipaackr/prowler-hipaa-scanner:poc` | `prowler-hipaa-scanner-poc.tar.gz` | ~500 MB |
| Security Runner | `ghcr.io/hipaackr/security-runner:poc` | `security-runner-poc.tar.gz` | ~300 MB |

### Bulk Export / Import Script

```bash
#!/bin/bash
# export-all-images.sh — Save all custom images as tar files

IMAGES=(
  "ghcr.io/hipaackr/zap-hipaa-scanner:poc"
  "ghcr.io/hipaackr/falco-hipaa-agent:poc"
  "ghcr.io/hipaackr/prowler-hipaa-scanner:poc"
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
