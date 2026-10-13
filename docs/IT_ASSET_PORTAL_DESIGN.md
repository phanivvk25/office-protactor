# IT Device & Asset Management Portal: Workflows and Features

Status: **draft for review (v3: internal portal for one organisation, tool-agnostic)**.
No code yet. Once this is agreed, the next step is the technical design and code
generation plan.

## 1. Context and assumptions

- An **internal portal for one organisation**. It is not a product sold to customers. The
  only users are the organisation's own IT team, HR, managers and employees.
- The organisation is small or medium-sized, and **does not use paid device-management
  tools** such as Microsoft Intune or Jamf because of their cost.
- Every employee works remotely. There is no office IT desk. The IT team is small.
- IT **ships** a laptop to the employee during onboarding and **collects** it during
  offboarding, through a courier.
- IT has full control of each device. It can restrict or enforce device features such as
  USB/COM ports, camera, microphone, Bluetooth, admin rights and installed software.
- Fleet assumed to be **Windows and macOS** laptops, with **Linux** as an optional later
  addition.

### 1.1 Key design principle: the portal is tool-agnostic

The portal describes **what** IT wants ("block USB drives for the Finance group", "lock
this laptop", "ship to this address"). It does not depend on **how** a specific tool does
it. Every outside system sits behind a **provider interface** (adapter), so a tool can be
replaced without changing the portal, its workflows or its data:

| Provider interface | What the portal asks of it | Examples of implementations (choose later) |
|---|---|---|
| **Device provider** | Enrol device, read inventory, apply policy, run remote action (lock, wipe, restart, message), report compliance | Free/open-source device management server, our own lightweight agent, or a paid tool if the budget changes later |
| **Identity provider** | Sign-in, MFA, list users and groups, disable account, revoke sessions | Google Workspace, Microsoft 365 (any plan), Okta, Keycloak |
| **Shipping provider** | Rates, label, pickup, tracking updates | Courier API, shipping aggregator, or manual tracking numbers |
| **HR source** | New hires, leavers, changes (role, manager, address) | HR tool API, or CSV upload |
| **Notification channel** | Send messages | Email, Slack, Microsoft Teams, SMS |

Each provider declares what it supports (for example "camera block on macOS: yes; USB
port block on macOS: no"). The portal shows that clearly and hides or greys out options a
provider cannot enforce, so IT never thinks a rule is applied when it is not.

### 1.2 Device-side facts that apply whatever tool is chosen

Because there is no Intune/Jamf, the device provider is something the organisation runs
itself (free/open-source software, possibly plus a small agent on each laptop). These OS
facts hold for any such choice and shape the workflows:

| Item | Detail |
|---|---|
| **Windows Home cannot be managed** | The built-in Windows device-management client is only in Pro, Business, Enterprise and Education. Buy **Windows Pro** laptops. Some settings only work on Enterprise/Education. |
| No Windows Autopilot | Autopilot needs Intune. So IT **enrols and sets up each laptop before shipping it**. This fits the process, because IT handles every laptop anyway. |
| Apple push certificate | Managing Macs needs an Apple push certificate. The organisation creates it for free with its own Apple Account and renews it every year (the portal reminds IT before expiry). |
| Mac automatic setup | **Apple Business Manager is free** and works with any Mac device-management server. Macs bought from Apple or an authorised reseller are linked to the organisation; others can be added with Apple Configurator, or enrolled by hand before shipping. |
| macOS USB-drive blocking | Not fully possible through Apple's built-in management. Needs a small agent on the Mac. |
| Agent removal | A local administrator could remove an agent. Employees therefore get **standard (non-admin) accounts**, and the portal alerts when a device stops reporting. |

## 2. Users and roles

| Role | What they do in the portal |
|---|---|
| **Super Admin** | Portal settings, provider connections, role assignment. |
| **IT Admin** | Devices, policies, remote actions, onboarding/offboarding cases. |
| **Logistics / Asset Manager** | Stock, purchasing, shipping, returns, repairs, disposal. |
| **Security / Compliance** | Policy sets, exception approvals, compliance reports (mostly read-only). |
| **HR** | Starts onboarding and offboarding (or this comes from the HR tool / CSV). |
| **Line Manager** | Approves device exceptions and extra hardware for their team. |
| **Employee** | Self-service: own devices, delivery tracking, requests, tickets, return kit. |
| **Auditor** | Read-only access to reports and the audit trail. |

Sign-in through the organisation's identity provider, always with MFA. In a small IT team
one person can hold several roles.

## 3. Device lifecycle

```
 Ordered ─► In Stock ─► Allocated ─► Enrolled & Configured ─► Shipped ─► Delivered ─► Active
                                                                                    │
            ┌───────────────────────── In Repair ◄──────────────────────────────────┤
            │                                                                       │
            ▼                                                                       ▼
         In Stock ◄── Refurbished ◄── Inspected ◄── Received ◄── Return Shipped ◄── Return Requested
                                         │
                                         └──► Retired ──► Disposed (with wipe/recycling certificate)

 Side states from Active: Lost / Stolen (locked, then wiped) · Non-returned (escalated)
```

Compared with v1: devices are **enrolled and configured before shipping**, because Windows
Autopilot is not available.

Every state change is time-stamped, tied to a person, and written to the audit log.

## 4. Core workflows

### 4.1 Onboarding: ship a laptop to a new hire

| # | Step | Owner | System action |
|---|---|---|---|
| 1 | New hire added (start date, role, address, manager) | HR / IT | Entered in the portal, imported from CSV, or received from the HR source. Creates an **Onboarding Case**. |
| 2 | Device profile picked from role (e.g. "Developer – MacBook Air", "Sales – Windows Pro 14″") | Auto, IT can change | Profile defines hardware model, software bundle and **policy set**. |
| 3 | Stock check | Auto | Device is reserved from stock. If none, a purchase request is created. |
| 4 | Laptop enrolled and set up before shipping | IT / warehouse | **Windows:** enrolment package or link run once (plus agent, if the device provider uses one). **Mac:** zero-touch via Apple Business Manager if available, otherwise enrolled by hand. Policies, apps and disk encryption applied. Device assigned to the employee. |
| 5 | Pre-ship check passes (encrypted, policies applied, agent healthy) | Auto | Device cannot be marked "Ready to ship" until it passes. |
| 6 | Shipping label booked; ship-by date set from start date and courier lead time | Logistics | Shipping provider books the label, or a tracking number is typed in by hand. |
| 7 | Employee gets a welcome email with tracking link and a "first sign-in" guide | Auto | Address confirmed through a link before dispatch. |
| 8 | Delivery confirmed | Courier webhook / manual | Device state → Delivered. |
| 9 | Employee signs in with the account IT created and sets their own password | Employee | Device checks in. Final user-specific settings (Wi-Fi, VPN, email) are applied. |
| 10 | Employee acknowledges acceptable-use policy and equipment receipt digitally | Employee | Signed record stored against the asset. |
| 11 | Case closed | Auto | Time from hire to working laptop recorded. |

Exceptions handled: address change, failed delivery, damaged on arrival (start replacement),
device fails to check in after delivery (alert IT), start date moved.

### 4.2 Offboarding: collect the laptop from a leaving employee

| # | Step | Owner | System action |
|---|---|---|---|
| 1 | Leaver entered (last day, voluntary / involuntary) | HR / IT | **Offboarding Case** created listing all assets held by that person. |
| 2 | Cut-off plan chosen | IT Admin | *Voluntary:* revoke on last day. *Involuntary:* revoke **immediately** and remote-lock the laptop, then collect. |
| 3 | Return kit sent (prepaid box + label) or courier pickup booked | Logistics | Shipping API. Employee gets pickup slot and packing instructions. |
| 4 | On last day: company accounts disabled, device locked with a "Please return this laptop" message | Auto + IT | Identity provider disables the account and revokes sessions; otherwise a checklist task for IT. Remote lock through our device layer. |
| 5 | Return tracked | Courier webhook / manual | Reminders to employee at day 3, 7, 14. |
| 6 | Not returned by deadline | Auto | Escalate to HR/manager. Option to **remote wipe**, and to follow company policy (deduction, legal recovery, write-off). |
| 7 | Device received | Logistics | Scan serial. State → Received. |
| 8 | Inspection: condition grade, accessories, damage photos | Logistics | Checklist stored on the asset. |
| 9 | Secure wipe; device unassigned | Auto / IT | Remote wipe command; wipe report stored. Device stays registered for reuse. |
| 10 | Disposition: back to stock, repair, or retire / recycle | Logistics | Retired devices get a disposal certificate from the recycler. |
| 11 | Case closed | Auto | Audit record final. |

### 4.3 Other workflows

| Workflow | Summary |
|---|---|
| **Hardware refresh** | Devices reaching age limit or warranty end are flagged. New device shipped first, then the old one is returned through 4.2 steps 3–10. |
| **Break / fix replacement** | Employee raises a ticket. If hardware failure: replacement shipped, faulty one returned and sent to vendor repair. |
| **Lost or stolen** | Employee reports in portal (one click), or IT marks it. Immediate remote lock + on-screen message, then wipe. Replacement shipped. |
| **Policy change** | IT changes a policy set → preview of affected devices → test on a few devices → roll out → compliance report. Rollback available. |
| **Temporary policy exception** | Employee asks e.g. "enable USB drive for 2 days". Manager approves. The portal applies it and **removes it automatically** when it expires. |
| **Accessory requests** | Monitor, headset, dock, etc. Approved by manager, shipped and tracked. |
| **Address change / relocation** | Employee updates shipping address; country change flags different shipping and legal rules. |
| **First-time setup** | Connect identity provider → connect device provider (incl. Apple push certificate) → choose default policy sets → import existing laptops (CSV) and enrol them in place with a link. |

## 5. Feature list by module

### 5.1 Dashboard
- Fleet overview: total devices, by state, by OS, by country.
- Compliance score; non-compliant devices list.
- Open onboarding / offboarding cases, shipments in transit, overdue returns.
- Alerts: devices not seen in N days, encryption off, agent missing, warranty expiring.

### 5.2 Asset inventory
- Asset record: serial, asset tag, model, specs, purchase date, cost, vendor, warranty,
  assigned user, ship address, state, condition, full history.
- Accessories tracked and linked to the main device.
- CSV import, barcode/QR scan with a phone camera.
- Stock levels per model and location, low-stock alerts.
- Simple depreciation / book value.

### 5.3 Device management (through the device provider)
- **Enrolment:** Windows enrolment package / link, Mac zero-touch via Apple Business
  Manager or enrolment profile, Linux install script.
- Live device data: OS version, last check-in, encryption state,
  installed apps, storage, battery health, logged-in user.
- **Remote actions:** lock, restart, wipe, run approved script, rotate local admin
  password, sync policies, collect logs, show message on screen, locate (only for
  lost/stolen, where legal).
- Device groups: by department, country, role, OS (manual or rule-based).

### 5.4 Policy & device control (the restriction engine)

Policies are offered as **plain-language toggles grouped into policy sets** ("Standard",
"Strict / Finance", "Developer"), with exceptions on top. The portal stores the rule in a
tool-neutral form; the device provider translates it for each OS. The table shows how each
OS can enforce it, for reference.

| Control | Options | Windows (how it can be enforced) | macOS (how it can be enforced) | Linux (how it can be enforced) |
|---|---|---|---|---|
| USB storage | Allow / Read-only / Block / Allow approved device IDs | MDM: removable storage and device-install policies | Agent (native MDM is limited, see §1.2) | USBGuard |
| Other USB / serial (COM) ports | Block by device class; always allow keyboards, mice, docks, headsets | MDM: device-install restrictions by class/ID; agent for detail | Agent | USBGuard / udev rules |
| Camera | Allow / Block | MDM: camera policy | MDM: restrictions profile | Agent: disable camera driver |
| Microphone | Allow / Block per app | MDM: privacy policy | MDM: privacy (PPPC) profile | Agent |
| Bluetooth | Allow / Block / Audio only | MDM: Bluetooth policy | MDM profile + agent | Agent |
| Wi-Fi / VPN | Push company Wi-Fi; VPN settings | MDM: Wi-Fi / VPN profiles | MDM: Wi-Fi / VPN profiles | Agent |
| Printing | Allow / Block | MDM / agent | MDM profile | Agent |
| Admin rights | Standard user only; temporary admin on approval | Agent + local admin password rotation | Agent + managed admin account | Agent (sudo rules) |
| Disk encryption | Enforced; recovery key stored in portal | BitLocker via MDM; agent sends recovery key | FileVault via MDM; key escrowed through MDM | Agent check (LUKS) |
| OS updates | Deadline and restart window | MDM: Windows Update policies | MDM: software update commands | Agent |
| Firewall & antivirus | Must be on | MDM: Microsoft Defender (built into Windows) | MDM: firewall profile; antivirus optional | Agent |
| App control | Required apps; blocked apps list | Agent installs (e.g. via winget) / blocks | MDM / agent installs; agent blocks | Agent (package manager) |
| Screen lock / password | Auto-lock time, password rules | MDM: device lock policy | MDM: passcode profile | Agent |
| Browser | Extension allow-list, settings | MDM: Chrome/Edge policies | MDM: Chrome/Edge profiles | Agent |

Policy features:
- Ready-made policy sets; custom sets for special groups.
- **Exceptions** with requester, reason, approver, start and expiry; auto-removed.
- Preview: which devices change before publishing.
- Version history and rollback.
- Drift detection: alert (and re-apply) when a device differs from its policy.
- Each rule shows a "works on" badge from the provider's declared support (Windows Pro / Enterprise only, macOS version, etc.).

### 5.5 Compliance & security
- Compliance rules: encryption on, OS up to date, agent running, firewall on, policies
  applied.
- When Google Workspace or Microsoft 365 is connected: block or alert on non-compliant
  devices where those platforms' plans allow it. Otherwise: alert IT and the employee.
- Simple security score per device and for the whole fleet.
- Evidence reports for audits and cyber-insurance questionnaires (SOC 2 / ISO 27001) (encryption coverage, patch level, inventory).

### 5.6 Software & patch management
- App catalogue: required apps (auto-installed) and optional apps (self-service install),
  using free package sources (winget on Windows, Homebrew or vendor installers on Mac).
- OS and app patch status per device.
- Basic licence counting (seats used vs purchased).

### 5.7 Logistics & shipping
- Shipping provider integration (rates, labels, pickups, tracking);
  manual tracking numbers also supported.
- Return kit management and return addresses per region.
- Address validation; customs documents for international shipping; laptop battery
  shipping rules.
- Ship-by dates and delays visible on the dashboard.

### 5.8 Procurement & vendors
- Purchase requests with simple approval.
- Standard models list with prices.
- Warranty and repair (RMA) tracking.

### 5.9 Employee self-service portal
- "My devices": what I hold, warranty, my return obligations.
- Delivery and return tracking.
- Requests: accessory, software, policy exception, replacement.
- Report lost/stolen (one click → device locked).
- Support tickets; first-sign-in guides.
- Digital acknowledgements (acceptable-use policy, equipment receipt).

### 5.10 Ticketing (built in)
- Light built-in ticketing linked to device and user (no separate ITSM tool assumed).
- Email-to-ticket. Optional later integration with Freshdesk / Zendesk / Jira.

### 5.11 Workflow & automation
- Onboarding / offboarding checklists, editable per country, department or role.
- Approvals, reminders, escalations.
- Event triggers (new hire, device non-compliant, shipment delivered) → actions.

### 5.12 Notifications
- Email, Slack, Microsoft Teams, SMS (delivery and return reminders).
- Templates per event and language.

### 5.13 Reporting
- Fleet age, cost per employee, spend by department.
- Time-to-productive for new hires; return rate and losses.
- Compliance trends; exception counts.
- CSV / PDF export.

### 5.14 Audit log
- Every admin action, remote command, policy change, approval and state change.
- Who, what, when, from where (IP), before/after values.
- Tamper-evident and exportable.

### 5.15 Administration
- Provider connections (device, identity, shipping, HR, notifications) with health status
  and "what this provider supports" view.
- Roles and permissions; approval rules (e.g. wipe needs a second approver).
- Master data: device models, policy sets, warehouses / return addresses, countries,
  checklists, notification templates.
- Certificate and credential expiry reminders (e.g. Apple push certificate, API keys).

### 5.16 Integrations (through provider interfaces, all optional and replaceable)
- Identity: Google Workspace, Microsoft 365 (any plan), Okta, Keycloak.
- User sync from the identity provider, or CSV.
- HR: CSV first; HR tool API later.
- Shipping: courier or aggregator API, or manual tracking numbers.
- Slack / Teams / email / SMS.
- Portal REST API and webhooks for the organisation's own scripts (later phase).

## 6. Non-functional requirements

| Area | Requirement |
|---|---|
| Security | MFA everywhere. RBAC with least privilege. Wipe, lock and policy publishing need re-authentication (and optionally a second approver). Provider credentials kept in a secrets store. Encryption in transit and at rest. Penetration test before go-live. |
| Tool independence | No provider-specific fields in the core data model. Swapping a provider must not need changes to workflows, policies or history. |
| Privacy | Employees work from home: no screenshots, webcam, keystroke or browsing monitoring. Location only for lost/stolen. Employees can see what is collected. Follow GDPR and local employment rules for each country where employees live. |
| Reliability | Remote lock/wipe and access revocation are queued and retried until the device or provider confirms, with clear status ("pending", "delivered", "done"). |
| Cost | Runs on modest infrastructure; no paid per-device licences required. |
| Scale | Sized for the organisation (hundreds to a few thousand devices), multiple countries. |
| Auditability | Full tamper-evident audit trail. |

## 7. High-level architecture (for later detailed design)

```
         Employees / IT / HR / Managers (browser, SSO + MFA)
                              │
                    ┌─────────▼──────────┐
                    │  Web portal (SPA)  │
                    └─────────┬──────────┘
                              │ REST API
        ┌─────────────────────▼──────────────────────────┐
        │  Portal backend (single organisation)           │
        │  ├ Users, roles, approvals                      │
        │  ├ Assets & inventory                           │
        │  ├ Workflow / case engine (on/offboarding)      │
        │  ├ Policy service (tool-neutral rules,          │
        │  │   policy sets, exceptions)                   │
        │  ├ Logistics service                            │
        │  ├ Ticketing, notifications, audit log          │
        │  └ Job queue + workers (retries, schedules)     │
        └──────────────────────┬─────────────────────────┘
                               │
         ┌──────────── Provider interfaces (adapters) ─────────────┐
         │  Device  │ Identity │ Shipping │ HR source │ Notification │
         └────┬─────┴────┬─────┴────┬─────┴─────┬─────┴──────┬───────┘
              ▼          ▼          ▼           ▼            ▼
       device mgmt    Google /   courier /   HR tool /   email / Slack /
       tool(s) and    M365 /     aggregator  CSV         Teams / SMS
       agents         Okta
              ▼
       Employee laptops (Windows Pro / macOS / Linux)
```

## 8. Suggested phasing

| Phase | Scope |
|---|---|
| **0. Proof of concept (2–3 weeks)** | Pick the first device provider by testing on real laptops: enrol, camera block, USB storage block (Windows Pro + macOS), encryption + recovery key storage, remote lock/wipe. |
| **1. MVP** | Sign-in + MFA, roles, asset inventory, onboarding/offboarding cases with checklists, manual tracking numbers, device provider interface + first provider, inventory data, remote lock/wipe, rules for camera / USB storage / encryption / screen lock, employee "My devices", audit log, dashboard. |
| **2** | Full policy set (COM ports, Bluetooth, admin rights, apps, updates), exceptions with auto-expiry, compliance view, shipping provider, notifications, built-in ticketing. |
| **3** | HR / directory sync, procurement & stock, refresh and RMA workflows, reports, Linux support. |
| **4** | Portal API & webhooks, extra providers as needed. |

## 9. What the portal deliberately does not do

- It does not monitor employees (no screenshots, webcam or keystroke capture).
- It does not support Windows Home (not manageable).
- It is not a full ITSM tool; ticketing is kept light.
- It does not hard-code any device-management, identity or courier vendor.

## 10. Open questions for review

1. Confirm **Windows Pro required** for all employee laptops.
2. Linux: needed in the first year, or later?
3. Number of employees / devices, and countries they live in (shipping, privacy law)?
4. Does IT keep laptops in its own stock and ship them, or does the vendor ship directly?
5. Which identity provider is used today (Google Workspace, Microsoft 365, other)?
6. Is there an HR tool, or is CSV / manual entry enough at first?
7. Policy for non-returned laptops (deduction, legal recovery, write-off)?
8. Who approves policy exceptions (manager only, or manager + security)?
9. Preferred tech stack and hosting (cloud or on-premises)?
