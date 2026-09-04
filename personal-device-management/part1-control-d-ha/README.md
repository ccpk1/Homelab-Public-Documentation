# Taming the iPhone: Remote Data Cutoff & Bypass Detection via Control D and Home Assistant

> **A note on scope:** this is a full-system build with a custom integration and advanced automation — genuinely more than most people need, and not exactly plug-and-play. It's here to show what becomes possible when you think about device control as a system rather than a handful of rules, and you can take just the pieces you want (only the DNS profile, only the access-mode automation, etc.). That said, I did take time to simplify and organize the programmatic components down to a single user, with structure and standardization, so it's as easy to follow and use as possible for anyone who wants to dig in. The goal isn't the complexity for its own sake — it's a next-level interface for holistic and/or granular control: repeatable, profile-based postures with standard overrides that can be called and monitored.

## **Why iPhones Are Hard (and How This Approaches It)**

Managing personal iOS devices is notoriously frustrating due to the lack of MDM options like corporations have. Commercial filtering and parental control apps are invasive, subscription-heavy, and clunky. Crucially, almost all of them are closed-source and strictly forbid API or programmatic control from external orchestration tools like Home Assistant.

The primary issue: a phone has **multiple connection points**. It's on Wi-Fi at home, then hops to cellular the moment you unplug from the network or walk out the door. If you've invested in your home network — a capable router, access points, firewall rules — that protection only exists while the phone is on that Wi-Fi. A phone can *instantly and silently* bypass all of it by just switching off Wi-Fi, and then it has clear, unrestricted Internet on cellular — governed only by whatever Apple's own settings allow, almost none of it programmatically controllable.

Forcing an "Always-On VPN" back to your home router closes that gap, but it brings its own tradeoffs: tougher to enforce, easily turned off, plus battery drain, a single point of failure, unstable connections, and high latency over cellular.

This architecture takes a different approach: it works at the **DNS layer**, which is the one layer that follows the phone across every connection. It deploys a lightweight iOS Mobile Config Profile (`.mobileconfig`) that hardcodes **Control D** as the system-wide Encrypted DNS resolver (DoH/DoT) across both Wi-Fi and cellular networks. Control D simplifies this deployment with a QR-code-based helper for instant profile application. By integrating Control D's cloud API with Home Assistant, you can dynamically update filtering rules — blocking specific services or enforcing a total internet cutoff — while implementing real-time detection for profile bypass or removal. Because it works at the DNS layer, the same protection follows the phone off the couch, off the Wi-Fi, and onto cellular. It's not a hard wall — a determined user can still bypass it — but the built-in bypass detection is there to *discourage* that and surface it when it happens.

---

## **Impactful Real-World Scenarios**

* **Scenario 1 — The "Dumb Phone" Data Cutoff:** Automatically reduce a kid’s iPhone to a basic voice-and-text device until their homework or daily chores are completed. Home Assistant flips a single Control D rule that drops all app data, web browsing, and background feeds over cellular and Wi-Fi, leaving native phone calls and SMS intact.
* **Scenario 2 — Automated Focus & Self-Regulation:** Eliminate distraction during deep work, exam prep, or sleep routines without relying on manual willpower or easily bypassed Screen Time passcodes. Home Assistant triggers Control D focus profiles automatically based on time, calendar events, or bedtime automations.
* **Scenario 3 — Real-Time Bypass Detection:** If a user attempts to delete the DNS profile settings to dodge filtering, Home Assistant catches it by comparing background telemetry from the HA Companion App against Control D's query logs. When a device is active online but failing to route DNS queries, Home Assistant fires an instant alert.

---

## **How the DNS Bypass Detection Engine Works**

Because iPhones are naturally chatty, background services and location check-ins hit the DNS services constantly, even while the phone is locked in a pocket or just charging overnight. That constant chatter is exactly what makes bypass detection possible.

