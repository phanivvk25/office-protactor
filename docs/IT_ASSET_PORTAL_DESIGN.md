# IT Device & Asset Management Portal: Workflows and Features

Status: **draft for review**. No code yet. Once this is agreed, the next step is the
technical design and code generation plan.

## 1. Context and assumptions

- Every employee works remotely. There is no office IT desk.
- IT **ships** a laptop to the employee during onboarding and **collects** it during
  offboarding, through a courier.
- IT has full control of each device. It can restrict or enforce device features such as
  USB/COM ports, camera, microphone, Bluetooth, admin rights and installed software.
- Fleet assumed to be **Windows and macOS** laptops. Linux, if needed, is listed as an open
  question in §10.

### Key design decision: build on top of an MDM, do not replace it

Locking ports or the camera on a laptop needs software running on the device with OS-level
rights. Writing and maintaining that agent ourselves is a large, security-critical job.

**Recommendation:** the portal is the **orchestration and workflow layer**. It enforces
policies through an existing MDM/UEM platform's API:

| OS | MDM option (examples) | Enrolment with no IT touch |
|---|---|---|
| Windows | Microsoft Intune | Windows Autopilot |
| macOS | Intune, Jamf Pro or Kandji | Apple Business Manager + Automated Device Enrollment |

Some controls, such as USB mass-storage blocking on macOS, are weak or absent in native MDM
and need an endpoint security/DLP agent (for example Microsoft Defender for Endpoint device
control, or Jamf Protect). The portal treats that agent as a second "enforcement provider".

What the portal adds on top of the MDM: one place for the whole lifecycle (HR trigger →
purchase → shipping → enrolment → policy → return → disposal), logistics tracking,
approvals, employee self-service, and audit and reporting across tools.

## 2. Users and roles

| Role | What they do in the portal |
|---|---|
| **Super Admin** | Tenant settings, integrations, role assignment. |
| **IT Admin** | Devices, policies, remote actions, onboarding/offboarding cases. |
| **IT Logistics / Asset Manager** | Stock, procurement, shipping, returns, repairs, disposal. |
| **Security / Compliance Officer** | Policy baselines, exception approvals, compliance reports, audit log (mostly read-only). |
| **HR** | Starts onboarding and offboarding (or this comes automatically from the HR system). |
| **Line Manager** | Approves device exceptions and extra hardware for their team. |
| **Employee** | Self-service: own devices, delivery tracking, requests, tickets, return kit. |
| **Auditor** | Read-only access to reports and the audit trail. |

All access uses single sign-on (Entra ID / Okta) with MFA and role-based access control.

## 3. Device lifecycle

```
 Ordered ─► In Stock ─► Allocated ─► Provisioning ─► Shipped ─► Delivered ─► Active
                                                                           │
            ┌───────────────────── In Repair ◄────────────────────────────┤
            │                                                              │
            ▼                                                              ▼
         In Stock ◄── Refurbished ◄── Inspected ◄── Received ◄── Return Shipped ◄── Return Requested
                                         │
                                         └──► Retired ──► Disposed (with wipe/recycling certificate)

 Side states from Active: Lost / Stolen (locked, then wiped) · Non-returned (escalated)
```

Every state change is time-stamped, tied to a person, and written to the audit log.

## 4. Core workflows

### 4.1 Onboarding: ship a laptop to a new hire

| # | Step | Owner | System action |
|---|---|---|---|
| 1 | New hire created in the HR system (start date, role, address, manager) | HR | HR system webhook creates an **Onboarding Case**; manual entry is also possible. |
| 2 | Device profile picked from role (e.g. "Engineer – MacBook Pro 16 GB", "Sales – Windows 14″") | Auto, IT Admin can change | Profile defines hardware model, software bundle and **policy baseline**. |
| 3 | Stock check | Auto | Device is reserved from stock. If none, a purchase request is created and sent for approval. |
| 4 | Device pre-registered for zero-touch enrolment | Auto | Serial number assigned to user in Autopilot / Apple Business Manager. Asset tag created. |
| 5 | Shipping label booked, ship-by date set from start date and courier lead time | IT Logistics | Courier API (e.g. FedEx, DHL, Shippo / EasyPost) books shipment. Tracking number stored. |
| 6 | Employee gets a welcome email with tracking link and "first boot" guide | Auto | Notification. Address confirmation link sent before dispatch. |
| 7 | Delivery confirmed | Courier webhook | Device state → Delivered. |
| 8 | Employee powers on and signs in with company account | Employee | Device enrols itself in MDM. Policies, apps, encryption and VPN applied automatically. |
| 9 | Compliance check passes (encryption on, OS version, agent healthy) | Auto | Only now does conditional access let the device reach company apps (email, code, etc.). |
| 10 | Employee acknowledges acceptable-use policy and the equipment receipt digitally | Employee | Signed record stored against the asset. |
| 11 | Case closed | Auto | SLA metrics recorded (days from hire to working laptop). |

