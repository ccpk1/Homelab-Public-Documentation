# Personal Device Management with Home Assistant

A multi-part series on building context-aware, layered device management for iOS devices using Home Assistant, Control D, and Firewalla.

## Why This Exists

Commercial parental control and device management tools are closed-source, subscription-heavy, and resist any kind of programmatic control. Home Assistant is the opposite — open, extensible, and designed for orchestration. This series shows how to bridge the two: using Home Assistant as the brain to drive DNS-level filtering, router-level enforcement, and task-based consequences across all of a device's connections (Wi-Fi and cellular).

## The Three Layers

This system is built in three layers, each covered by a dedicated post:

### [Part 1: Remote Profile Management & DNS Bypass Detection](part1-control-d-ha/)
**Control D + Home Assistant** — The DNS layer that follows the phone everywhere.

- Deploying Control D as the system-wide encrypted DNS resolver via iOS `.mobileconfig` profiles
- Dynamically toggling filtering rules, focus modes, and internet kill-switches via Home Assistant
- Real-time bypass detection using the HA Companion App, iCloud3, and Control D query telemetry
- Drop-in YAML helpers, templates, automations, and dashboard cards

### [Part 2: Local Network Orchestration](part2-firewalla-ha/) *(coming soon)*
**Firewalla + Home Assistant** — Router-level enforcement for devices that can't run DNS profiles.

- Hardware-level L3/L4 blocking for home devices (consoles, smart TVs, IoT, guest Wi-Fi)
- Automating Firewalla target lists and rule switches via Home Assistant
- Combining on-net (Firewalla) and off-net (Control D) controls into a unified driver

### [Part 3: Household Automation & Context-Aware Enforcement](part3-choreops/) *(coming soon)*
**ChoreOps + Home Assistant** — Linking task management to automated network consequences.

- Setting up ChoreOps for essential household tasks vs. allowance chores
- The Grace Engine: preventing lockouts during events and handling arrival grace periods
- Driving the Control D and Firewalla layers from automated task state

## Advanced: The Combined Production Model

The articles above each present their layer independently for clarity. In practice, all three layers are wired together into a single unified system where:

- The **Access Enforcer** automation drives both Control D (DNS) and Firewalla (router) from one shared effective-mode state
- The **Grace Engine** overrides both layers based on presence and context (e.g., pausing enforcement when a kid is at a school event)
- A single dashboard provides visibility across all layers, and a single mode change cascades everywhere

The `examples/` directory in Part 1 contains the single-user, single-layer version. The combined production model is documented across the full series — each part builds on the previous, and the final integration is the sum of all three layers working together.

## Prerequisites

- [Home Assistant](https://www.home-assistant.io/) with the Companion App installed on managed devices
- [Control D](https://controld.com/) account (paid, free trial available)
- [Control D Manager](https://github.com/ccpk1/controld-manager-ha) custom integration
- [iCloud3 v3](https://github.com/gcobb321/icloud3) custom integration
- For Part 2+: [Firewalla](https://firewalla.com/) router with API access
- For Part 3+: [ChoreOps](https://github.com/ccc13/home-assistant-choreops) or equivalent task management

## Naming Convention

The examples use **Kaden** as the placeholder name. Each article includes a find/replace checkpoint at the bottom of every file — replace `kaden`/`Kadens`/`kadens` with your own user's name to match your setup.

| Tool | Example Entity | Pattern |
|------|----------------|---------|
| Person tracker | `person.kaden` | `person.<name>` |
| iCloud3 v3 | `device_tracker.kadens_phone_ic3` | `<name>_phone_ic3` |
| Control D Manager | `sensor.kadens_phone_status` | `<name>_phone_status` |
