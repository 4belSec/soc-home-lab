# SOC Home Lab

A self-built home lab simulating a corporate SOC environment — built to practice detection engineering, log analysis, and incident investigation as part of my transition into a SOC Analyst role.

## Architecture

![Proxmox Dashboard](screenshots/01-proxmox-dashboard.png)

| Component | Role | Stack |
|---|---|---|
| Hypervisor | Virtualization host | Proxmox VE 9.2.3 |
| Endpoint | Simulated corporate workstation | Windows 10 Pro + Sysmon |
| SIEM | Log ingestion & analysis | Splunk Enterprise 10.4.0 |
| Attacker | Threat simulation | Kali Linux |
| Remote Access | Secure access without port forwarding | Tailscale |

**Data flow:** Windows generates activity → Sysmon captures it → Splunk Universal Forwarder ships it → Splunk indexes and displays it in real time.

---

## 1. Hypervisor Setup (Proxmox)

Started by moving off Proxmox's paid enterprise repositories to the free no-subscription repos, and fixing a Ceph repo authentication error along the way.

![Repo Configuration Fix](screenshots/02-repo-config-fix.png)

Key steps:
- Disabled `pve-enterprise.sources` and `ceph.sources`
- Added the `pve-no-subscription` repo
- Installed [Tailscale](https://tailscale.com) on the host for remote access without router port forwarding — critical since my ISP (AT&T) doesn't support easy port forwarding
- Enabled IP forwarding and subnet routing so I can SSH/RDP directly into any VM remotely

---

## 2. Endpoint: Windows 10 + Sysmon

Installed Windows 10 Pro and configured Sysmon using the [SwiftOnSecurity config](https://github.com/SwiftOnSecurity/sysmon-config) — the industry-standard baseline for high-quality event tracing.

![Windows Install](screenshots/03-win10-install.png)
![Sysmon Running](screenshots/04-sysmon-install.png)

```powershell
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```

This generates real security telemetry (process creation, network connections, etc.) under:
`Applications and Services Logs → Microsoft → Windows → Sysmon → Operational`

---

## 3. SIEM: Splunk Enterprise on Ubuntu Server

![Splunk Download](screenshots/05-splunk-download.png)

Installed Splunk Enterprise on Ubuntu Server 24.04. One notable fix: Splunk initially ran as root, which it actively warns against. Reconfigured it to run as a non-privileged user:

```bash
chown -R analyst:analyst /opt/splunk
/opt/splunk/bin/splunk start --accept-license
/opt/splunk/bin/splunk enable boot-start -user analyst
```

Also had to expand the VM's LVM partition (started with only 19GB usable, Splunk requires 5GB minimum free space to run queries):

```bash
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

---

## 4. Forwarding Sysmon Logs to Splunk

Installed the Splunk Universal Forwarder on the Windows endpoint and configured it to ship Sysmon logs to the indexer.

**outputs.conf:**
```ini
[tcpout]
defaultGroup = default-autolb-group

[tcpout:default-autolb-group]
server = 192.168.X.X:9997
```

**inputs.conf:**
```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = false
index = main
sourcetype = WinEventLog:Sysmon
```

**The real troubleshoot:** the Forwarder connected fine but Sysmon logs weren't arriving. The splunkd log revealed the issue:
