# Feasibility: Camera-Proctored Microsoft 365 (Office) Forms

## 1. Requirement

1. An author creates questions in **Microsoft Forms** (Office 365).
2. A respondent opens the form link.
3. On open, the **camera is switched on automatically, after the respondent consents**,
   and stays on while they answer.

**Context:** internal use only. Employees in our own Microsoft 365 tenant take skill-set
evaluations. This is not an external or public exam. See §2a for what that changes.

## 2. Verdict

**Achievable, but not inside Microsoft Forms itself.** It needs a small wrapper web app
("the proctor page") that the respondent opens instead of the raw Forms link.

| Part of the requirement | Feasible? | How |
|---|---|---|
| Author creates questions in Microsoft Forms | Yes | Unchanged: authors keep using forms.office.com. |
| Camera turns on when the link is opened | Yes, with one click | The proctor page asks for consent as soon as it loads. The browser always shows its own "Allow camera" prompt. No website can skip that prompt. |
| Camera stays on while the form is answered | Yes | The form is embedded in the proctor page, which owns the camera stream. |
| Camera code running *inside* Microsoft Forms | **No** | Forms does not allow custom scripts, add-ins or camera access. Microsoft hosts the page, so it cannot be modified. |
| Proctor page reads answers or knows when "Submit" is clicked | **No** (directly) | The embedded form is on another origin (forms.office.com), so the browser blocks the wrapper from reading it. Responses are linked afterwards (see §5). |

"Automatically" therefore means: **the consent prompt appears automatically, and the camera
starts the moment the respondent agrees.** Starting the camera with no user action is
impossible in every modern browser, and it would also defeat the consent requirement.

## 2a. What changes because this is internal (team skill evaluation)

| Area | Effect |
|---|---|
| Identity | Easier. Everyone is in the same Entra ID tenant. Use *"Only people in my organization can respond"* and *"Record name"*, plus single sign-on in the proctor page. |
| Sign-in inside the iframe | Lower risk than for external users, because people are already signed in to Microsoft 365 in the browser. It still needs checking in the proof of concept: Safari and strict third-party-cookie settings can break it. Managed Edge or Chrome via Intune can be configured to allow it. |
| Hosting | Can be internal-only, for example Azure Static Web Apps limited to the tenant, or a SharePoint/Teams tab. Teams tabs can request the camera through the Teams JS SDK device permissions. |
| Stakes | Lower than a certification exam, so a lighter build is enough: periodic snapshots plus a tab-switch log, no video and no face analysis. |
| Employee monitoring (the main new constraint) | Camera monitoring of employees is regulated more strictly than monitoring students. Consent from an employee is often **not considered "freely given"** (GDPR, because of the employer–employee power imbalance), so do not rely on consent alone. Involve HR, Legal and the DPO, and the works council or union where one exists (mandatory in e.g. Germany). Document the legitimate purpose. Offer a non-camera alternative with no penalty. State in writing that the recordings are not used for performance or disciplinary decisions beyond the evaluation itself. |
| Is a camera needed at all? | Worth asking. For internal skill mapping, the honesty-statement + time-limit + randomised-question approach is often enough, and it avoids the monitoring issues. Recommendation: make the camera **optional or limited to specific high-stakes evaluations**. |

## 3. Why Forms alone cannot do it

- Microsoft Forms has no extension or add-in model and no custom-JavaScript question type.
- Question types are fixed (choice, text, rating, date, ranking, Likert, file upload, NPS).
  None of them use the camera.
- Browsers only grant camera access (`navigator.mediaDevices.getUserMedia`) to code running
  on a page, over HTTPS, after the user allows it. We cannot add code to a Forms page.

## 4. Proposed architecture

```
Respondent
   │  opens https://proctor.<your-domain>/exam/<examId>
   ▼
┌─────────────────────────── Proctor page (our web app) ───────────────────────────┐
│ 1. Sign in with Microsoft Entra ID (MSAL)  → we know who the respondent is         │
│ 2. Consent screen (what is recorded, why, retention, contact)  → "I agree"         │
│ 3. getUserMedia() → browser prompt "Allow camera?" → live self-view in a corner    │
│ 4. Embed the Microsoft Form in an <iframe> (Forms "Embed" share option)            │
│ 5. While open: snapshot every N seconds and/or record video, log tab switches,     │
│    camera lost, no face in view → upload to storage with session id + timestamps    │
└────────────────────────────────────────────────────────────────────────────────────┘
        │ snapshots / events                         │ answers go straight to Microsoft
        ▼                                            ▼
  Azure Blob Storage / SharePoint            Microsoft Forms responses
        │                                            │
        └──────────── Power Automate flow: "When a new response is submitted"
                      → match response to proctor session → flag mismatches
```

### Components

