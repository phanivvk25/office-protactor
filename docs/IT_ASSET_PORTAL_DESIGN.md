# IT Device & Asset Management Portal for SMBs: Workflows and Features

Status: **draft for review (v2: SMB target, no Intune / Jamf)**. No code yet. Once this is
agreed, the next step is the technical design and code generation plan.

## 1. Context and assumptions

- **Target customer: small and medium businesses (SMBs)**, roughly 10–500 employees, that
  cannot afford Microsoft Intune, Jamf or similar commercial device-management tools.
- The portal is sold as a **multi-tenant SaaS product**: many SMB customers on one platform,
  each with its own isolated data.
- Every employee works remotely. There is no office IT desk. Often the "IT team" is one or
  two people, or an outside IT provider (MSP).
- IT **ships** a laptop to the employee during onboarding and **collects** it during
  offboarding, through a courier.
- IT has full control of each device. It can restrict or enforce device features such as
  USB/COM ports, camera, microphone, Bluetooth, admin rights and installed software.
- Fleet assumed to be **Windows and macOS** laptops, with **Linux** as an optional later
  addition.

### What SMBs need that is different

| Need | Effect on the product |
|---|---|
| Low price | No dependency on paid device-management licences. Our own device layer, priced per device per month. |
| Very small IT team | Sensible default policies out of the box, guided setup, automation over configuration. |
| No in-house expertise | Plain-language policy names ("Block USB drives"), not OS setting names. |
| Mixed, cheap hardware | Must handle Windows **Pro** (Windows Home cannot be managed, see §1.2) and older Macs. |
| Often served by an MSP | One MSP login can manage several customer companies. |
| Light tooling | Works with Google Workspace or Microsoft 365 Business Basic/Standard sign-in, simple HR tools or CSV, built-in ticketing. |

### 1.1 Key design decision: our own device-management layer

Without Intune or Jamf, the product must manage the laptops itself. That needs two things
on the device side:

1. **An MDM server** that speaks the device-management protocol built into each OS. Windows
   and macOS both include an MDM client at no cost. Only the server side is needed.
2. **A lightweight agent** of our own, installed on every laptop, for what the built-in MDM
   protocol cannot do (detailed USB/COM port control, Linux, inventory detail, health
   checks).

Options for building that layer:

| Option | What it means | Effort | Risk |
|---|---|---|---|
| A. Build everything | Write our own Windows MDM server, Apple MDM server and agent | Very high | Protocols are complex. Long time to a first version. |
| **B. Build on open-source components (recommended)** | Use proven open-source projects for the protocol work, wrapped by our portal and our agent | Medium | Licence terms must be checked. We depend on upstream projects. |
| C. Agent only, no MDM | Our agent does everything through OS commands and scripts | Medium | No zero-touch Mac enrolment. Agent can be removed by a local admin. Weaker remote wipe. Not recommended alone. |

Open-source building blocks to evaluate in a proof of concept:

| Component | Candidate | Notes |
|---|---|---|
| Combined device layer (Windows + macOS + Linux) | **Fleet** (fleetdm, osquery-based) | Has MDM for macOS and Windows plus an agent. Core is open source, but some features are paid ("Premium"). **Check the licence for each feature we rely on, and whether reselling it inside a SaaS product is allowed.** |
| Apple MDM server | **NanoMDM** (+ NanoDEP for zero-touch) | Open source, protocol only. We build everything above it. |
| Windows MDM server | Own implementation of the Windows MDM protocol (MS-MDE2 enrolment + OMA-DM) | Protocol is publicly documented. Fewer mature open-source servers than for Apple. |
| Inventory / queries | **osquery** | Open source. Read device details through SQL-style queries. |
| Linux USB control | **USBGuard** | Open source. Allow-list of USB devices. |

**Recommendation:** run a 2–3 week proof of concept with Fleet first, because it covers
all three OSes. If its licence or feature split does not fit, fall back to NanoMDM (Mac) +
our own Windows MDM server + our own agent. In both cases, put the device layer behind a
**provider interface** in our code so we can swap it later. A customer that already has
Intune can still be connected through the same interface later. This is optional and not
part of the core product.

