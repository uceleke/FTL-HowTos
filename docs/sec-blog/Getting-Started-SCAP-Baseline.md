# Getting Started with SCAP: How to Establish a Security Baseline for a Windows System

**Author:** Jesse Esposo
**Published:** May 13, 2026
**Category:** Security+ CEU / Cybersecurity

---

## Introduction

Before hardening a Windows system, you need to understand its current security posture. A security baseline gives you a measurable starting point by identifying configuration gaps, policy weaknesses, and areas of risk — before you start making changes. Rather than guessing whether a system is secure, administrators can use standardized benchmarks and automated scanning tools to compare the current configuration against known security requirements.

One of the most common methods for doing this is through the **Security Content Automation Protocol**, better known as **SCAP**. SCAP is a standardized framework for checking system configurations against published security benchmarks in a machine-readable, repeatable way. In enterprise and government environments, SCAP is typically paired with **DISA Security Technical Implementation Guides (STIGs)** to evaluate whether systems are configured in accordance with approved hardening guidance.

This post walks through how to get started with SCAP, how to obtain the scanner and benchmark content you need, and how to use the results to build a defensible, repeatable security baseline process for Windows systems.

---

## What Is SCAP and How Is It Different from Vulnerability Scanning?

SCAP is a collection of open standards maintained by NIST that automates security configuration assessment. A SCAP-compatible scanner reads a machine-readable benchmark and checks each rule against the target system, returning results of **pass**, **fail**, **not applicable**, or **manual review required**.

For Windows systems, SCAP can evaluate settings such as:

- Password and account lockout policy
- Audit policy and event log configuration
- User rights assignments
- Windows Firewall configuration
- Registry-based security controls
- Service configuration and other OS hardening settings

**SCAP is not a replacement for vulnerability scanning.** They serve different purposes and answer different questions:

- A **vulnerability scanner** (such as Nessus or OpenVAS) asks: *"What known weaknesses — missing patches, CVEs, exposed services — exist on this system?"*
- A **SCAP scan** asks: *"Does this system meet the required configuration standard?"*

Both are valuable. In a DoD or federal environment you will commonly use both as part of a continuous monitoring program.

---

## Why Establish a Baseline?

A baseline is the starting point for measurable security improvement. Without one, it is difficult to know what changed, what improved, and what still needs attention.

For example, if you scan a fresh Windows Server and find 150 configuration findings, that initial report *is* the baseline. After applying Group Policy changes, registry settings, or other hardening steps, you scan again and compare. The delta tells you whether your remediation actually worked.

A baseline helps answer:

- What is the current risk posture of this system?
- Which findings are the highest priority?
- Which settings can be remediated quickly versus which require testing or an exception?
- Did the remediation actually fix the issue?

This approach directly supports risk management, change control, and continuous monitoring requirements — all relevant to RMF-based programs and eMASS-tracked systems.

---

## Step 1: Identify and Document the Target System

Start with a clearly defined target. Before scanning, document:

- Hostname and IP address
- Operating system version and build
- System role (domain controller, member server, workstation, etc.)
- Domain membership
- Business function
- Environment (production, test, or lab)

This context matters. Not every STIG finding applies equally to every system role. A domain controller, application server, workstation, and database server will have different hardening considerations — and in some cases, different STIG benchmarks entirely.

---

## Step 2: Download a SCAP Scanner

To run a SCAP scan, you need a compatible scanner. In DoD and federal environments, the most commonly used tool is the **SCAP Compliance Checker (SCC)**, developed and maintained by NIWC Atlantic. SCC supports Windows, Linux, and other platforms, and is designed to work directly with DISA STIG SCAP content.

**Where to get it:**

