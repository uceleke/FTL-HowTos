# Getting Started with SCAP: How to Establish a Security Baseline for a Windows System

**Author:** Jesse Esposo
**Published:** May 13, 2026
**Category:** Security+ CEU / Cybersecurity

---

## Introduction

Before hardening any information system, you need to understand its current security posture. A security baseline gives you a measurable starting point by identifying configuration gaps, policy weaknesses, and areas of risk before you start making changes to harden your system/environment. Rather than guess if a system is secure, administrators can use standardized benchmarks and automated scanning tools to compare the current configuration against known security requirements.

One of the most practical ways to do this is through the **Security Content Automation Protocol**, or **SCAP**. SCAP is a standardized framework for checking system configurations against published benchmarks in a machine-readable, repeatable way. In enterprise and government environments, SCAP is typically paired with **DISA Security Technical Implementation Guides (STIGs)** to evaluate whether systems are configured in accordance with approved hardening guidance.

This post walks through how to get started with SCAP — how to get the scanner, grab the right benchmark content, and use the results to build a repeatable baseline process for Windows systems.

---

## What Is SCAP and How Is It Different from Vulnerability Scanning?

SCAP is a collection of open standards maintained by NIST that automates security configuration assessment. A SCAP-compatible scanner reads a machine-readable benchmark and checks each rule against the target system, returning results of **pass**, **fail**, **not applicable**, or **manual review required**.

For Windows systems, SCAP can evaluate things like:

- Password and account lockout policy
- Audit policy and event log settings
- User rights assignments
- Windows Firewall configuration
- Registry-based security controls
- Service configuration and other OS hardening settings

One thing worth clarifying upfront — **SCAP is not a replacement for vulnerability scanning**. They answer different questions:

- A **vulnerability scanner** like Nessus or OpenVAS asks: *"What known weaknesses — missing patches, CVEs, exposed services — exist on this system?"*
- A **SCAP scan** asks: *"Does this system actually meet the required configuration standard?"*

You need both. In a DoD or federal environment you'll typically run both as part of a continuous monitoring program, and they complement each other well.

---

## Why Bother Establishing a Baseline?

A baseline is the starting point for any meaningful security improvement. Without one, it's hard to know what changed, what got better, and what still needs work.

Say you scan a fresh Windows Server and come back with 150 configuration findings. That report is your baseline. After you apply Group Policy changes, registry settings, or other hardening steps, you scan again and compare. The delta tells you whether your remediation actually did anything.

A solid baseline helps answer questions like:

- What's the actual risk posture of this system right now?
- Which findings are the highest priority?
- Which settings can be fixed quickly versus which need testing or a formal exception?
- Did the changes I made actually fix the issue?

For anyone operating under RMF, this feeds directly into continuous monitoring, POA&M tracking, and keeping your ATO in good standing.

---

## Step 1: Know What You're Scanning

Before you run anything, document the basics on your target system:

- Hostname and IP address
- OS version and build
- System role (domain controller, member server, workstation, etc.)
- Domain membership
- Business function
- Environment (production, test, or lab)

This matters more than it might seem. Not every STIG finding applies equally to every system role. A domain controller, an application server, a workstation, and a database server all have different hardening considerations — and in some cases, different STIG benchmarks entirely. Knowing the role up front saves you from chasing findings that don't actually apply.

---

## Step 2: Get a SCAP Scanner

To run a SCAP scan you need a compatible scanner. In DoD and federal environments, the go-to tool is the **SCAP Compliance Checker (SCC)**, built and maintained by NIWC Atlantic. SCC supports Windows, Linux, and other platforms and is designed to work natively with DISA STIG content.

**Where to get it:**