### 1.2 Platform prerequisites and limits (must be told to customers)

| Item | Detail |
|---|---|
| **Windows Home cannot be managed** | The Windows MDM client is only in Pro, Business, Enterprise and Education. SMBs must buy **Windows Pro** laptops, or upgrade Home to Pro. Some settings only work on Enterprise/Education. Check each policy on Pro in the proof of concept. |
| No Windows Autopilot | Autopilot needs Intune. Instead, IT **enrols the laptop before shipping** (at the warehouse/office), or the employee opens an enrolment link on first boot. Shipping pre-enrolled devices fits this business well, since IT handles every laptop anyway. |
| Apple push certificate | Apple MDM needs an Apple push certificate. Either each customer creates one with their own Apple ID (free), or **we become an Apple-approved MDM vendor** (Apple Developer Program) to make setup simpler. |
| Mac zero-touch enrolment | **Apple Business Manager is free** and works with any MDM, including ours. Macs must be bought from Apple or an authorised reseller who adds them to the customer's account. Otherwise they can be added with Apple Configurator, or enrolled manually before shipping. |
| macOS USB storage blocking | Not fully available through native MDM. Needs our agent, using the macOS disk-mount control APIs or Endpoint Security. **Endpoint Security needs a special entitlement from Apple.** Apply early, because approval can take time. |
| Agent removal | A local administrator could remove our agent. So employees get **standard (non-admin) accounts**, and the MDM layer reports and re-installs a missing agent. |

## 2. Users and roles

| Role | What they do in the portal |
|---|---|
| **Platform Admin (us)** | Manage customer tenants, billing, support access (only when the customer allows it). |
| **MSP Admin** | Optional. One login for many customer companies; switch between them, see a combined view. |
| **Company Owner / Super Admin** | Company settings, billing, integrations, role assignment. |
| **IT Admin** | Devices, policies, remote actions, onboarding/offboarding cases. In small SMBs this is the same person as the Owner. |
| **Logistics / Asset Manager** | Stock, purchasing, shipping, returns, repairs, disposal. |
| **HR** | Starts onboarding and offboarding (or this comes from the HR tool / CSV). |
| **Line Manager** | Approves device exceptions and extra hardware for their team. |
| **Employee** | Self-service: own devices, delivery tracking, requests, tickets, return kit. |
| **Auditor** | Read-only access to reports and the audit trail. |

Sign-in: Google Workspace, Microsoft 365 (any plan) or email + password, always with MFA.
Roles are simplified for small customers: one person can hold several.

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
| 1 | New hire added (start date, role, address, manager) | HR / IT | Entered in the portal, imported from CSV, or received from a supported HR tool (e.g. BambooHR, HiBob, Gusto). Creates an **Onboarding Case**. |
| 2 | Device profile picked from role (e.g. "Developer – MacBook Air", "Sales – Windows Pro 14″") | Auto, IT can change | Profile defines hardware model, software bundle and **policy set**. |
| 3 | Stock check | Auto | Device is reserved from stock. If none, a purchase request is created. |
| 4 | Laptop enrolled and set up before shipping | IT / warehouse | **Windows:** enrolment package or link run once; our agent is installed. **Mac:** zero-touch via Apple Business Manager if available, otherwise enrolled by hand. Policies, apps and disk encryption applied. Device assigned to the employee. |
| 5 | Pre-ship check passes (encrypted, policies applied, agent healthy) | Auto | Device cannot be marked "Ready to ship" until it passes. |
| 6 | Shipping label booked; ship-by date set from start date and courier lead time | Logistics | Low-cost shipping API (e.g. Shippo or EasyPost, which reach many couriers), or a tracking number typed in by hand. |
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
| 4 | On last day: company accounts disabled, device locked with a "Please return this laptop" message | Auto + IT | Disables account in Google Workspace / Microsoft 365 when connected; otherwise a checklist task for IT. Remote lock through our device layer. |
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
| **New customer setup (SaaS)** | Sign up → connect sign-in (Google / Microsoft) → Apple push certificate wizard → choose default policy template → enrolment instructions → import existing laptops (enrol them in place with a link). |

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

### 5.3 Device management (our device layer)
- **Enrolment:** Windows enrolment package / link, Mac zero-touch via Apple Business
  Manager or enrolment profile, Linux install script. Agent installed automatically.
- Live device data (via agent / osquery): OS version, last check-in, encryption state,
  installed apps, storage, battery health, logged-in user.
- **Remote actions:** lock, restart, wipe, run approved script, rotate local admin
  password, sync policies, collect logs, show message on screen, locate (only for
  lost/stolen, where legal).
- Device groups: by department, country, role, OS (manual or rule-based).

### 5.4 Policy & device control (the restriction engine)

Policies are offered as **plain-language toggles grouped into templates** ("Standard",
"Strict / Finance", "Developer"), with exceptions on top. Each toggle maps to the right
mechanism per OS.

| Control | Options | Windows (built-in MDM / our agent) | macOS (built-in MDM / our agent) | Linux (agent) |
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
- Ready-made templates for SMBs; custom policy sets for advanced users.
- **Exceptions** with requester, reason, approver, start and expiry; auto-removed.
- Preview: which devices change before publishing.
- Version history and rollback.
- Drift detection: the agent reports when a device differs from its policy, and fixes it.
- Each policy shows a "works on" badge (Windows Pro / Enterprise only, macOS version, etc.).

### 5.5 Compliance & security
- Compliance rules: encryption on, OS up to date, agent running, firewall on, policies
  applied.
- When Google Workspace or Microsoft 365 is connected: block or alert on non-compliant
  devices where those platforms' plans allow it. Otherwise: alert IT and the employee.
- Simple security score per device and per company.
- Evidence reports for customers going through SOC 2 / ISO 27001 / cyber-insurance
  questionnaires (encryption coverage, patch level, inventory).

### 5.6 Software & patch management
- App catalogue: required apps (auto-installed) and optional apps (self-service install),
  using free package sources (winget on Windows, Homebrew or vendor installers on Mac).
- OS and app patch status per device.
- Basic licence counting (seats used vs purchased).

### 5.7 Logistics & shipping
- Shipping API integration (rates, labels, pickups, tracking) through one low-cost
  aggregator; manual tracking numbers also supported.
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
- Light built-in ticketing linked to device and user, since most SMBs have no ITSM tool.
- Email-to-ticket. Optional later integration with Freshdesk / Zendesk / Jira.

### 5.11 Workflow & automation
- Ready-made onboarding / offboarding checklists, editable per company.
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

### 5.15 SaaS administration (new)
- Self-service sign-up, free trial, per-device monthly billing (e.g. Stripe).
- Plans: e.g. *Basic* (inventory + logistics), *Standard* (+ device control), *Pro*
  (+ automation, reports, integrations).
- **MSP mode**: one login, many customer companies, combined dashboard.
- Tenant isolation, per-tenant data export, and deletion when a customer leaves.
- Support access to a customer tenant only with customer approval, and logged.

### 5.16 Integrations (kept cheap and optional)
- Sign-in: Google Workspace, Microsoft 365 (any plan), email + MFA.
- User sync: Google Workspace / Microsoft 365 directory, or CSV.
- HR: CSV first; then BambooHR, HiBob, Gusto, etc.
- Shipping aggregator, Slack / Teams, Stripe.
- Public REST API and webhooks (later phase).

## 6. Non-functional requirements

| Area | Requirement |
|---|---|
| Security | MFA everywhere. RBAC. Wipe, lock and policy publishing need re-authentication (and a second approver if the company enables it). The **device layer is the most sensitive part**: one bug could affect every customer's laptops. So it needs signed agent updates, mutual TLS between agent and server, signed commands and strict tenant isolation. Plan an external penetration test before launch. |
| Agent quality | Small footprint (CPU, memory, battery). Auto-update with staged rollout and rollback. Signed and notarised installers (Windows code signing certificate, Apple Developer ID). |
| Privacy | Employees work from home: no screenshots, webcam, keystroke or browsing monitoring. Location only for lost/stolen. Employees can see what is collected. Help customers with GDPR and local employment rules. |
| Reliability | Remote lock/wipe commands are queued and retried until the device checks in, with clear status ("pending", "delivered", "done"). |
| Cost to run | Must stay low, to keep the product cheap for SMBs. Shared multi-tenant infrastructure; efficient agent check-ins. |
| Scale | Thousands of tenants, tens of thousands of devices at first. |
| Data residency | EU and US regions (later phase). |

