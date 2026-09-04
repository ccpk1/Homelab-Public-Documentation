# Part 2: Local Network Orchestration (Firewalla + Home Assistant)

> Coming soon — this post covers router-level L3/L4 enforcement using Firewalla and Home Assistant, combining on-net controls with the DNS layer from [Part 1](../part1-control-d-ha/) into a unified driver.

## What This Will Cover

- Automating Firewalla target lists and rule switches via Home Assistant
- Hardware-level blocking for devices that can't run DNS profiles (consoles, smart TVs, IoT)
- Combining Firewalla on-net controls with Control D off-net profiles in a single driver script
- Handling network state edge cases, HA reboots, and preventing state drift

## Prerequisites

Everything from [Part 1](../part1-control-d-ha/), plus:

- [Firewalla](https://firewalla.com/) router with API access
- Firewalla integration or API key configured in Home Assistant
