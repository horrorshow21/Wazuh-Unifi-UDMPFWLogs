# Wazuh Decoders and Rules for UniFi UDM Pro

Custom Wazuh decoders and rules for ingesting firewall/policy engine logs from a Ubiquiti UniFi Dream Machine Pro (UDM Pro).

Tested on:
- UniFi OS 4.x
- UniFi Network 9.x / 10.x
- Wazuh 4.14.x

---

## What These Parse

The UDM Pro's policy engine logs iptables-style syslog messages for each firewall rule hit. These are **not** CEF format — they look like this:

```
UDMPRO [WAN_LOCAL-D-2147483647] DESCR="[WAN_LOCAL]Block All Traffic" IN=eth8 OUT= MAC=74:ac:b9:... SRC=x.x.x.x DST=x.x.x.x LEN=53 PROTO=UDP SPT=22146 DPT=27016
```

Fields extracted:
| Field | Description |
|---|---|
| `action` | `A` (allow) or `D` (deny/block) |
| `extra_data` | Rule chain (e.g. `WAN_LOCAL`, `CUSTOM1_CUSTOM6`) |
| `rule_name` | Rule description from UniFi UI |
| `srcip` | Source IP |
| `dstip` | Destination IP |
| `protocol` | TCP, UDP, ICMP, etc. |
| `srcport` | Source port |
| `dstport` | Destination port |

---

## Prerequisites

### 1. Wazuh syslog listener

Add a syslog remote block to `/var/ossec/etc/ossec.conf` inside `<ossec_config>`:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>YOUR_UDM_IP_OR_SUBNET</allowed-ips>
  <local_ip>YOUR_WAZUH_IP</local_ip>
</remote>
```

Replace `YOUR_UDM_IP_OR_SUBNET` with your UDM Pro's IP or subnet (e.g. `192.168.1.0/24`). If your Wazuh server is on a different subnet than your UDM Pro's primary IP, add both:

```xml
<allowed-ips>192.168.1.0/24</allowed-ips>
<allowed-ips>192.168.5.0/24</allowed-ips>
```

### 2. Enable syslog logging on UniFi firewall rules

In UniFi Network UI:

**Settings → CyberSecure → Policy Engine → Policy Table**

For each rule you want to log, click **Edit** and enable **Syslog Logging**.

### 3. Enable the SIEM syslog destination

In UniFi Network UI:

**Settings → CyberSecure → Traffic Logging → Activity Logging (Syslog)**

Select **SIEM Server**, set **Server Address** to your Wazuh IP and **Port** to `514`.

Also configure:

**Settings → Control Plane → Integrations → Activity Logging (Syslog)**

Select **SIEM Server** with the same address and port for administrative/CEF events.

---

## Installation

```bash
# Copy files
sudo cp udmpro_decoders.xml /var/ossec/etc/decoders/
sudo cp udmpro_rules.xml /var/ossec/etc/rules/

# Set permissions
sudo chown wazuh:wazuh /var/ossec/etc/decoders/udmpro_decoders.xml
sudo chown wazuh:wazuh /var/ossec/etc/rules/udmpro_rules.xml
sudo chmod 660 /var/ossec/etc/decoders/udmpro_decoders.xml
sudo chmod 660 /var/ossec/etc/rules/udmpro_rules.xml

# Test configuration
sudo /var/ossec/bin/wazuh-analysisd -t

# Restart if clean
sudo systemctl restart wazuh-manager
```

---

## Verify

First, grab a real log line from your system:

```bash
sudo grep "UDMPRO" /var/ossec/logs/archives/archives.log | head -5
```

Copy one of the log lines starting from the timestamp (e.g. `Apr 15 06:32:53 UDMPRO [WAN_LOCAL-D-...`), then run:

```bash
sudo /var/ossec/bin/wazuh-logtest
```

Paste the log line at the prompt.

Expected output:
```
**Phase 2: Completed decoding.
        name: 'udmpro-firewall'
        action: 'D'
        extra_data: 'WAN_LOCAL'
        rule_name: '[WAN_LOCAL]Block All Traffic'

**Phase 3: Completed filtering (rules).
        id: '110002'
        level: '5'
        description: 'UDM Pro: Traffic blocked by firewall rule: ...'
```

Check live alerts:

```bash
sudo tail -f /var/ossec/logs/alerts/alerts.log | grep -i udmpro
```

Dashboard filter: `rule.groups: udmpro`

---

## Rules

| Rule ID | Level | Description |
|---|---|---|
| 110000 | 3 | Any UDM Pro firewall event (base rule) |
| 110001 | 3 | Traffic allowed by firewall rule |
| 110002 | 5 | Traffic blocked by firewall rule |
| 110003 | 6 | Inbound WAN traffic blocked |
| 110004 | 5 | Inter-VLAN traffic blocked |
| 110005 | 7 | Cleartext DNS blocked (possible bypass attempt) |
| 110006 | 10 | High frequency WAN blocks from same source (scan/attack) |
| 110007 | 8 | Repeated inter-VLAN blocks from same source |

Rule IDs use the range `110000-110099`. If this conflicts with another ruleset, renumber them in `udmpro_rules.xml`.

---

## Notes

- These rules parse **iptables-style** policy engine logs only. CEF-format logs from the Control Plane integration require separate decoders (see [Halino/Wazuh-Unifi](https://github.com/Halino/Wazuh-Unifi)).
- The UDM Pro sends syslog from whichever interface faces your Wazuh server — make sure that IP is covered by `allowed-ips`.
- High-volume allow rules (DNS, logging) will generate a lot of level-3 alerts. Consider raising rule 110001 to level 0 or adding `<if_sid>` exceptions to suppress specific rule names once you've confirmed everything is working.