| Component | Suggested technology | Notes |
|---|---|---|
| Proctor page | Static SPA (plain JS/React) on Azure Static Web Apps or any HTTPS host | HTTPS is mandatory for camera access. |
| Sign-in | MSAL.js against the same Entra ID tenant as the Forms | Gives a trusted identity to match against the Forms responder email. |
| Camera | `getUserMedia` + `<canvas>` snapshots, or `MediaRecorder` for video | Snapshots use far less storage and bandwidth than video. |
| Storage | Azure Blob (SAS upload) or SharePoint via Microsoft Graph | Apply a retention policy, e.g. auto-delete after 30–90 days. |
| Linking responses | Power Automate Forms connector | Trigger "When a new response is submitted", then "Get response details". |
| Optional face detection | In-browser only (e.g. MediaPipe face detection) | Flags "no face / multiple faces". Do **not** do face *recognition* without legal sign-off. |

## 5. Linking each response to its camera session

The wrapper cannot read the iframe, so the two are matched afterwards:

1. **Identity + time (simplest):** set the form to *"Only people in my organization can
   respond"* and *"Record name"*. Match the responder email and submit time from Power Automate
   with the signed-in user and session window in the proctor log.
2. **Session token (stronger):** add a short-text question "Session code". Open the form
   using a **pre-filled link** that puts the proctor session id into that question.
   Power Automate then checks the code against the proctor backend. Responses with no valid
   code, or whose code belongs to someone else, are flagged.

## 6. Risks and limitations

| Risk | Impact | Mitigation |
|---|---|---|
| The respondent finds the raw Forms URL (it is visible in the iframe source) and answers without the camera | High | Flag every response without a matching session (§5). Treat those as invalid. Share only the proctor link. |
| Org-restricted forms need Microsoft sign-in **inside the iframe**. Browsers that block third-party cookies (Safari, some Chrome/Edge settings) can break this. | Medium–High | **Validate in the proof of concept first.** Fallback: open the form in a second window or tab while the proctor tab keeps the camera running. Tab-switch detection is then weaker. |
| Respondent denies or revokes the camera, or covers it | Medium | Block the form until the camera is live. Hide or blur the iframe if the stream ends. Log black or face-less frames. |
| Tab-switch detection is noisy (clicking into the iframe fires `blur`) | Low | Use `document.visibilitychange`, not `window.blur`. |
| Mobile browsers (iOS Safari) restrict background camera use and iframes | Medium | Support desktop Chrome or Edge first. Test iOS separately or state that it is unsupported. |
| Wrapper is not a locked-down browser (respondent can use a second device, notes, etc.) | Inherent | Camera monitoring deters and gives evidence. It does not prevent cheating. For high-stakes exams, add Safe Exam Browser or a commercial proctoring tool. |
| Microsoft changes Forms embedding or sign-in behaviour | Low–Medium | Keep the wrapper thin. Monitor Microsoft 365 message centre. |

## 7. Privacy, consent and compliance (must-have, not optional)

- Show an explicit consent screen **before** the browser prompt. Say what is captured
  (images/video, tab events), why, who can see it, how long it is kept, and how to object.
- Provide an alternative route for people who do not consent (for example in-person
  assessment).
- Complete a DPIA (GDPR or local equivalent). Video of a person's face is personal data.
  If you do face analysis, it may be biometric data (GDPR Art. 9, Illinois BIPA, India DPDP Act, etc.).
- Keep the minimum data needed. Store it in the tenant's region, encrypt it, restrict access, and auto-delete it.
- Store the consent record (user, timestamp, consent text version) with the session.

## 8. Alternatives considered

| Option | Pros | Cons |
|---|---|---|
| **A. Wrapper + embedded Microsoft Form (recommended)** | Authors keep using Forms. Small build. | Responses are linked afterwards, not live. Iframe sign-in risk. |
| B. Build our own quiz app (questions authored elsewhere) | Full control: camera, timer, submit gating, anti-tamper | Authors lose the Forms UI, and responses have to be stored ourselves. Bigger build. |
| C. Commercial proctoring (Proctorio, Respondus, Honorlock, etc.) | Mature, includes lockdown browser | Licence cost. Usually built for an LMS (Moodle, Canvas), not Microsoft Forms. |
| D. Microsoft Teams meeting with cameras on during the form | No development | Manual, does not scale, no per-response evidence. |

## 9. Suggested plan

| Phase | Scope | Estimate |
|---|---|---|
| 0. Proof of concept | Static page: consent → camera preview → embedded org-only form. Test Chrome, Edge, Safari and Firefox, including sign-in inside the iframe. | 2–3 days |
| 1. MVP | Entra ID sign-in, periodic snapshots to Blob, tab-visibility log, pre-filled session code, Power Automate matching and flagging | 2–3 weeks |
| 2. Hardening | Admin review screen (sessions, snapshots, flags), retention jobs, face-presence detection, accessibility, DPIA sign-off | 2–4 weeks |

**Go/no-go gate:** if the phase 0 test shows org-only forms cannot sign in inside an iframe on
your target browsers, switch to the second-window fallback (§6) or option B.

## 10. Summary

- Questions are created in Microsoft Forms exactly as today.
- Respondents open the **proctor link**. A consent screen and the browser's camera prompt
  appear immediately. After "Allow", the camera runs and the form shows in the same page.
- The camera cannot run *inside* Forms or start without the user's click. Both are
  platform restrictions, not something code can work around.
