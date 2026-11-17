# AdGuard Home on OpenWrt behind a BSNL ONT (no bridge mode)

Network-wide ad and tracker blocking with a private AdGuard Home instance, for a setup where the ISP's ONT/router **cannot be put in bridge mode** (the OLT binds the connection to the ONT's MAC address) and OpenWrt acts only as a switch/access point.

## Setup

| Device | Role | Address |
|---|---|---|
| Ubiqom ONT (BSNL, PPPoE) | Modem + router + DHCP server | `192.168.1.200` |
| JioRouter AX6000 (OpenWrt 25.12) | Switch / AP + AdGuard Home | `192.168.1.2` |

- DHCP stays on the ONT (pool `192.168.1.33` to `192.168.1.254`). OpenWrt's IP is outside the pool, so it is never handed to another device.
- OpenWrt's own DHCP is **not** used. The ONT tells every client to use `192.168.1.2` as its DNS.

```
Client --> AdGuard Home (192.168.1.2) --> DoH upstream (Quad9) --> Internet
                                              ^ via ONT (192.168.1.200) / PPPoE
```

## 1. Install AdGuard Home (all in LuCI)

1. **System → Software** → *Update lists* → install `adguardhome`.
2. **System → Startup** → `dnsmasq`: **Stop**, then switch to **Disabled**. dnsmasq listens on port 53 even with DHCP off, so it conflicts with AdGuard Home.
3. **Network → Interfaces → LAN → Edit**:
   - IPv4 gateway: `192.168.1.200`
   - Advanced Settings → custom DNS: `94.140.14.14` (so the router itself can resolve names for updates and DoH bootstrap)
4. **System → Startup** → `adguardhome`: **Enabled**, then **Start**.
5. Open `http://192.168.1.2:3000` and run the wizard:
   - Admin interface: all interfaces, port `3000` (not 80, which LuCI uses)
   - DNS server: all interfaces, port `53`
   - Create the admin login.
6. **Settings → DNS settings**:
   - Upstream: `https://dns10.quad9.net/dns-query` (or any DoH provider)
   - Bootstrap: `94.140.14.14`, `1.1.1.1`
7. **Filters → DNS blocklists**: enable *AdGuard DNS filter* and optionally *AdAway Default Blocklist*.

## 2. Point the network at it (ONT)

In the ONT's **Network → LAN IP Address → IPv4 Configuration**:

- DHCP server: enabled
- Lan DNS Mode: **Static**
- Primary DNS: `192.168.1.2`
- Secondary DNS: `192.168.1.2` (same, so clients cannot fall back to another resolver)

Renew leases on clients (`ipconfig /release` then `ipconfig /renew`, or reconnect to Wi-Fi) and check `ipconfig /all`.

## 3. IPv6 (the part that leaks)

By default the ONT's **LAN IPv6 DNS Server** is set to *WANConnection*, so it advertises **BSNL's own IPv6 DNS servers** (`2001:4490:...`). Windows prefers IPv6 DNS, so queries bypass AdGuard Home even though IPv4 is configured correctly.

Options, from most to least involved:

1. **Advertise AdGuard Home over IPv6.** On the ONT's **IPv6 Configuration** page, set *LAN DNS* to **Static** and enter the link-local address of OpenWrt's `br-lan` (listed in AdGuard Home's Setup Guide as `fe80::...%br-lan`; **omit the `%br-lan` suffix**).
   - Do **not** use the ULA address (`fdb2:...`): clients have no route to it.
   - Do **not** use `eth0`'s link-local address; use the bridge's.
   - Link-local addresses do not change when BSNL changes your delegated prefix.
   - Also check the ONT's **RA Configuration** and **IPv6 DHCP Server Configuration** pages, as DNS can be advertised through those too.
2. **Stop advertising IPv6 DNS/RA on the ONT** so devices use IPv4 only.
3. **Disable IPv6 on the client** (per device).

**Verify:** `ipconfig /all` should list only `192.168.1.2` and your chosen AdGuard IPv6 address under DNS Servers, with no `2001:4490:...`.

## 4. Browsers bypass system DNS

Chrome, Edge, Brave and Firefox can have **Secure DNS / DNS over HTTPS** enabled with their own provider. That skips AdGuard Home completely.

- Edge: `edge://settings/privacy` → *Use secure DNS*
- Brave: *Settings → Privacy and security → Security → Use secure DNS*

Turn it off or set it to use the system DNS. Symptom we hit: `nslookup doubleclick.net 192.168.1.2` returned `0.0.0.0`, but the site still loaded in the browser.

## 5. Blocklists show 0 rules / reset after reboot

Two separate causes.

### a) The working directory is in RAM

The package defaults the **Working directory** to `/var/lib/adguardhome`. On OpenWrt `/var` is a symlink to `/tmp` (RAM), so downloaded filter lists, stats and the query log are wiped on every reboot. The config survives because it lives in `/etc/adguardhome/`.

Check at **Services → AdGuard Home** (**Working directory**).

### b) The ONT takes about 40 s to dial PPPoE

OpenWrt boots faster than the ONT. AdGuard Home starts, tries to download its lists before the internet is up, fails, and does not retry until its next scheduled update (24 h by default).

### Fix options

**Option A: keep the data on flash.** Set **Working directory** to `/etc/adguardhome-data` at **Services → AdGuard Home** and **Save & Apply**. Lists persist across reboots.

Trade-off is flash wear: the query log, statistics and list updates write to internal flash. To keep it low, set query log and statistics retention to 24 hours (or turn the query log off) under **Settings → General settings**. The lists themselves are a few MB written about once a day. A USB drive or SD card as the working directory avoids the wear entirely if the router has a port.

**Option B: keep it in RAM, but wait for the internet at boot.** Add this to **System → Startup → Local Startup**, above `exit 0`:

```sh
(
  for i in $(seq 1 60); do
    ping -c1 -W2 1.1.1.1 >/dev/null 2>&1 && break
    sleep 5
  done
  /etc/init.d/adguardhome restart
) &
```

This waits up to 5 minutes for the internet, then restarts AdGuard Home so it re-downloads the lists. Expect a few minutes without filtering after each reboot, and no flash writes.

### Before you reboot the router

The ONT hands out only `192.168.1.2` as DNS, so if AdGuard Home does not start, the whole network loses DNS. Confirm `adguardhome` is **Enabled** under **System → Startup** and test one reboot.

If the *AdGuard DNS filter* still shows 0 rules while another list loads, click **Check for updates** again and check **Status → System Log**.

## 6. Verification checklist

```
ipconfig /flushdns
nslookup doubleclick.net 192.168.1.2     # expect 0.0.0.0 / ::
nslookup doubleclick.net                 # same, using the system DNS
ipconfig /all                            # DNS servers: only AdGuard Home addresses
```

- **Filters → DNS blocklists**: rule counts are non-zero (the AdGuard DNS filter is tens of thousands of rules).
- Opening `doubleclick.net` in the browser gives `DNS_PROBE_POSSIBLE`.
- **Query Log** shows your devices as clients and blocked entries.
- Reboot OpenWrt once and confirm the lists are still loaded.

## 7. Other devices that can bypass it

- **Android:** Settings → Private DNS → Off or Automatic.
- **iPhone:** turn off iCloud Private Relay for your Wi-Fi.
- **Smart TVs / streaming boxes** often hardcode `8.8.8.8`. With OpenWrt acting only as a switch, DNS cannot be redirected, so this is a known limitation.

## 8. Troubleshooting summary

| Symptom | Cause | Fix |
|---|---|---|
| `nslookup` returns real IP for ad domains | Lists empty (0 rules) | Section 5 |
| `nslookup` blocks, browser loads site | Browser Secure DNS | Section 4 |
| BSNL `2001:4490:...` DNS in `ipconfig /all` | ONT advertises WAN IPv6 DNS | Section 3 |
| Lists empty after every reboot | Working dir in RAM | Section 5a |
| Lists empty only after a power cut | ONT PPPoE not up yet | Section 5b |
| Router cannot download lists | Wrong clock or no gateway/DNS | Check NTP, LAN gateway `192.168.1.200`, bootstrap DNS |
| Whole network has no DNS | AdGuard Home not running | Check **System → Startup**; restart the service |

## Notes

- Back up the OpenWrt config (**System → Backup / Flash Firmware**) and `AdGuardHome.yaml` after you finish tuning.
- Before a firmware upgrade, use Attended Sysupgrade so installed packages are kept. A plain upgrade can remove AdGuard Home and take down DNS for the whole network.
- Menu names and package defaults may vary between OpenWrt and AdGuard Home versions. This was done on OpenWrt 25.12.5 with AdGuard Home 0.107.80.