1. Go to the DoD Cyber Exchange at [https://public.cyber.mil/tools/](https://public.cyber.mil/tools/)
2. Search for **SCAP Compliance Checker**
3. Download the latest Windows version
4. Install or extract it on an approved admin workstation
5. Read through the included docs before your first scan — worth it

If you're in an air-gapped environment, download it on an internet-connected system first, transfer via approved media, validate the file hash, and then bring it in. Don't skip the hash check.

---

## Step 3: Get the Right Benchmark Content

The scanner alone won't do anything useful — you also need the **SCAP benchmark content** that matches your target system. For DISA STIG assessments, this content lives on the DoD Cyber Exchange.

**Where to get it:**

1. Go to [https://public.cyber.mil/stigs/](https://public.cyber.mil/stigs/)
2. Search for the OS or product you're assessing
3. Download the applicable STIG SCAP benchmark (e.g., Windows Server 2022, Windows 11)
4. Make sure the benchmark version matches your OS as closely as possible
5. Put the files somewhere SCC can find them

Using the wrong benchmark will give you misleading results — scanning a Server 2022 system with Server 2019 content is going to cause problems. Double-check before you run anything.

**What about CIS Benchmarks?** If your org doesn't operate under DISA STIG requirements — say you're in a commercial environment or doing work for a client outside the federal space — the **CIS Benchmarks** from the Center for Internet Security ([https://www.cisecurity.org/cis-benchmarks](https://www.cisecurity.org/cis-benchmarks)) are a solid alternative. They're well-documented, widely used, and cover a broad range of Windows versions and roles.

---

## Step 4: Run Your Initial Baseline Scan

With the scanner installed and benchmark content ready, run the first scan. The goal here is assessment only — don't start changing settings until you've actually looked at what came back.

General flow in SCC:

1. Launch SCC as an administrator
2. Select the correct benchmark for your target OS
3. Choose local or remote target depending on your setup
4. Run it and let it finish
5. Export results — HTML is easiest to read, XML or checklist format if you need it downstream

Name your report in a way that makes it easy to track over time. Something like:

```
HOSTNAME_WinSrv2022_STIG_Baseline_2026-05-13.html
```

This first report is your official baseline. Don't touch settings before you've saved it.

---

## Step 5: Review Findings and Build a Remediation Plan

Once the scan is done, go through the findings with severity and operational impact in mind. DISA STIGs break findings into three categories:

- **CAT I** — Highest severity. Biggest risk. Start here.
- **CAT II** — Medium severity. Usually the bulk of your work.
- **CAT III** — Lower severity. Address when practical.

A straightforward way to approach remediation:

1. Hit CAT I findings first
2. Go after high-impact CAT II findings that are easy wins
3. Batch related findings together — fix all audit policy settings at once, all password policy settings at once, etc.
4. Test in a lab or non-production system before applying to production
5. Use Group Policy wherever possible for enterprise-wide changes; local policy works fine for standalone systems and lab work
6. Document anything you can't immediately apply

One thing I'd emphasize — don't blindly apply every setting without testing first. Some STIG controls can break authentication flows, affect remote access, mess with application compatibility, or cause issues with service accounts. Test before you push to prod.

---

## Step 6: Document Exceptions and Compensating Controls

Not everything is going to get fixed right away. Some settings conflict with a business requirement, a vendor support constraint, a legacy app dependency, or some operational reality you can't just override.

When a finding can't be remediated, document it. At minimum you want:

- Finding ID and severity
- Affected system
- Why the control can't be applied
- Risk explanation
- Compensating control if you have one
- Planned remediation date or POA&M entry
- System owner

Accepting risk is not the same as ignoring it. A documented exception shows you reviewed the issue and made a decision based on both security and business requirements. In RMF programs this usually ends up as a POA&M entry tracked in eMASS.

---

## Step 7: Re-Scan and Check Your Work

After remediation, run the scan again with the same benchmark and compare against your original baseline. You want to see measurable improvement — going from 150 failed findings to 40 is a concrete result you can show.

This comparison validates that remediation worked, gives you documentation for audits and compliance reviews, and becomes your new baseline for the next round of changes.

---

## Step 8: Keep the Baseline Current

A baseline isn't a one-time thing. Systems change — patches go in, software gets installed, Group Policy gets updated, roles shift. Configuration drift happens, and it happens faster than most people expect.

A practical scanning schedule might look like:

- Before production deployment
- After major config changes or OS upgrades
- On a recurring schedule (monthly, quarterly, or whatever your continuous monitoring policy requires)
- Whenever DISA or CIS drops updated benchmark content

Staying current means drift gets caught early, before it turns into a compliance finding or an audit issue.

---

## Wrapping Up

SCAP gives you a structured, repeatable way to assess a Windows system's configuration against a defined security standard. The process itself isn't complicated — get the scanner, get the benchmark, run a baseline scan, review findings, remediate, re-scan, document exceptions, repeat.

The scan report is just a starting point. The actual security improvement comes from what you do with it — reviewing findings with context, testing changes before you push them, documenting what you can't fix, and validating that your work had the intended effect.

For anyone working toward Security+ or already in the field, this is one of those skills that shows up constantly in both enterprise and government work. It connects directly to hardening, vulnerability management, risk management, compliance, and continuous monitoring — and it's a lot more useful once you've actually run a few scans yourself.

---

## References

- NIST Security Content Automation Protocol: [https://csrc.nist.gov/projects/security-content-automation-protocol](https://csrc.nist.gov/projects/security-content-automation-protocol)
- DoD Cyber Exchange STIGs: [https://public.cyber.mil/stigs/](https://public.cyber.mil/stigs/)
- DoD Cyber Exchange Tools: [https://public.cyber.mil/tools/](https://public.cyber.mil/tools/)
- SCAP Compliance Checker (NIWC Atlantic): [https://www.niwcatlantic.navy.mil/Technology/SCAP/](https://www.niwcatlantic.navy.mil/Technology/SCAP/)
- CIS Benchmarks: [https://www.cisecurity.org/cis-benchmarks](https://www.cisecurity.org/cis-benchmarks)
