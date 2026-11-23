# 07 – Scenario design (planned)

> This page describes how I intend to choose and build purple team scenarios. No attack scenarios have been run yet.

## Scenario first, ATT&CK second

The goal is not to cover as many MITRE ATT&CK techniques as possible. Scenarios should come from realistic risks and attack paths in an Azure environment:

```
Relevant risk in the environment
        ↓
Plausible attacker behaviour
        ↓
MITRE ATT&CK tactic + technique
        ↓
Which telemetry should be created?
        ↓
Which log sources are needed?
        ↓
How can Sentinel / Defender / Entra detect it?
        ↓
What needs to improve if the activity isn't visible?
```

The purpose of each scenario is to check whether the lab's telemetry and detection capability are actually good enough to see the behaviour. Running the technique is not the point by itself.

## Tools and inspiration

- **Atomic Red Team** can generate controlled, ATT&CK-mapped activity. It's a way to produce test activity once a scenario and technique have been chosen, not the starting point for the design.
- **Real threat actors**, for example Scattered Spider, can be used as inspiration. The aim is not to recreate a whole attack chain, but to find documented behaviours that are relevant for the environment.

## Spread scenarios across telemetry layers

| Scenario area | Main telemetry |
| --- | --- |
| PowerShell execution | Endpoint: Windows events, PowerShell 4104 |
| Identity / RBAC abuse | Entra ID logs, Azure Activity Log |
| Lateral movement | Authentication and endpoint telemetry |
| Data access / exfiltration | Azure resource logs, Defender, Sentinel |

Examples of realistic Azure-focused scenarios: password spraying, suspicious PowerShell use, abuse of a managed identity, lateral movement through an over-privileged service principal, and data exfiltration from Azure Storage.

## How a scenario runs in the lab

```
Default deny → explicit allow for the scenario → run → observe telemetry → detect → remove the allow rule
```

See [03-segmentation.md](03-segmentation.md) for how the Attacker zone is opened per scenario.