Exceptions handled: address change, failed delivery, damaged on arrival (start replacement),
enrolment stuck (alert IT), start date moved.

### 4.2 Offboarding: collect the laptop from a leaving employee

| # | Step | Owner | System action |
|---|---|---|---|
| 1 | Termination entered in HR system (last day, type: voluntary / involuntary) | HR | **Offboarding Case** created listing all assets held by that person. |
| 2 | Access cut-off plan chosen | IT Admin | *Voluntary:* revoke on last day. *Involuntary:* revoke **immediately**, lock the device remotely, then collect. |
| 3 | Return kit sent (prepaid box + label) or courier pickup booked | IT Logistics | Courier API. Employee gets pickup slot and packing instructions. |
| 4 | On last day: account disabled, sessions revoked, device placed in "Return mode" | Auto | MDM: remote lock or restricted mode; device can no longer reach company data. |
| 5 | Pickup / return tracked | Courier webhook | Reminders to employee at day 3, 7, 14. |
| 6 | Not returned by deadline | Auto | Escalate to HR/manager; option to trigger **remote wipe** and mark for payroll deduction or legal recovery (per company policy). |
| 7 | Device received at warehouse/IT | IT Logistics | Scan serial. State → Received. |
| 8 | Inspection: condition grade, accessories present, damage photos | IT Logistics | Checklist stored on the asset. |
| 9 | Secure wipe and removal from user | Auto / IT | MDM wipe, de-assign from user, keep in zero-touch for redeployment. Wipe certificate stored. |
| 10 | Disposition: back to stock, repair, or retire / recycle | IT Logistics | Retired devices get a disposal certificate from the recycler. |
| 11 | Case closed | Auto | Offboarding checklist complete; audit record final. |

### 4.3 Other workflows

| Workflow | Summary |
|---|---|
| **Hardware refresh** | Devices reaching age limit (e.g. 3–4 years) or warranty end are flagged. New device is shipped first, data moved, old device returned through the offboarding return steps (4.2 steps 3–10). |
| **Break / fix replacement** | Employee raises a ticket. IT does remote troubleshooting; if hardware failure, an **advance replacement** is shipped, and the faulty one is returned and sent to vendor repair (RMA). |
| **Lost or stolen** | Employee reports in portal (or IT marks it). Immediate remote lock + show message on screen, location request if enabled, then wipe. Police report number captured. Replacement shipped. |
| **Policy change** | Security changes a baseline → approval → pilot group → staged rollout → compliance report. Rollback available. |
| **Temporary policy exception** | Employee asks e.g. "enable USB storage for 2 days for a client file". Manager + Security approve. The portal applies an exception and **removes it automatically** when it expires. |
| **Accessory requests** | Monitor, headset, dock, etc. Approval by manager, shipped and tracked like laptops (lower-value items may not need return). |
| **Address change / relocation** | Employee updates shipping address; country change flags different policies, tax, export rules. |

## 5. Feature list by module

### 5.1 Dashboard
- Fleet overview: total devices, by state, by OS, by country.
- Compliance score; non-compliant devices list.
- Open onboarding / offboarding cases, shipments in transit, overdue returns.
- Alerts: devices not seen in N days, encryption off, policy failures, warranty expiring.

### 5.2 Asset inventory (CMDB-lite)
- Asset record: serial, asset tag, model, specs, purchase date, cost, vendor, warranty,
  assigned user, location / ship address, state, condition, full history.
- Accessories and peripherals tracked too, linked to the main device.
- Bulk import (CSV, vendor order files) and barcode/QR scan at warehouse.
- Depreciation and book value for finance.
- Stock levels per model and per warehouse, with low-stock alerts.

### 5.3 Device management (through MDM)
- Live device data: OS version, last check-in, disk encryption state, installed apps,
  storage, battery health, logged-in user.
- **Remote actions:** lock, restart, wipe, retire, rotate local admin password (LAPS),
  sync policies, collect logs, show message on screen, locate (where legal).
- Device groups: static and dynamic (by department, country, role, OS).

### 5.4 Policy & device control (the restriction engine)

Policies are built as **baselines** attached to groups, with exceptions on top. Each control
says which enforcement provider applies it on each OS.