1. Go to the DoD Cyber Exchange public site at [https://public.cyber.mil/tools/](https://public.cyber.mil/tools/)
2. Search for **SCAP Compliance Checker**
3. Download the latest Windows version
4. Install or extract on an approved administrative workstation
5. Review the included documentation before running your first scan

In an air-gapped environment, download on an internet-connected system, transfer via approved media, and validate the file hash before introducing it into the environment.

---

## Step 3: Download the Correct Benchmark Content

The scanner alone is not enough — you also need the **SCAP benchmark content** that matches your target system. For DISA STIG assessments, benchmark content is available from the DoD Cyber Exchange STIGs section.

**Where to get it:**

1. Go to [https://public.cyber.mil/stigs/](https://public.cyber.mil/stigs/)
2. Search for the operating system or product you are assessing
3. Download the applicable STIG SCAP benchmark (e.g., Windows Server 2022 STIG, Windows 11 STIG)
4. Confirm the benchmark version matches your OS version as closely as possible
5. Store the benchmark files where SCC can reference them

Using the wrong benchmark produces inaccurate results. Scanning a Windows Server 2022 system with Windows Server 2019 content will generate misleading findings.

**A note on CIS Benchmarks:** If your organization does not operate under DISA STIG requirements — for example, if you are in a commercial environment or building a baseline for a client outside the federal space — the **CIS Benchmarks** from the Center for Internet Security ([https://www.cisecurity.org/cis-benchmarks](https://www.cisecurity.org/cis-benchmarks)) are a solid alternative. CIS Benchmarks are widely used, well-documented, and available for a broad range of Windows versions and roles.

---

## Step 4: Run the Initial Baseline Scan

With the scanner installed and benchmark content in place, run the first scan. The goal here is *assessment only* — do not start changing settings until you have reviewed the results.

Typical process in SCC:

1. Launch SCC as an administrator
2. Select the correct benchmark content for your target OS
3. Choose the local system or specify a remote target
4. Run the scan and wait for it to complete
5. Export results in your preferred format (HTML for readability, XML or checklist for downstream use)

Save the report with a clear naming convention so you can track changes over time. For example:

```
HOSTNAME_WinSrv2022_STIG_Baseline_2026-05-13.html
```

This first report becomes your official baseline.

---

## Step 5: Review Findings and Prioritize Remediation

After the scan, review findings in the context of **severity** and **operational impact**. DISA STIGs categorize findings as:

- **CAT I** — Highest severity. These represent the most significant risk and should be addressed first.
- **CAT II** — Medium severity. These typically make up the bulk of findings and remediation work.
- **CAT III** — Lower severity. Still worth addressing when practical, but generally lower urgency.

A straightforward prioritization approach:

1. Address all CAT I findings first
2. Target high-impact CAT II findings that are straightforward to fix
3. Group related findings (e.g., all audit policy findings, all password policy findings) to apply changes efficiently
4. Test changes in a lab or non-production system before applying to production
5. Apply changes via Group Policy where possible — local policy is acceptable for standalone systems and lab work, but enterprise environments should use centralized management
6. Document any finding that cannot be immediately applied

Do not blindly apply every setting without testing. Some STIG controls can affect authentication flows, remote access, application compatibility, service account behavior, and administrative workflows. Test first.

---

## Step 6: Document Exceptions and Compensating Controls

Not every finding can be remediated immediately. Some settings may conflict with a business requirement, vendor support constraint, legacy application dependency, or operational need.

When a finding cannot be fixed, document it formally. At minimum, capture:

- Finding ID and severity
- Affected system
- Reason the control cannot be applied
- Risk explanation
- Compensating control (if applicable)
- Planned remediation date or POA&M entry
- System owner or responsible party

Accepting risk is not the same as ignoring risk. A documented exception demonstrates that the organization reviewed the issue and made an informed decision. In RMF-based programs, this is typically captured in a Plan of Action and Milestones (POA&M) tracked in eMASS.

---

## Step 7: Re-Scan and Measure Improvement

After remediation, run the SCAP scan again using the same benchmark content and compare results against the original baseline. The goal is measurable improvement — for example, reducing from 150 failed findings to 40.

This comparison validates that remediation worked, provides documentation for audits and compliance reviews, and becomes the new baseline for future comparisons.

---

## Step 8: Maintain the Baseline Over Time

A security baseline is not a one-time activity. Systems change — patches are applied, software gets installed, Group Policy is updated, roles change. Configuration drift is real and common.

A practical ongoing scanning schedule might include:

- Before production deployment
- After major configuration changes or OS upgrades
- On a recurring schedule (monthly, quarterly, or per your organization's continuous monitoring policy)
- When DISA or CIS releases updated benchmark content

Keeping baselines current ensures systems remain aligned with security requirements and that drift is detected before it becomes a compliance or audit issue.

---

## Conclusion

SCAP gives administrators a structured, repeatable way to assess a Windows system's security posture against a defined configuration standard. The process — scanner, benchmark, baseline scan, findings review, remediation, re-scan, documentation — is straightforward to implement and scales from a single workstation to an enterprise environment.

The scan report is not the end goal. Security improves when findings are reviewed, prioritized, tested, remediated, documented, and validated over time. For organizations operating under RMF, this process feeds directly into continuous monitoring, POA&M management, and ATO maintenance.

For Security+ professionals, understanding SCAP reinforces core exam domains: hardening, vulnerability management, risk management, compliance, and continuous monitoring. More practically, it is a skill that translates directly into real work in enterprise and government environments.

---

## References

- NIST Security Content Automation Protocol: [https://csrc.nist.gov/projects/security-content-automation-protocol](https://csrc.nist.gov/projects/security-content-automation-protocol)
- DoD Cyber Exchange STIGs: [https://public.cyber.mil/stigs/](https://public.cyber.mil/stigs/)
- DoD Cyber Exchange Tools: [https://public.cyber.mil/tools/](https://public.cyber.mil/tools/)
- SCAP Compliance Checker (NIWC Atlantic): [https://www.niwcatlantic.navy.mil/Technology/SCAP/](https://www.niwcatlantic.navy.mil/Technology/SCAP/)
- CIS Benchmarks: [https://www.cisecurity.org/cis-benchmarks](https://www.cisecurity.org/cis-benchmarks)