```
┌────────────────┐      Background Telemetry / Location       ┌─────────────────┐
│   iPhone       ├───────────────────────────────────────────►│  Home Assistant │
│  (Cell / Wi-Fi)│                                            │                 │
└───────┬────────┘                                            └────────┬────────┘
        │                                                              │
        │ Encrypted DNS Queries                                        │ Control D API
        ▼                                                              ▼
┌────────────────┐          Queries Stopped?                  ┌─────────────────┐
│   Control D    ├───────────────────────────────────────────►│  Bypass Engine  │
│   DNS Engine   │     (Last Query vs. Location Delta)        │  & Alerts       │
└────────────────┘                                            └─────────────────┘
```

The engine's job is to prove two independent things about the phone before it will raise an alarm: that the phone **is actually online**, and that it **isn't routing through Control D**. Only when a device is demonstrably active but the protected resolver has stopped seeing it does Home Assistant call it a bypass — and it requires the signal to persist before alerting, so a transient blip or an idle phone doesn't cause a false alarm. (There's a separate, router-level layer for a phone that drops off the home network entirely, covered in a later article.) The exact checks and thresholds live in the deep dive below.

## **How Access Control From Home Assistant Is Implemented**

Enforcement is built on a simple two-layer model that resolves to a single effective mode per user:

- **Access Profile** — the normal scheduled posture (`Standard`, `Focus`, `Downtime`) that changes freely through the day.
- **Access Enforcement** — a temporary override or consequence (`Pause Rules`, `Chore Lock`, `Lockdown`) that takes precedence when active.

The **effective mode** is the single resolved value — *the enforcement wins if one is active, otherwise the profile applies.* Every mode change (or a Home Assistant restart, to prevent state drift) is pushed through one shared driver that applies the matching policy to every device through Control D — so the same behavior holds whether the phone is on home Wi-Fi or out on cellular. How that's wired, and the exact category map, is in the deep dive below.

---

## **Prerequisites: The Tools I Use**

This is the exact stack I run. You can customize and tailor each piece to your own needs once you understand what it contributes — but this is a proven, working combination.

* **Home Assistant** — the orchestration brain. All policy, state, and automation logic lives here.
* **Home Assistant Companion App** — installed on the managed iPhone. Provides presence, location, and activity telemetry that Home Assistant uses to prove the device is online.
* **Control D** — a paid account (a free trial is available). The cloud DNS engine that actually enforces the filtering rules.
* **Control D Manager** — a custom integration I wrote that exposes Control D's per-device controls to Home Assistant as native entities (switches and selects) that automations can drive.
* **iCloud3 v3** — a custom integration (very well done) that gives detailed iPhone integration, including battery level, last-located timestamps, and rich location telemetry that feeds the bypass detection engine.

**A Note on Naming (Do This Up Front)**

This is much easier if you take time up front to name your phone(s) consistently across every tool. You want to be able to recognize at a glance which entity comes from iCloud3, which comes from Control D, and which comes from the Companion App — all for the same physical device.

I'd go as far as recommending you actually change the name of the phone itself in **iPhone Settings → About → Name**. That device name becomes the default for things like iCloud and the Companion App, so naming it something meaningful up front (e.g. "Kadens Phone") means the entities those tools create will inherit a sensible name automatically. Also, I don't use an apostrophe in the name because that becomes an "_" in HA.  It's not strictly required — you can rename entities in Home Assistant afterward — but it's much easier to get it right at the source.

For reference, here is the convention I use. Each phone gets a stable prefix based on the owner, and each integration appends its own source tag:

| Tool | Example Entity | Pattern |
|------|----------------|---------|
| Person tracker | `person.kaden` | `person.<name>` |
| iCloud3 v3 (device tracker) | `device_tracker.kadens_phone_ic3` | `<name>_phone_ic3` |
| iCloud3 v3 (sensors) | `sensor.kadens_phone_ic3_battery` | `<name>_phone_ic3_*` |
| Control D Manager | `sensor.kadens_phone_status` | `<name>_phone_status` |

