# SOC Home Lab — Troubleshooting Log

## Overview
**Environment:** VirtualBox host running two VMs — an Ubuntu Server VM (Wazuh Manager, Indexer, Dashboard) and a Windows 11 VM (Wazuh Agent).
**Goal:** Build a functioning SIEM lab to practice log monitoring, agent management, and vulnerability detection triage.

---

## Case 1 — Agent Not Visible in Dashboard (Network Isolation)

**Symptom**
After confirming the Wazuh Agent service was running on the Windows VM (`services.msc` showed "Running"), the agent still did not appear in the Wazuh Dashboard's Agents summary.

**Investigation**
1. Verified the Wazuh Manager was active on the Ubuntu VM via `systemctl status wazuh-manager` — confirmed `active (running)`.
2. Tested network reachability from the Windows VM to the Ubuntu VM using `ping <manager-IP>`.

**Root Cause**
The VirtualBox network adapter for both VMs was set to **Internal Network** mode, which fully isolates VM traffic from the host and from each other unless explicitly bridged. This prevented the Windows agent from reaching the manager at all.

**Resolution**
Switched the network adapter mode to **Host-only Adapter**, allowing host-to-VM and VM-to-VM communication while keeping the lab isolated from the external network. Re-ran the ping test to confirm connectivity before proceeding.

**Lesson Learned**
Network topology should be validated with a basic connectivity test (`ping`) *before* troubleshooting application-layer issues — saves time by ruling out infrastructure problems first.

---

## Case 2 — "Invalid Agent Name" Enrollment Error

**Symptom**
Agent installation completed and the Windows service was running, but the manager log rejected enrollment with:
```
ERROR: Invalid agent name: nitro5. Unable to add agent (from manager)
```

**Investigation**
1. Checked the Wazuh Dashboard's Agents list for any existing entry using the same hostname.
2. Found a stale/incomplete agent registration left over from an earlier failed installation attempt.

**Root Cause**
A previous installation attempt had partially registered an agent under the same name in the manager's agent database, causing a naming conflict on re-enrollment.

**Resolution**
1. Removed the stale agent entry from the manager (`manage_agents` / Dashboard delete).
2. Cleared the leftover `client.keys` file on the Windows host to remove the old registration reference.
3. Re-ran the agent deployment command from the Dashboard's "Deploy new agent" flow.

**Lesson Learned**
Failed installs can leave partial state behind (both on the agent and manager side). When re-registering, check for and clean up stale entries on *both* ends, not just the one that errored.

---

## Case 3 — 109 Critical Vulnerabilities on a Fresh Windows VM

**Symptom**
Wazuh's Vulnerability Detection module reported 109 Critical-severity findings on the Windows 11 agent shortly after enrollment.

**Investigation**
Drilled into the Vulnerability Detection dashboard's "Top 5 packages" breakdown, which attributed 105 of the 109 findings to a single package: Mozilla Firefox.

**Root Cause**
The VM image had an outdated Firefox installation with several years' worth of unpatched CVEs (browser software accumulates CVEs quickly due to frequent security releases).

**Resolution**
Since Firefox was not actively used on this VM, it was uninstalled entirely rather than patched — reducing the attack surface instead of maintaining a browser with no operational need.

**Lesson Learned**
Applied the security principle of **attack surface reduction**: unused software should be removed rather than maintained, since every installed package is a potential vulnerability source regardless of whether it's ever opened.

---

## Tools Used
- VirtualBox (VM hypervisor, network configuration)
- Wazuh (SIEM: manager, indexer, dashboard, agent)
- Windows 11 (agent endpoint)
- Ubuntu Server (manager host)

## Skills Demonstrated
- VM network configuration and troubleshooting (host-only vs internal network modes)
- Log-based root cause analysis (`ossec.log`, `systemctl status`)
- Agent lifecycle management (enrollment, deregistration, re-enrollment)
- Vulnerability triage and attack surface reduction reasoning
