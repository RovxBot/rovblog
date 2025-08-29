# Drop‑in first post

## Introduction

Welcome to my first post on Windows Defender Application Control (WDAC), also known as App Control for Business. This post will walk you through creating your first WDAC policy using SpyNetGirl's AppControl Manager tool, which simplifies the process significantly.

## WDAC capabilities at a glance

WDAC is a powerful security feature in Windows that allows you to control which applications and scripts can run on your devices. Key capabilities include:

- Creating base and supplemental policies
- Defining rules based on file attributes like publisher, file hash, path, and more
- Running in audit mode to monitor without blocking
- Enforcing policies to block unapproved applications
- Generating detailed event logs for analysis

## Prerequisites

- Windows 10 or Windows 11 device with administrative privileges
- PowerShell 5.1 or later
- A test app you expect to be blocked (so we can generate audit events).

## Install AppControl Manager

**Option A — Microsoft Store (recommended):**
- Open Microsoft Store and install **AppControl Manager**.

Option B — One‑liner (offline‑friendly bootstrap from GitHub):
```powershell
(irm 'https://raw.githubusercontent.com/HotCakeX/Harden-Windows-Security/main/Harden-Windows-Security.ps1')+'AppControl'|iex
```

## Create a base policy

Open AppControl Manager > Create AppControl Policy > Choose a starting template:
- DefaultWindows_Audit - allows Windows + drivers + Store apps.
- Allow Microsoft - adds trust for Microsoft-signed apps (Office, Teams, etc.)

Set these policy rule option sin the UI:
- Enabled: Audit Mode
- Enabled: Allow Supplemental Policies
- Managed Installer (If you use Intune or ConfigMgr to deploy your apps.)

Using Audit mode first allows you to deploy your policy to a group of test users and collect real world audit telemetry that you can use to creaft your supplemental policies later.

## Deploy

Deploy the base policy to your devices via Group Policy, Intune, or local deployment:

- For local deployment, run:

```powershell
Set-CIPolicy -PolicyPath "C:\Path\To\YourPolicy.xml" -Deploy
```

- For Intune, upload the XML as a custom OMA-URI policy.

## Generate audit data

To gather audit data for supplemental policies, switch your base policy to audit mode:

```powershell
Set-CIPolicy -PolicyPath "C:\Path\To\YourPolicy.xml" -Audit
```

Deploy this audit mode policy to your devices and collect event logs over time.

## Create supplemental policy

Use AppControl Manager to create supplemental policies based on audit events:

1. Review audit events to identify blocked applications you want to allow.
2. Create a supplemental policy in AppControl Manager.
3. Add rules for the identified applications.
4. Link the supplemental policy to your base policy.
5. Export and deploy the supplemental policy.

## Flip to enforce

After testing and verifying your policies, switch from audit to enforce mode to block unapproved applications:

```powershell
Set-CIPolicy -PolicyPath "C:\Path\To\YourPolicy.xml" -Enforce
```

Deploy the enforce mode policy to your devices.

## What’s next

In upcoming posts, I will cover:

- Deep dive into WDAC capabilities
- Using AppControl Manager advanced features
- Building supplemental policies from audit logs
- Deploying policies with Intune
- Managing multiple policies and updates
- WDAC hardening and ongoing management strategies

Stay tuned!

---

# WDAC (App Control for Business) Capabilities

## Overview
This post will explore WDAC/App Control for Business capabilities, including:
- Policy rule options
- File rule levels
- Event logging
- Audit vs Enforce modes

*Coming soon...*

---

# Building Supplemental Policies

## Overview
This post will show how to build supplemental policies from audit events, link them to a base policy, and deploy them.

*Coming soon...*

---

# WDAC Deployment with Intune

## Overview
This post will detail deploying WDAC policies via Microsoft Intune, including both built-in App Control for Business and custom XML/OMA-URI deployments.

*Coming soon...*

---

# Managing Multiple WDAC Policies

## Overview
This post will explain base vs supplemental, side-by-side policies, and changes in recent Windows updates.

*Coming soon...*

---

# WDAC Hardening and Ongoing Management

## Overview
This post will cover using Microsoft's recommended driver and app block lists, handling packaged apps, and maintaining WDAC policies over time.

*Coming soon...*