One thing worth calling out: the Control D Manager entity names don't just tag the *device* — they reflect the **dedicated Control D profile** we create for that phone (the "User's Phone" profile described below). Because each phone gets its own isolated profile, the entity name carries that same `<name>_phone` prefix. So when you see `kadens_phone_status`, it's shorthand for "the Control D profile that belongs to Kaden's phone" — which is exactly why we name the profile and the entities the way we do. It keeps the profile, the device, and the Home Assistant entities all aligned under one consistent name.

The `person.<name>` tracker is the anchor — it's what ties the physical person to their phone. Here's the important part: **each person should have exactly one device tracker assigned to them, and that should be the iCloud3 tracker** (`device_tracker.kadens_phone_ic3`). iCloud3 is already smart enough to combine the Companion App's presence data with the location data it pulls directly from the phone via iCloud, so you don't need a separate Companion App tracker on the person. Keeping a single tracker per person avoids conflicting presence signals and keeps the bypass engine and access driver cross-referencing one clean source of truth.

---

## **Control D Setup: A Dedicated Phone Profile**

The Control D side has one setup detail that's a little unique, and it's worth getting right from the start.

Instead of lumping the phone into a shared profile with all your other devices, I create a **separate profile that is dedicated to the phone device alone**. This is deliberate: it lets Home Assistant control that one profile — and therefore that one device — independently, without affecting any other device on my network. When the access driver flips a rule for the phone, it only touches the phone.

The flow is:

1. **Create a dedicated profile** in Control D (e.g. "User's Phone") that will hold only this device.
2. **Add the device to that profile** using the QR code in the Control D helper. Scanning the QR code installs the `.mobileconfig` profile on the iPhone, which hardcodes Control D as the system-wide encrypted DNS resolver across both Wi-Fi and cellular.

Once the device is bound to its own profile, every rule you toggle on that profile applies to the phone and nothing else — which is exactly the isolation Home Assistant needs to enforce per-user policy without collateral impact on the rest of the household.

**Services Set to Always Bypass**

Before you start blocking things, add a few key services to the profile that will **always be bypassed** — meaning they are never filtered, so their traffic always works. This matters because even if you decide to block all internet to the phone, there are still core services you want data flowing to. This is a basic recommended list, but you'll certainly want to consider your own situation.

**Control D-Defined Services (Always Bypass)**

Control D defines these as services you can bypass directly, so you don't have to create many individual rules for each vendor (e.g. all of Apple):

- **Apple**
- **Google**
- **DigiCert**
- **Microsoft**

**Base Allowed Bypass Rules**

1. `<your personal home network domain if you have one>`
2. `controld.com`
3. `icloud.com`
4. `in-addr.arpa`
5. `nabu.casa`
6. `nabucasa.com`
7. `resolver.arpa`
8. `t-mobile.com` (or your mobile phone provider)

> **Advanced note:** I say "unique" because this profile is dedicated to the phone — but it doesn't have to hold only the phone. If you run a VPN on the phone, you can also assign the phone's **VPN profile** to this same Control D profile. That way, traffic routed through the VPN follows the same DNS rules and reports into Control D properly, instead of bypassing the filtering. That's an advanced setup I'm not covering here, but if you use one, you'd include that VPN endpoint in this profile too.

---

## **Wiring Control D into Home Assistant**

Now that the profile is set up, the next step is connecting Control D to Home Assistant. The reality is this is straightforward — you just install the integration and connect it with an API key.

1. **Install the Control D Manager integration** in Home Assistant. It's a custom integration I wrote — follow the install instructions in the [controld-manager-ha README](https://github.com/ccpk1/controld-manager-ha/blob/main/README.md).
2. **Create an API key in Control D** (the README walks you through this). This key is what the integration uses to connect to your Control D account.
3. **Enable the "User's Phone" profile** in the integration's configuration options. This is the dedicated profile you created in the previous section.

The default setup gives you full internet blocking plus filters for things like social media. You're welcome to add other services or filters you want to control — but note that for a service to show up in the integration for selection, you need to **activate it in the Control D profile first**. Only services that are active in the profile will appear as controllable entities in Home Assistant.

---

## **The Build: Background Setup**

