# 06 – Troubleshooting

Working method:

```
Symptom → Hypothesis → Test → Observation → Conclusion → Next hypothesis
```

I try to change one thing at a time, and to keep observation, hypothesis and verified conclusion apart.

---

## Custom detection ran but never created an alert

**Symptom:** The custom detection for PowerShell on the DC ran with status *Completed*, but no alert appeared, even though the query found matches in Advanced hunting.

**What I ruled out:**

| Test | Result | Conclusion |
| --- | --- | --- |
| Alert queue filtered on all statuses and sources | 0 alerts | Not a filter problem |
| `AlertInfo` in Advanced hunting, 30 days back | 0 rows | The alert was never created, it wasn't hidden |
| Rule query run manually against `law-lab-purpleteam` | 16 matches with correct HostName/DnsDomain | The detection logic works |
| Sentinel workspace settings in the Defender portal | A second workspace (used for a course) was **primary**, the lab workspace was secondary | Possible cause |

**Hypothesis:** According to Microsoft Learn at the time, creating custom detections was limited to the primary workspace, and support for other onboarded workspaces was listed as planned. The rule probably ran against the primary workspace, which had no logs from `vm-dc01`.

**Action:** Set `law-lab-purpleteam` as primary. Created a new rule (`Lab v2`) with the same query and settings, turned the original off, and created a new test event.

**Verification:**

- The v2 rule created an alert with `vm-dc01` as entity.
- Control observation: the unchanged original rule created no alert at one run, but did create one at the next run, after the primary workspace was switched, with the same events in the lookback window.
- A later scheduled run added the new test event to the alert.

**Conclusion:** Strong support that the wrong primary workspace was the cause. I don't call it proven, because the exact time of the switch is not visible in the Azure Activity Log. Changes in the Defender portal are not logged there, so a manual change log is needed.

The same trap had come up earlier: a query returned nothing in Advanced hunting because the wrong workspace was selected. When a query works in one tool but not in another, check **workspace, data source and scope** before changing the query or the log configuration.

---

## Entity mapping pointed to missing columns

During the same troubleshooting I found that the rule query no longer returned `HostName` and `DnsDomain`, but entity mapping still expected them. I fixed the query to create both columns from `Computer` and verified the result in Advanced hunting.

It is not proven that this mismatch alone stopped the alert, but it had to be fixed for the Device entity to work.

---

## Event 4104 stopped appearing

**Symptom:** 4104 events existed historically, but new PowerShell tests created no new events.

**Hypothesis:** Script Block Logging was not active.

**Test:** Checked the registry and effective Group Policy with `gpresult`. No active Script Block Logging policy was found.

**Action:** Created a separate GPO for Domain Controllers with Script Block Logging enabled.

**Verification:** The registry showed `EnableScriptBlockLogging = 1`, and a new test created event 4104.

**Root cause:** The events were never created locally, so there was nothing to collect.

---

## Duplicate alerts from one test

Overlapping query windows (5-minute frequency, 10-minute lookback). See [detection tuning](05-detections.md#detection-tuning).

---

## Recurring Bastion "Connection Error"

**Symptom:** Bastion sessions to both `vm-win11-01` and `vm-dc01` kept dropping with *Connection Error*.

**What I ruled out on the DC:** `Get-Service DNS,NTDS,TermService` were all *Running*, CPU usage was low and memory was available. I had also increased `vm-win11-01` from 4 to 8 GiB RAM earlier, but that didn't solve the problem.

**Cause:** The lab uses **Bastion Developer**, which supports one connection at a time. A session to one VM could break when a session to another VM was opened.

**Lesson:** When several components show the same symptom, check their shared dependencies early. A technical limit can look like instability.

To get better visibility, Bastion audit logs are now sent to `law-lab-purpleteam` (diagnostic setting `diag-bastion-purpleteam`).

**Decision:** Keep the Developer SKU, since it is enough for the lab and costs nothing extra. One Bastion session at a time.

---

## Bastion could not reach the DC, but everything else worked

**Symptom:** After a restart, Bastion could no longer open an RDP session to `vm-dc01`. Bastion to Kali (SSH) still worked.

**Checked on the DC:** VM running, `TermService` running, TCP/3389 listening, Windows Firewall rules for RDP active for all profiles with no block rules, no NSG on the Management subnet or the DC's NIC, effective routes OK.

**Checked in the Activity Log:** All recent NSG changes concerned the Attacker subnet, and RDP to the DC had worked after the last of them. The NSG work was ruled out.

**Positive control:** A test from `vm-win11-01` failed at first, but IP flow verify showed that it was blocked by my own rule `Deny-RDP-Endpoints-to-Management`. That test said nothing about the DC. I added a temporary allow rule from 10.0.0.68 to 10.0.0.132 on TCP/3389. After that:

- `Test-NetConnection` → `TcpTestSucceeded : True`
- A full RDP session with `mstsc` from Win11 to the DC worked with the domain account
- The DC logged event 261 (RDP connection received) and 4624 with logon type 10 from source IP 10.0.0.68

This proved that the whole RDP stack on the DC worked (TCP, RDP, TLS/NLA, domain authentication), so the problem was limited to Bastion Developer → `vm-dc01`.

**Cause:** An old Bastion session was hanging. After it was released, login to the DC worked again.

**Cleanup:** The temporary rule `TEMP-Allow-Win11-DC-RDP` was removed, and IP flow verify again showed *Access denied* for Endpoints → Management on TCP/3389.

**Lessons:**

- Don't draw conclusions from a test until you know that the test traffic is allowed to reach the target.
- Use positive controls to tell a fault in the target apart from a fault in the path to it.
- Use the Activity Log to correlate configuration changes with the last known working time.

---

## AllocationFailed when starting a VM

**Symptom:** `vm-win11-01` could not start: `ProvisioningState/failed/AllocationFailed`. Azure reported that there was not enough capacity for the VM size in the region.

The error code points at Azure compute capacity, not at Windows, NSG, Bastion or the logging configuration, so troubleshooting started there.

| Step | Result |
| --- | --- |
| Reapply | Still AllocationFailed |
| Deallocate → Start | Still AllocationFailed |
| Resize from `Standard_B2as_v2` to `Standard_B2s_v2` (both 2 vCPU / 8 GiB) | VM started, Bastion login worked |

**Root cause:** Azure capacity for the previous VM size in Sweden Central.