## 7. High-level architecture (for later detailed design)

```
     Employees / IT / HR / MSPs (browser, SSO + MFA)
                          │
                ┌─────────▼──────────┐
                │   Web portal (SPA) │
                └─────────┬──────────┘
                          │ REST API
        ┌─────────────────▼───────────────────────────────┐
        │  Portal backend (multi-tenant)                   │
        │  ├ Tenants, billing, RBAC                        │
        │  ├ Assets & inventory                            │
        │  ├ Workflow / case engine (on/offboarding)       │
        │  ├ Policy service (templates, exceptions)        │
        │  ├ Logistics service                             │
        │  ├ Ticketing, notifications, audit log           │
        │  └ Device provider interface ───────────────┐    │
        └──────────────┬──────────────────────────────┼────┘
                       │ job queue + workers          │
        ┌──────────────▼────────┐            ┌────────▼─────────────────────────┐
        │ Shipping / HR / SSO / │            │  Device layer (ours)              │
        │ Slack / Stripe APIs   │            │  ├ Apple MDM server  (e.g. NanoMDM│
        └───────────────────────┘            │  │   or Fleet)  ◄── Apple push     │
                                             │  ├ Windows MDM server (OMA-DM)     │
                                             │  └ Agent server (check-ins,       │
                                             │      commands, osquery results)   │
                                             └───────┬───────────────┬───────────┘
                                                     │               │
                                     built-in OS MDM client     our agent
                                                     ▼               ▼
                                       Employee laptops (Windows Pro / macOS / Linux)
```

## 8. Suggested phasing

| Phase | Scope |
|---|---|
| **0. Proof of concept (2–3 weeks)** | Fleet vs NanoMDM + own Windows MDM. Prove on real devices: enrol, camera block, USB storage block (Windows Pro + macOS), encryption + key escrow, remote lock/wipe. Confirm licences. Apply for Apple MDM vendor status and the Endpoint Security entitlement. |
| **1. MVP** | Multi-tenant SaaS, sign-in + MFA, asset inventory, onboarding/offboarding cases with checklists, manual tracking numbers, enrolment + agent, inventory data, remote lock/wipe, policies for camera / USB storage / encryption / screen lock, employee "My devices", audit log, dashboard. |
| **2** | Full policy set (COM ports, Bluetooth, admin rights, apps, updates), exceptions with auto-expiry, compliance view, shipping API, notifications, built-in ticketing, billing. |
| **3** | MSP mode, HR / directory sync, procurement & stock, refresh and RMA workflows, reports, Linux support. |
| **4** | Public API & webhooks, data residency, optional connectors to existing Intune / Jamf / Google endpoint management for customers who have them. |

## 9. What the product deliberately does not do

- It does not monitor employees (no screenshots, webcam or keystroke capture).
- It does not support Windows Home (not manageable).
- It is not a full ITSM tool; ticketing is kept light.

## 10. Open questions for review

1. Confirm **Windows Pro required** is acceptable for target customers.
2. Linux: needed in the first year, or later?
3. Are we comfortable depending on Fleet (open-core licence), or do we prefer the
   NanoMDM + own Windows MDM route for full control?
4. Should we become an Apple MDM vendor (simpler customer setup), or have each customer
   create their own Apple push certificate?
5. Main customer regions (affects shipping partners, privacy law, data residency)?
6. Do we hold stock / ship for customers (a logistics service), or only track shipments
   the customer makes?
7. Is MSP mode a priority (selling through IT providers)?
8. Pricing idea: per device per month, and which features go in which plan?
9. Policy for non-returned devices (deduction, legal recovery, write-off): a fixed
   product feature or customer-configurable?
10. Preferred tech stack and cloud hosting?
