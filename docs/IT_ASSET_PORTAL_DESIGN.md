# IT Device & Asset Management Portal: Workflows and Features

Status: **draft for review (v4: any OS, Asia, HR bundle, last-day lock and legal escalation)**.
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
- **Any laptop OS is allowed**: Windows (Home or Pro), macOS, Linux. Each OS is managed as
  far as it technically allows (see §1.2).
- **Employees live in Asian countries only** for now (e.g. India, Philippines, Indonesia,
  Vietnam, Malaysia, Singapore). Shipping, privacy law and legal recovery follow the
  employee's country.
- **No HR tool exists.** The portal includes an **HR add-on bundle** (§5.17) that holds
  the employee record, joining date and last working day.
- **On the employee's last working day the laptop is locked and cannot be used.** If it is
  not returned, the organisation **registers a legal case in the jurisdiction of the
  employee's address** (§4.3, §4.4).

### 1.1 Key design principle: the portal is tool-agnostic

The portal describes **what** IT wants ("block USB drives for the Finance group", "lock
this laptop", "ship to this address"). It does not depend on **how** a specific tool does
it. Every outside system sits behind a **provider interface** (adapter), so a tool can be
replaced without changing the portal, its workflows or its data:

| Provider interface | What the portal asks of it | Examples of implementations (choose later) |
|---|---|---|
| **Device provider** | Enrol device, read inventory, apply policy, run remote action (lock, wipe, restart, message), report compliance | Free/open-source device management server, our own lightweight agent, or a paid tool if the budget changes later |
| **Identity provider** | Sign-in, MFA, list users and groups, disable account, revoke sessions | Google Workspace, Microsoft 365 (any plan), Okta, Keycloak |
| **Shipping provider** | Rates, label, pickup, tracking updates, proof of delivery | Local courier APIs per country, or manual tracking numbers |
| **E-signature** | Sign custody agreement and other documents | E-sign service, or OTP-based signing in the portal |
| **HR source** | New hires, leavers, last working day, changes (role, manager, address) | Our HR add-on bundle (§5.17), CSV upload, or an external HR tool later |
| **Notification channel** | Send messages | Email, SMS, WhatsApp Business, Slack, Microsoft Teams |

Each provider declares what it supports (for example "camera block on macOS: yes; USB
port block on macOS: no"). The portal shows that clearly and hides or greys out options a
provider cannot enforce, so IT never thinks a rule is applied when it is not.

### 1.2 OS support: what can be controlled on each OS

Because there is no Intune/Jamf, the device provider is something the organisation runs
itself: a device-management server using the management features built into the OS, plus a
**small agent** (background service) installed on every laptop. The agent is what makes
"any OS" possible: where the OS has no built-in management (Windows Home, Linux), the agent
does the work.

| OS | How it is managed | Strength of control | Notes |
|---|---|---|---|
| **macOS** | Built-in Apple management + agent | **Strong** | Firmware-level lock with PIN; Apple Business Manager (free) ties the Mac to the organisation even after a reinstall. |
| **Windows Pro / Business / Enterprise** | Built-in Windows management + agent | **Strong** | BitLocker encryption managed; lock via BitLocker recovery (§4.3). |
| **Windows Home** | Agent only (Home has no built-in management client) | **Medium** | Agent runs as a system service and applies settings directly. Encryption only if the hardware supports "Device Encryption". Recommend upgrading to Pro for new purchases. |
| **Linux** (Ubuntu, Fedora, etc.) | Agent only | **Medium–strong** | Full-disk encryption (LUKS) set up by IT before shipping; agent controls USB, camera, accounts. |
| **ChromeOS** | Needs Google's paid Chrome Enterprise Upgrade | Not supported for now | Listed so procurement avoids buying Chromebooks. |

Other facts that hold whatever tool is chosen:

| Item | Detail |
|---|---|
| Set up before shipping | Windows' automatic first-boot setup (Autopilot) needs Intune, and Linux has none. So IT **enrols and sets up every laptop before shipping it**, for every OS. |
| Firmware (BIOS/UEFI) password | Set by IT on every laptop before shipping, with boot from USB disabled. Without it, someone could reinstall the OS and bypass the lock. On Macs, Apple Business Manager + Activation Lock does this job. |
| Disk encryption mandatory | Required on every laptop. It is what makes the last-day lock real (§4.3). Laptops that cannot be encrypted are flagged and not issued, or issued only with IT approval. |
| Apple push certificate | Managing Macs needs an Apple push certificate. The organisation creates it for free and renews it every year (the portal reminds IT before expiry). |
| Agent removal | Employees get **standard (non-admin) accounts**, so they cannot remove the agent. The portal alerts when a device stops reporting. |

## 2. Users and roles

| Role | What they do in the portal |
|---|---|
| **Super Admin** | Portal settings, provider connections, role assignment. |
| **IT Admin** | Devices, policies, remote actions, onboarding/offboarding cases. |
| **Logistics / Asset Manager** | Stock, purchasing, shipping, returns, repairs, disposal. |
| **Security / Compliance** | Policy sets, exception approvals, compliance reports (mostly read-only). |
| **HR** | Employee records, address verification, joiners and leavers, last working day (HR bundle). |
| **Legal** | Non-return cases, notices, lawyers, evidence packs. |
| **External lawyer** | Limited access to the cases assigned to them. |
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

 Offboarding: Active ─► Lock Scheduled ─► Locked (LWD) ─► Return Shipped ─► Received ...
 Side states: Lost / Stolen (locked, then wiped) · Not Returned ─► Legal Case ─► Recovered / Written off
```

Devices of every OS are **enrolled and configured before shipping** (§1.2).

Every state change is time-stamped, tied to a person, and written to the audit log.

## 4. Core workflows

### 4.1 Onboarding: ship a laptop to a new hire

| # | Step | Owner | System action |
|---|---|---|---|
| 1 | New hire added (start date, role, address, manager) | HR | Entered in the HR bundle (or imported from CSV). Creates an **Onboarding Case**. |
| 2 | Device profile picked from role (e.g. "Developer – MacBook Air", "Sales – Windows 14″", "Support – Ubuntu") | Auto, IT can change | Profile defines hardware model, software bundle and **policy set**. |
| 3 | Stock check | Auto | Device is reserved from stock. If none, a purchase request is created. |
| 4 | Laptop enrolled and set up before shipping | IT / warehouse | Enrolled in the device server and agent installed (any OS). **Disk encryption on, IT recovery key stored, firmware password set, USB boot disabled.** Mac: Apple Business Manager where available. Policies and apps applied. Device assigned to the employee. |
| 5 | Pre-ship check passes (encrypted, policies applied, agent healthy) | Auto | Device cannot be marked "Ready to ship" until it passes. |
| 6 | Shipping label booked; ship-by date set from start date and courier lead time | Logistics | Shipping provider books the label, or a tracking number is typed in by hand. |
| 7 | Employee gets a welcome email/SMS with tracking link and a "first sign-in" guide | Auto | Address must already be **verified by HR** (ID + address proof, §5.17) before dispatch. |
| 8 | Delivery confirmed **with proof** (OTP to employee's phone, signature or photo) | Courier webhook / manual | Device state → Delivered. Proof of delivery stored (used as evidence if ever needed, §4.4). |
| 9 | Employee signs in with the account IT created and sets their own password | Employee | Device checks in. Final user-specific settings (Wi-Fi, VPN, email) are applied. |
| 10 | Employee signs the **asset custody agreement** and acceptable-use policy | Employee | E-signed. Agreement states asset value, return obligation, that the laptop **will be locked on the LWD**, and that legal action may follow non-return. Device stays limited until signed. |
| 11 | Case closed | Auto | Time from hire to working laptop recorded. |

Exceptions handled: address change, failed delivery, damaged on arrival (start replacement),
device fails to check in after delivery (alert IT), start date moved.

### 4.2 Offboarding: collect the laptop from a leaving employee

| # | Step | Owner | System action |
|---|---|---|---|
| 1 | Resignation or termination recorded with **last working day (LWD)** | HR (HR bundle) | **Offboarding Case** created listing all assets held by that person. Employee's verified address and time zone attached. |
| 2 | Lock scheduled | Auto | A **signed lock order** for LWD (end of day, employee's local time) is sent to the laptop straight away, so it fires even if the laptop is offline on that day (§4.3). |
| 3 | Return pickup booked for LWD or LWD + 1 | Logistics | Shipping provider. Employee gets pickup slot and packing instructions. |
| 4 | Reminders before LWD | Auto | LWD − 7, − 3, − 1: "Back up anything you need; the laptop will be locked on <date>; pickup on <date>". |
| 5 | **LWD: accounts disabled and laptop locked** | Auto | Identity provider disables accounts and ends sessions. Laptop locks and cannot be used (§4.3). Lock screen shows return instructions and IT contact. |
| 6 | Return tracked | Courier webhook / manual | Reminders at LWD + 3 and + 7. |
| 7 | Not returned by deadline | Auto | **Legal escalation** starts (§4.4). |
| 8 | Device received | Logistics | Scan serial. State → Received. Any legal case is closed or withdrawn (§4.4). |
| 9 | Inspection: condition grade, accessories, damage photos | Logistics | Checklist stored on the asset. Damage recorded for recovery if policy allows. |
| 10 | Unlock with IT key, secure wipe, device unassigned | IT | Wipe report stored. Device stays registered for reuse. |
| 11 | Asset clearance given | Auto | HR bundle marks the "IT assets returned" item of exit clearance as done. |
| 12 | Case closed | Auto | Audit record final. |

Special cases:
- **Termination with immediate effect or absconding**: the lock runs **at once**, not at
  end of LWD.
- **LWD changes** (notice period extended or shortened): the lock order is replaced
  automatically. Only HR can change LWD, and the change is logged.
- **Device returned before LWD**: lock order cancelled; device handled from step 8.

### 4.3 Last-working-day lock (laptop cannot be used)

**Goal:** from the end of the LWD, the laptop cannot be used by anyone except IT, whether it
is online or offline, and even if the OS is reinstalled.

**How it is triggered (three layers, so it works even if the laptop stays offline):**

| Layer | How it works |
|---|---|
| 1. Scheduled lock order | When LWD is set, the portal sends a signed lock order with the date and time. The agent stores it and locks the laptop at that time even with no internet. The agent uses trusted time (last server time + its own elapsed-time counter), so changing the PC clock does not delay it. |
| 2. Live lock command | At LWD the portal also sends a lock command through the device provider. Whichever arrives first wins. |
| 3. Offline limit ("check-in lease") | During the notice period the laptop must contact the server at least every 24–48 hours (configurable). If it does not, it locks itself. For other employees a longer limit (e.g. 14–30 days) can be set as a protection against loss. |

**How the laptop is locked (per OS):**

| OS | Lock method | Result |
|---|---|---|
| macOS | Device-lock command with a 6-digit PIN known only to IT | Mac restarts to a firmware lock screen and cannot start without the PIN. Activation Lock / Apple Business Manager stop reuse after a reinstall. |
| Windows Pro / Enterprise | Agent removes the normal BitLocker unlock (TPM) and restarts; disables the employee's account | Laptop asks for the **BitLocker recovery key** at start-up. Only IT holds it (stored in the portal). Firmware password stops booting from USB. |
| Windows Home | With Device Encryption: same as above. Without it: agent disables all user accounts, shows a lock screen, firmware password blocks reinstall | Strong with encryption; weaker without it (§1.2). |
| Linux | Agent removes the employee's disk-encryption key slot (IT key slot kept), disables accounts and restarts | Disk cannot be unlocked without the IT passphrase. |

**Unlocking:** only after the laptop is received (or an approved exception, e.g. LWD
cancelled). Recovery keys and PINs are shown to IT only, with re-authentication, and every
view is logged.

**Safeguards:**
- The lock order can only be created by an approved offboarding case (HR + IT), never by a
  single person editing a device.
- Employees are told in the custody agreement (signed at onboarding) and in the reminders
  that the laptop will be locked on their LWD, and that personal data must not be stored on
  it.
- A wrongly locked laptop (e.g. LWD entered by mistake) can be unlocked remotely by IT
  within minutes once it is online.

### 4.4 Non-return: legal escalation in the employee's jurisdiction

If the laptop is not returned, the portal builds the case file and tracks the legal process.
Timelines are configurable per country; example defaults:

| When | Step | Owner | System action |
|---|---|---|---|
| LWD + 7 | Final reminder by email, SMS and phone call | Logistics / HR | Logged with delivery receipts. |
| LWD + 10 | **Formal demand notice** | HR / Legal | Generated from a country template; sent by email **and** registered post / tracked courier to the verified address. Proof of dispatch and delivery stored. |
| LWD + 25 | **Legal notice from counsel** | Legal / external lawyer | Lawyer assigned by region. Notice and proof of delivery stored. |
| LWD + 40 | **Decision to file case** | Legal (approval) | Portal produces the **evidence pack** (below) and shows the **jurisdiction** for filing. |
| After filing | Case tracking | Legal | Case number, police station / court, lawyer, hearing dates, documents, status, costs. |
| Device recovered or settled | Close case | Legal | Withdrawal / settlement recorded; device goes to inspection (§4.2 step 8). |

**Evidence pack** (generated as one PDF + attachments, with audit references):
- Signed **asset custody agreement** (with return obligation, lock notice and asset value).
- Employee identity and **verified address** at the time of LWD (§5.17).
- **Proof of delivery** of the laptop at onboarding (courier record with OTP / signature /
  photo).
- Asset details: serial number, model, purchase invoice, current value.
- Offboarding record: LWD, pickup bookings, all reminders and notices with delivery proof.
- Device records: last check-in time, lock confirmation, and location **only if** lawfully
  collected after the device was reported as not returned.

**Jurisdiction:** the portal maps the employee's verified address (country → state/province
→ district/city → postal code) to the competent police station and court, using a
jurisdiction table that Legal maintains per country. The address used is the one on file
at LWD, so employees must keep their address up to date (§5.17).

**Country differences:** what can be filed (police complaint, criminal case, civil recovery
suit) and the notice periods differ by country. Example: in India, non-return of entrusted
company property is commonly pursued as criminal breach of trust and/or a civil recovery
suit. Each country's templates and steps must be **confirmed by local counsel**; the portal
holds them as configuration and does not give legal advice.

**Alternatives before court** (configurable, subject to local labour law): hold relieving /
experience letter, recover the device value from the final settlement, or offer a "buy-out"
at depreciated value. Whether these are lawful varies by country, so each is off by default
and enabled per country by Legal.

### 4.5 Other workflows

| Workflow | Summary |
|---|---|
| **Hardware refresh** | Devices reaching age limit or warranty end are flagged. New device shipped first, then the old one is returned through §4.2 steps 3 and 6–10. |
| **Break / fix replacement** | Employee raises a ticket. If hardware failure: replacement shipped, faulty one returned and sent to vendor repair. |
| **Lost or stolen** | Employee reports in portal (one click), or IT marks it. Immediate remote lock + on-screen message, then wipe. Replacement shipped. |
| **Policy change** | IT changes a policy set → preview of affected devices → test on a few devices → roll out → compliance report. Rollback available. |
| **Temporary policy exception** | Employee asks e.g. "enable USB drive for 2 days". Manager approves. The portal applies it and **removes it automatically** when it expires. |
| **Accessory requests** | Monitor, headset, dock, etc. Approved by manager, shipped and tracked. |
| **Address change / relocation** | Employee requests address change with proof; HR approves. Country change flags different shipping, privacy and legal rules. |
| **First-time setup** | Enable HR bundle and import employees → connect identity provider → connect device provider (incl. Apple push certificate) → choose default policy sets → import existing laptops (CSV) and enrol them in place with a link. |

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
OS can enforce it, for reference. On Windows Home and Linux the agent applies the same rules
directly, because there is no built-in management client.

| Control | Options | Windows (how it can be enforced) | macOS (how it can be enforced) | Linux (how it can be enforced) |
|---|---|---|---|---|
| USB storage | Allow / Read-only / Block / Allow approved device IDs | Built-in management (Pro) or agent (Home): removable storage and device-install policies | Agent (native MDM is limited, see §1.2) | USBGuard |
| Other USB / serial (COM) ports | Block by device class; always allow keyboards, mice, docks, headsets | MDM: device-install restrictions by class/ID; agent for detail | Agent | USBGuard / udev rules |
| Camera | Allow / Block | MDM: camera policy | MDM: restrictions profile | Agent: disable camera driver |
| Microphone | Allow / Block per app | MDM: privacy policy | MDM: privacy (PPPC) profile | Agent |
| Bluetooth | Allow / Block / Audio only | MDM: Bluetooth policy | MDM profile + agent | Agent |
| Wi-Fi / VPN | Push company Wi-Fi; VPN settings | MDM: Wi-Fi / VPN profiles | MDM: Wi-Fi / VPN profiles | Agent |
| Printing | Allow / Block | MDM / agent | MDM profile | Agent |
| Admin rights | Standard user only; temporary admin on approval | Agent + local admin password rotation | Agent + managed admin account | Agent (sudo rules) |
| Disk encryption | **Mandatory**; IT recovery key stored in portal | BitLocker via MDM; agent sends recovery key | FileVault via MDM; key escrowed through MDM | Agent check (LUKS) |
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
- Return kit management and return addresses per country.
- Courier choice per country (local couriers usually cover remote areas better).
- Proof of delivery (OTP / signature / photo) required for outbound laptops.
- Address validation; customs documents for international shipping; laptop battery
  shipping rules.
- Ship-by dates and delays visible on the dashboard.

### 5.8 Procurement & vendors
- Purchase requests with simple approval.
- Standard models list with prices.
- Warranty and repair (RMA) tracking.

### 5.9 Employee self-service portal
- "My devices": what I hold, warranty, my return obligations and lock date (if leaving).
- My profile: request address change (with proof), submit resignation (HR bundle).
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
- Email, SMS, WhatsApp Business, Slack, Microsoft Teams (delivery, lock and return reminders).
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
- HR: our HR add-on bundle (§5.17), CSV import; external HR tool connector later.
- Shipping: local courier APIs per country, or manual tracking numbers.
- E-signature: e-sign service or OTP-based in-portal signing.
- Email / SMS / WhatsApp Business / Slack / Teams.
- Portal REST API and webhooks for the organisation's own scripts (later phase).

### 5.17 HR add-on bundle (new)

A light HR module, sold/enabled as an **add-on bundle**. It plugs in through the same
**HR source** interface (§1.1), so a real HR tool can replace it later without changing the
rest of the portal. Payroll, leave and attendance are not part of it for now.

- **Employee master:** employee ID, name, personal and work email, phone, country, time
  zone, department, role, manager, employment type, joining date.
- **Identity and address verification:** government ID and address proof uploaded and
  checked by HR (store only what each country's law allows, e.g. masked ID numbers where
  required). Address changes need new proof and HR approval, and keep full history. This is
  what the legal jurisdiction is based on (§4.4).
- **Joiner workflow:** offer accepted → joining date → creates the IT onboarding case
  (§4.1).
- **Leaver workflow:** resignation submitted by employee (or termination entered by HR) →
  manager / HR approval → notice period calculated per country/contract → **LWD fixed** →
  creates the IT offboarding case and schedules the lock (§4.2, §4.3).
- **Mover workflow:** role, manager or country change → device policy set and shipping
  address updated.
- **Exit clearance checklist:** IT assets returned, manager handover, finance, HR. Shows
  which items block the relieving letter (per country rules, §4.4).
- **Documents and e-sign:** offer letter, asset custody agreement, acceptable-use policy,
  relieving letter; signed with an e-signature provider or an OTP-based in-portal signature.
- **Emergency / alternate contact** (used for return follow-up).
- **Org chart and directory.**

### 5.18 Legal case management (new)

- Non-return cases with status: *Reminder → Demand notice → Legal notice → Filed →
  Hearing → Recovered / Settled / Written off*.
- Notice templates per country and language; generated with employee and asset data.
- Dispatch tracking for registered post / courier, with proof of delivery attached.
- **Jurisdiction table** per country (postal code / district → police station, court).
- External lawyer directory per region; assignment and access limited to their cases.
- One-click **evidence pack** (§4.4).
- Case timeline, hearing reminders, costs, outcomes, reports (recovery rate, average time).
- Strict access: Legal and named HR only; every view and download logged.

## 6. Non-functional requirements

| Area | Requirement |
|---|---|
| Security | MFA everywhere. RBAC with least privilege. Wipe, lock, unlock-key views and policy publishing need re-authentication (and optionally a second approver). Lock orders and agent updates are **digitally signed** so a fake server cannot lock or unlock laptops. Recovery keys, PINs and ID documents encrypted with separate keys. Penetration test before go-live. |
| Tool independence | No provider-specific fields in the core data model. Swapping a provider must not need changes to workflows, policies or history. |
| Privacy (Asia) | Follow each country's data-protection law, e.g. India DPDP Act 2023, Singapore PDPA, Philippines Data Privacy Act, Indonesia PDP Law, Vietnam Decree 13/2023, Malaysia PDPA. Collect only what is needed; employee notice and consent where required; retention limits for ID documents and legal files. No screenshots, webcam, keystroke or browsing monitoring. Location only after a device is reported lost, stolen or not returned, and only where lawful. |
| Data location | Check per country whether employee or ID data must stay in-country (e.g. Vietnam, Indonesia have localisation rules for some data). Host in an Asian cloud region by default. |
| Evidence quality | Records used in legal cases (signatures, delivery proofs, notices, audit log) must be tamper-evident (hashes, timestamps) and exportable, so they can be used as evidence. |
| Reliability | Lock, wipe and access revocation are queued and retried until confirmed, with clear status ("scheduled", "delivered", "locked"). Scheduled lock works offline (§4.3). |
| Localisation | Employee-facing screens, notices and emails in English plus local languages as needed; local time zones, date formats and phone numbers. |
| Cost | Runs on modest infrastructure; no paid per-device licences required. |
| Scale | Sized for the organisation (hundreds to a few thousand devices), multiple Asian countries. |

## 7. High-level architecture (for later detailed design)

```
     Employees / IT / HR / Managers / Legal (browser, SSO + MFA)
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
        │  ├ Lock service (signed lock orders, keys/PINs) │
        │  ├ Logistics service                            │
        │  ├ HR bundle (add-on)                           │
        │  ├ Legal case management                        │
        │  ├ Ticketing, notifications, audit log          │
        │  └ Job queue + workers (retries, schedules)     │
        └──────────────────────┬─────────────────────────┘
                               │
         ┌──────────── Provider interfaces (adapters) ─────────────────────┐
         │ Device │ Identity │ Shipping │ HR source │ E-sign │ Notification │
         └───┬────┴────┬─────┴────┬─────┴─────┬─────┴───┬────┴──────┬───────┘
             ▼         ▼          ▼           ▼         ▼           ▼
        device mgmt  Google /   local      HR bundle  e-sign /   email / SMS /
        server +     M365 /     couriers   or HR tool OTP sign   WhatsApp / Slack
        agents       Okta       per country
             ▼
       Employee laptops (Windows Home/Pro, macOS, Linux)
```

## 8. Suggested phasing

| Phase | Scope |
|---|---|
| **0. Proof of concept (2–3 weeks)** | Pick the device server and build the agent prototype. Prove on real laptops (macOS, Windows Pro, Windows Home, Ubuntu): enrol, camera block, USB storage block, encryption + key storage, **scheduled offline lock and unlock**, firmware password. |
| **1. MVP** | Sign-in + MFA, roles, asset inventory, **HR bundle basics** (employee master, address verification, joiner/leaver with LWD), onboarding/offboarding cases, custody agreement e-sign, manual tracking numbers, device provider + agent, remote lock/wipe, **LWD scheduled lock**, rules for camera / USB storage / encryption / screen lock, employee "My devices", audit log, dashboard. |
| **2** | **Legal case management + evidence pack + notice templates** (first country: the one with most employees), full policy set (COM ports, Bluetooth, admin rights, apps, updates), exceptions with auto-expiry, compliance view, courier integration, notifications (SMS / WhatsApp), ticketing. |
| **3** | More countries (templates, jurisdiction tables, couriers), exit clearance, procurement & stock, refresh and RMA workflows, reports. |
| **4** | Portal API & webhooks, connector for an external HR tool, extra providers as needed. |

## 9. What the portal deliberately does not do

- It does not monitor employees (no screenshots, webcam or keystroke capture).
- It does not give legal advice; country legal steps are configured with local counsel.
- It is not a full HR system (no payroll, leave or attendance for now) or a full ITSM tool.
- It does not support Chromebooks for now.
- It does not hard-code any device-management, identity, courier or e-sign vendor.

## 10. Open questions for review

1. Which Asian countries are in scope first, and roughly how many employees in each?
2. Who keeps laptop stock and ships them: one central location (cross-border shipping with
   customs and battery rules) or stock / vendor per country (recommended)?
3. Lock time on LWD: end of working day (e.g. 18:00 local), or midnight?
4. Offline limit during notice period: 24 hours, 48 hours, or other?
5. Do you already have external lawyers per country/region, and legal notice templates?
6. Which recovery steps are allowed before court (hold relieving letter, deduct from final
   settlement, buy-out)? To be confirmed per country with counsel.
7. Which identity provider is used today (Google Workspace, Microsoft 365, other)?
8. Is a third-party e-signature service acceptable, or OTP-based signing inside the portal?
9. Preferred tech stack and hosting (cloud region, or on-premises)?