| Control | Options | Windows mechanism | macOS mechanism |
|---|---|---|---|
| USB mass storage | Allow / Read-only / Block / Allow only approved device IDs | Defender device control, Device Installation policies | Endpoint security agent (native MDM is limited) |
| Other USB / serial (COM) ports | Block by device class; allow keyboards, mice, docks, headsets | Device Installation restrictions by class/ID | Endpoint security agent |
| Camera | Allow / Block | Policy CSP `Camera/AllowCamera` | Restrictions payload (camera) |
| Microphone | Allow / Block per app | Privacy CSP | Privacy preferences (PPPC) per app |
| Bluetooth | Allow / Block / Allow audio only | Bluetooth CSP | Restrictions + agent |
| Wi-Fi / network | Push corporate Wi-Fi, block hotspots, force always-on VPN | Wi-Fi & VPN profiles | Wi-Fi & VPN payloads |
| Screen capture / recording | Block for sensitive groups | DLP agent | Restrictions payload |
| Printing | Allow / Block / Approved printers only | Printer policies | Printing payload |
| Admin rights | Standard user only; just-in-time admin on approval | LAPS + Endpoint Privilege Management | Managed admin / privilege tool |
| Disk encryption | Enforced; recovery key escrowed in portal/MDM | BitLocker | FileVault |
| OS updates | Update rings, deadline, forced restart window | Windows Update for Business | Software update enforcement |
| Firewall & antivirus | Enforced on, tamper protection | Defender | Firewall payload + EDR |
| Application control | Allow-list / block-list, required apps | App deployment + WDAC / AppLocker | App deployment + agent |
| Browser | Managed browser settings, extension allow-list | Edge/Chrome policies | Edge/Chrome policies |
| Login / screen lock | Password rules, auto-lock time, biometric | Device lock CSP | Passcode payload |
| Cloud storage / data loss | Block personal cloud sync, USB copy of labelled files | Purview DLP | Purview DLP / agent |

Policy features:
- Baselines per role (e.g. Finance = strictest, Engineering = USB allowed for approved
  hardware keys).
- **Exceptions** with requester, reason, approvers, start and expiry; auto-revoke.
- Simulation / "what-if": which devices would change before publishing.
- Version history, diff between versions, and rollback.
- Drift detection: alert when a device's actual state differs from its assigned policy.

### 5.5 Compliance & security
- Compliance rules (encryption, OS version, agent running, not jail-broken, policy applied).
- Feeds **conditional access**: non-compliant devices lose access to company apps.
- Reports for audit frameworks (ISO 27001, SOC 2): encryption coverage, patch level,
  device inventory accuracy.
- Vulnerability view (from the EDR/patching tool) by device.

### 5.6 Software & patch management
- App catalogue: required apps (auto-installed) and optional apps (self-service install).
- License tracking: seats used vs purchased; reclaim on offboarding.
- Patch status per device and deployment rings.

### 5.7 Logistics & shipping
- Courier integration: rate quotes, label creation, pickup booking, tracking webhooks.
- Return kit management (boxes, labels) and return-to-warehouse routing per region.
- Address validation; international shipping documents (customs, export control,
  battery shipping rules).
- Support for multiple warehouses or a third-party logistics (3PL) / device-as-a-service
  partner.
- SLA tracking: dispatch time, transit time, delivered before start date.

### 5.8 Procurement & vendors
- Purchase requests with approval chain and budget per department.
- Vendor catalogue with standard models and prices.
- Warranty and RMA (repair) tracking with the vendor.

### 5.9 Employee self-service portal
- "My devices": what I hold, warranty, my return obligations.
- Delivery and return tracking.
- Request: accessory, software, policy exception, replacement.
- Report lost/stolen (one click → device locked).
- Raise a support ticket; knowledge base / first-boot guides.
- Digital acknowledgements (acceptable-use policy, equipment receipt).

### 5.10 Ticketing / service desk
- Built-in light ticketing, **or** integration with an existing tool (Jira Service
  Management, ServiceNow, Freshservice) — to decide.
- Tickets linked to the device and the user.

### 5.11 Workflow & automation engine
- Configurable onboarding/offboarding checklists per country / role.
- Approval chains, SLAs, reminders and escalations.
- Event triggers (HR change, device non-compliant, shipment delivered) → actions.

### 5.12 Notifications
- Email, Slack / Microsoft Teams, SMS (for delivery and return reminders).
- Templates per event and language.

### 5.13 Reporting & analytics
- Fleet age, cost per employee, spend by department.
- Time-to-productive for new hires; return rate and recovery losses.
- Policy compliance trends; exception counts.
- Scheduled exports (CSV / PDF) and an API for BI tools.