The remaining configuration — the helpers, templates, automations, and dashboard cards that turn all this into a working system — is substantial. Rather than pasting it into this post section by section, I've packaged it into two files you can drop straight into your configuration.

You **can** paste the contents directly into `configuration.yaml`. But the easier and recommended approach is to use the **packages** method, which keeps all of it together but logically separated from the rest of your config — so it's easy to find, version, and remove later. We're not going to dig into how packages work here; just know that if you use them, this all lives cleanly in one place instead of cluttering the main config file.

There are two pieces:

* **`personal_devices_single_user.yaml`** — the logic. Helpers (`input_select`, `input_number`, `input_boolean`), the effective-mode template, the DNS bypass sensor, the shared access driver script, the per-user automation, and the bypass alert.
* **`personal_devices_dashboard_cards.yaml`** — the dashboard cards that give you the at-a-glance view and the manual controls (access profile, enforcement, and internet toggles).

Both use a consistent naming convention around the owner's name. The files use **Kaden** as the example; replace it throughout with your own name using the find/replace checkpoint at the bottom of the config file.

> **Note on the dashboard cards:** they use a few custom cards (`button-card`, `stack-in-card`, `auto-entities`, `multiple-entity-row`, `fold-entity-row`) and `card-mod`, plus the `simple-icons` icon pack. Install those via HACS before the dashboard cards will render. The Control D auto-entities card also relies on the Control D Manager integration exposing `profile_name`, `item_type`, and `item_name` on each entity — so it works with that integration specifically.

---

## **The Dashboard**

The dashboard gives you the full picture at a glance *and* puts manual control at your fingertips. It's organized into a per-user view (one per managed person), with each view built from four cards. All four live in `personal_devices_dashboard_cards.yaml`.

**Status** — the at-a-glance health of the setup. It shows the person's **current location** (Home / Away / a specific zone) and the **DNS filtering status** — a green "Secured" shield when the phone is routing through Control D, red "Bypassed" when the `dns_bypassed` sensor is active. This is the quick sanity check: is the phone where it should be, and is its DNS protection actually on?

**Device Rules** — shows the **effective access mode** as a colored pill (Standard green, Focus blue, Downtime indigo, Pause Rules teal, Chore Lock orange, Lockdown red), with the two layers that produced it below: the current **Access Profile** and the active **Access Enforcement**. This makes the "enforcement wins if active, otherwise profile applies" rule concrete at a glance.

**Device Management** — the interactive control stack. This is where you take manual action:
* **Access Profile** — three pill buttons (Standard / Focus / Downtime) that set the scheduled posture.
* **Access Enforcement** — four override buttons (Follow Profile / Pause Rules / Chore Lock / Lockdown). `Follow Profile` is the green default (no override); the others apply a consequence that takes precedence.
* A **Configure** fold with the alert toggle and the bypass confirmation delay.

Every pill and button fires an `input_select.select_option` service call on the same `kaden_access_profile` or `kaden_access_enforcement` helper the automation reads — so tapping the dashboard is identical to an automation flipping that helper. That's what keeps manual and automatic control on the exact same state.

**Internet Controls (Control D)** — an auto-entities card that discovers your Control D access rules (internet, gaming, social, YouTube, etc.) from the Control D Manager integration and renders each as an on/off toggle. In the single-user template it renders the phone profile only; if you have a second Control D profile for other devices, you can set it to render both columns.

The four cards work together with the logic in `personal_devices_single_user.yaml` — the card entities are literally the helpers, template sensors, and bypass sensors that file creates, so nothing is duplicated between the two.

One thing worth knowing: **Device Management** and **Internet Controls** are **control surfaces** — anyone who can see them can change the access mode or toggle blocking. **Status** and **Device Rules** are read-only (they only display state). You may want to restrict visibility of the two control cards to just the household admins, so a kid can't change their own access or flip off the filtering. The `personal_devices_dashboard_cards.yaml` file notes where the `visibility` condition goes for this.

---

## **DNS Bypass Detection: How It Decides**

