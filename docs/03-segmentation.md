# 03 – Network segmentation with NSGs

## Mental model

```
Subnet  = structure
Routing = is there a path?
NSG     = is the traffic allowed to use it?
```

Azure routes traffic between subnets in the same VNet by default. Isolation therefore needs filtering, here with NSGs on subnet level.

NSG rules are evaluated by priority. A lower number means higher priority, and the first matching rule wins. NSGs are stateful, so return traffic for an allowed connection doesn't need its own rule.

```
Specific allow (e.g. 100–300)
        ↓
Broad deny (400+)
        ↓
Azure default rules (65000+)
```

---

## nsg-endpoints: no RDP from clients to Management

**Rule:** `Deny-RDP-Endpoints-to-Management` (outbound)

```
Endpoints 10.0.0.64/26 → Management 10.0.0.128/27 → TCP/3389 → Deny
```

**Why:** A client should not normally start admin RDP sessions against management systems. A compromised endpoint gets one less path for lateral movement towards the DC.

**Verification:**

| Test | Before | After |
| --- | --- | --- |
| `Test-NetConnection 10.0.0.132 -Port 3389` | True | False |
| Network Watcher IP flow verify, TCP/3389 | – | Access denied by `Deny-RDP-Endpoints-to-Management` |
| IP flow verify, TCP/445 (control test) | – | Access allowed |

The control test shows that the rule is specific to RDP and doesn't block all traffic between the subnets.

---

## nsg-attacker: default deny for the attacker zone

### Baseline before the NSG

| From Kali to | ICMP | TCP/3389 |
| --- | --- | --- |
| DC 10.0.0.132 | Worked | Worked |
| Win11 10.0.0.68 | No reply | Worked |

The missing ping reply from Win11 is most likely the Windows firewall dropping ICMP. TCP/3389 still worked, so the client was **not** isolated. Ping alone is not a reliable isolation test.

### Target policy

| Source | Destination | Traffic | Baseline |
| --- | --- | --- | --- |
| Azure Bastion | Attacker | TCP/22 | Allow |
| Management | Attacker | Any | Deny |
| Attacker | Management | Any | Deny |
| Attacker | Endpoints | Any | Deny |
| Attacker | Internet | Any | Deny |
| Attacker | Endpoints | Scenario-specific | Opened only when a scenario needs it |

Internet access for Kali is opened only temporarily, for example to update or install tools.

### Rules

**Inbound**

| Priority | Name | Source | Destination | Port | Action |
| --- | --- | --- | --- | --- | --- |
| 100 | `Allow-Bastion-SSH` | 168.63.129.16 | 10.0.0.196 | TCP/22 | Allow |
| 400 | `Deny-VNet-Attacker` | VirtualNetwork | 10.0.0.192/28 | Any | Deny |

**Outbound**

| Priority | Name | Source | Destination | Port | Action |
| --- | --- | --- | --- | --- | --- |
| 400 | `Deny-Attacker-Management` | 10.0.0.192/28 | 10.0.0.128/27 | Any | Deny |
| 410 | `Deny-Attacker-Endpoints` | 10.0.0.192/28 | 10.0.0.64/26 | Any | Deny |
| 420 | `Deny-Attacker-Internet` | 10.0.0.192/28 | Internet | Any | Deny |

The rules only took effect once the NSG was **associated** with the Attacker subnet.

**Bastion source address:** The first version of the allow rule used `10.0.0.0/26` (AzureBastionSubnet) as source, and Bastion stopped working. The error message and the login record on Kali (`Last login ... from 168.63.129.16`) showed that the Bastion Developer connection came from `168.63.129.16`. The Developer SKU runs on shared Microsoft infrastructure and doesn't use the AzureBastionSubnet the way the dedicated SKUs do.

### Verification after the NSG

| Flow | Test | Before | After |
| --- | --- | --- | --- |
| Bastion → Kali TCP/22 | SSH session, `whoami` | Worked | Works |
| Kali → DC ICMP | `ping -c 2 10.0.0.132` | Worked | 100 % packet loss |
| Kali → DC TCP/3389 | `nc -zv -w 3 10.0.0.132 3389` | Succeeded | Timed out |
| Kali → Win11 ICMP | `ping -c 2 10.0.0.68` | No reply | No reply (not a valid before/after test) |
| Kali → Win11 TCP/3389 | `nc -zv -w 3 10.0.0.68 3389` | Succeeded | Timed out |
| Kali → Internet TCP/443 | `nc -zv -w 3 1.1.1.1 443` | Not tested | Timed out |
| DC → Kali TCP/22 | `Test-NetConnection 10.0.0.196 -Port 22` | – | False |

The internet test used an IP address instead of a name so that DNS wouldn't affect the result.

### Opening a scenario

When a purple team scenario needs a path, a specific allow rule is added with a priority below 400, for example:

- SMB scenario: TCP/445
- RDP scenario: TCP/3389
- WinRM: TCP/5985/5986

When the scenario is done the rule is removed and the zone goes back to baseline. That way it's always clear which rule makes an attack path possible.

---

## Limitation

These tests prove that the **tested flows** are blocked or allowed. They don't prove that every port and protocol in every direction is blocked.

## Security perspective

- **Attack:** Without the NSG, a compromised or misused machine in the Attacker zone could reach the DC and clients directly, and send data out to the internet.
- **Defence:** Default deny limits lateral movement and outbound communication. Scenario-specific allow rules make every opened path deliberate and documented.
- **What to log:** NSG rule changes are control plane operations and end up in the Azure Activity Log, which the lab already sends to Sentinel (see [04-logging-pipeline.md](04-logging-pipeline.md)).