### 5.14 Audit log
- Every admin action, remote command, policy change, approval and state change.
- Who, what, when, from where (IP), before/after values.
- Tamper-evident and exportable; retention configurable.

### 5.15 Administration & integrations
- SSO (Entra ID / Okta / Google), SCIM for user sync.
- HR system (Workday, BambooHR, HiBob, …) for joiner / mover / leaver events.
- MDM (Intune, Jamf, Kandji), EDR/DLP agent, courier/3PL, ticketing, finance/ERP.
- Public REST API and outgoing webhooks.

## 6. Non-functional requirements

| Area | Requirement |
|---|---|
| Security | SSO + MFA, RBAC with least privilege, sensitive actions (wipe, policy publish) need a second approver or re-authentication. Secrets in a vault. Encryption in transit and at rest. |
| Privacy | Remote employees work from home, so location tracking, camera and screen data are sensitive. Locate only on lost/stolen; no camera/screen monitoring features. Show employees what is collected. Check local employment and data-protection law (GDPR, works councils) per country. |
| Reliability | Remote lock/wipe and access revocation must work even if one integration is slow: queued commands with retries and status tracking. |
| Scale | Designed for thousands of devices, multiple countries and warehouses. |
| Multi-region | Data residency if needed (EU / US). Time zones and languages in notifications. |
| Auditability | Full immutable audit trail; reports ready for ISO 27001 / SOC 2 evidence. |

## 7. High-level architecture (for later detailed design)

```
             Employees / IT / HR (browser, SSO)
                          │
                ┌─────────▼──────────┐
                │   Web portal (SPA) │
                └─────────┬──────────┘
                          │ REST / GraphQL
                ┌─────────▼──────────────────────────────┐
                │  Backend API                            │
                │  ├ Assets & inventory                   │
                │  ├ Workflow / case engine (on/offboard) │
                │  ├ Policy service (baselines, exceptions)│
                │  ├ Logistics service                    │
                │  ├ Notification service                 │
                │  └ Audit log                            │
                └───┬────────┬─────────┬─────────┬────────┘
                    │        │         │         │   job queue + workers
          ┌─────────▼┐ ┌─────▼────┐ ┌──▼──────┐ ┌▼────────────┐
          │ MDM APIs │ │ EDR/DLP  │ │ Courier │ │ HR / SSO /  │
          │ Intune / │ │ agent    │ │ / 3PL   │ │ Ticketing   │
          │ Jamf     │ │ API      │ │ APIs    │ │ APIs        │
          └────┬─────┘ └────┬─────┘ └─────────┘ └─────────────┘
               ▼            ▼
           Employee laptops (Windows / macOS)
```

Each external system sits behind a **connector interface** so that, for example, Intune and
Jamf (or two couriers) can be swapped or used together.

## 8. Suggested phasing

| Phase | Scope |
|---|---|
| **MVP (1)** | SSO + RBAC, asset inventory, manual onboarding/offboarding cases with checklists, shipping tracking (manual tracking numbers), employee "My devices", audit log, basic dashboard. |
| **Phase 2** | MDM integration (read device data, remote lock/wipe), policy baselines for camera / USB / Bluetooth / encryption, compliance view, courier API, notifications. |
| **Phase 3** | HR system automation, policy exceptions with auto-expiry, procurement & stock, refresh and RMA workflows, ticketing integration. |
| **Phase 4** | Advanced analytics, DLP / app control, multi-region data residency, public API & webhooks. |

## 9. What the portal deliberately does not do

- It does not build its own device agent (uses MDM + EDR/DLP vendors instead).
- It does not monitor employees (no screenshots, webcam or keystroke capture).
- It is not a full ITSM tool; it integrates with one if the company already has one.

## 10. Open questions for review

1. Which OSes are in scope? Windows + macOS only, or Linux too?
2. Is there an existing MDM (Intune, Jamf, …) and EDR/DLP tool, or is the choice open?
3. Which HR system and identity provider are used?
4. How many employees / devices, and in how many countries?
5. Shipping model: own warehouse, a 3PL partner, or direct ship from vendor?
6. Existing ticketing / ITSM tool to integrate with, or build a light one?
7. Policy for non-returned devices (payroll deduction, legal recovery, write-off)?
8. Should employees be able to request temporary exceptions (e.g. USB) themselves?
9. Data residency or compliance targets (SOC 2, ISO 27001, GDPR)?
10. Preferred tech stack and hosting (e.g. Azure because of Intune / Entra ID)?