At the center of the detection engine is a single binary sensor — `binary_sensor.kadens_phone_dns_bypassed` — that answers one question: *"is this phone supposed to be protected but currently isn't?"* Everything upstream of that sensor is the logic that earns an accurate yes/no.

The engine cross-checks three things before it will fire, so a single glitch can't trigger a false alarm:

1. **The Control D data connection is healthy and current.** The integration has to confirm it recently synced with Control D — if it can't talk to the API, we can't trust its view of the phone, so the check holds.

2. **There's independent proof the phone is online.** The engine needs to confirm the phone actually has a live internet connection *before* it will accuse it of bypassing DNS. `Proof of internet` — a local router-based device tracker is the ideal signal here, but the GPS last-known timestamp from iCloud3 also works, so the check functions regardless of whether the phone is on home Wi-Fi or out on cellular.

3. **The DNS protection isn't actually active.** The engine confirms the phone isn't routing through Control D on its main path *and* isn't covered by the VPN fallback. Only if protection is genuinely off does the bypass condition get closer to true.

But even when all three hold, the sensor doesn't immediately fire. A bypass flips the indicator, and instead of alerting right away, Home Assistant uses that window to **actively re-pull the data sources** — triggering a fresh sync of the Control D and location trackers and letting the responses come back — so the decision is made on the most current snapshot, not a possibly-stale one. The alert only fires once that confirm-then-recheck runs and the bypass still holds. This is what rules out a transient blip, a missed sync, or a phone that simply went idle, without the data being wrong.

Once the condition survives that recheck window, a second sensor — `binary_sensor.kaden_network_bypass_alert_active` — flips on and gates the actual alert, so a repeated notification doesn't spam until someone fixes the underlying state.

The single-user template ships this refresh automation as part of the package — on a bypass it re-syncs the Control D and iCloud3 sources, so the confirmation window makes its call on fresh data. In the full system that same automation also syncs the Firewalla runtime, but that's router-level and covered in a later part.

For the exact timings, the sensor thresholds, and the data source your setup should pull from, dig into the `dns_bypassed` sensor in `personal_devices_single_user.yaml` — every threshold is tuned there and well commented.

---

## **Access Mode & Management: How It Decides**

Access control runs on the same principle as bypass detection: one central state, computed from a small set of inputs, driving a single consistent action. Everything hangs off two helpers and one template sensor.

**The two helpers are the inputs.**

* **`input_select.kaden_access_profile`** — the *scheduled* posture. It holds one of `Standard`, `Focus`, or `Downtime`, and changes freely through the day (a time-based automation, a bedtime routine, a calendar event — whatever you schedule).
* **`input_select.kaden_access_enforcement`** — the *override*. It's `Auto` (the default, meaning no override) or one of `Pause Rules`, `Chore Lock`, `Lockdown`. An enforcement always takes precedence over the profile when it's set.

**The template sensor resolves them.**

`sensor.kaden_access_mode` computes the single **effective mode** with one rule: *if the enforcement is anything other than `Auto`, the enforcement wins; otherwise the profile applies.* That's the entire decision — enforcement > profile. The dashboard's **Device Rules** card is literally showing this sensor, which is why the pill and the actual behavior can never disagree.

**The driver applies it.**

When the effective mode changes — or when Home Assistant starts, to prevent state drift — the **Access Enforcer** automation fires and calls the shared `apply_devices` script. That script maps the effective mode to a set of **categories** (`internet`, `entertainment`, `social`, `gaming`, `video`, ...) via a mode→category map, then walks the Control D inventory and drives every item to the state that map dictates: full internet blocking for `Lockdown`/`Chore Lock`, entertainment-only cutoffs for `Focus`, everything open for `Standard`.

Because the enforcement helper is the *same* input the dashboard buttons write to, a manual tap and an automation landing on the same mode produce the exact same end state — there's no second copy of the policy to keep in sync. That single shared driver is why the system doesn't drift: every mode, automatic or manual, resolves through one path.

For the exact mode→category map and the inventory items, see the `apply_devices` script and the Access Enforcer automation in `personal_devices_single_user.yaml`.



