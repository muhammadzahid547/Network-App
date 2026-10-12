
# NetSentry

Network monitoring app for Android (tested target: Xiaomi Redmi Note 14, Android 14/15, no root needed).

## Features
1. **Endpoint/API monitoring of the phone itself** (works on BOTH WiFi and Cellular):
   - Local VPN engine (`NetWatchService`) intercepts every IPv4 packet, runs a small
     user-space NAT, and logs: destination IP, port, protocol (TCP/UDP/TLS), TLS SNI
     domain name, bytes, and underlying transport (wifi/cellular).
2. **Infiltration detection on your LAN**: ARP + ping + TCP-port sweep of your subnet.
   Every MAC address is remembered; any **new, unknown MAC** triggers a high-priority
   "possible infiltration" notification.
3. CSV export of all captured endpoints.

## Build
- Open in Android Studio (or use CLI):
  ```
  gradle wrapper --gradle-version 8.7   # once
  ./gradlew assembleDebug
  ```
- Install `app/build/outputs/apk/debug/app-debug.apk`.
- On the phone: allow install from unknown sources, open app, press **Start monitoring VPN**,
  accept the VPN consent dialog. Set Battery = "No restrictions" for the app in system settings.

## Architecture
- `NetWatchService` (VpnService): TUN fd -> parse IPv4 -> UDP NAT (`UdpFlow`) / minimal TCP relay (`TcpFlow`) -> `EndpointDb` (SQLite).
- `LanScanner`: gateway from `LinkProperties`, parallel host sweep, `/proc/net/arp` for MACs, OUI vendor guess.
- `DeviceStore`: SharedPreferences MAC set; diff = new-device alert.
- Tabs: Endpoints (live log), Devices (LAN scan + NEW badges), Status (controls, stats, export).

## Honest limitations (IMPORTANT)
- **Only your phone's own traffic** is visible. A phone in normal WiFi client mode physically
  cannot see other devices' frames. To see ALL LAN devices' endpoints you need one of:
  - a rooted phone doing ARP spoofing (e.g. `arpspoof` + `tcpdump`), or
  - router-level capture (OpenWrt `tcpdump`, Pi-hole, ASUS/Merlin traffic analyzer), or
  - port mirroring on a managed switch.
- **An outsider who has NOT joined your network cannot be detected at all** — that needs
  monitor-mode WiFi (your chipset does not support it). This app detects outsiders *after
  they join*, which is the realistic threat model for home networks.
- The built-in TCP relay is a best-effort minimal stack: no TCP options (MSS defaults to
  536 -> slower bulk transfers), no retransmission/window scaling. It works for API calls,
  browsing and most apps; heavy streaming may be slow. If you only need capture, consider
  routing analysis instead of full forwarding.
- IPv6 traffic bypasses the VPN (only `0.0.0.0/0` is routed). DNS-over-HTTPS hides domain
  names (you will still see the destination IP).
- QUIC (HTTP/3) appears as UDP to Google/Cloudflare IPs.

## Root upgrade path (optional)
With root you can extend the app to dump full pcaps of the phone:
`su -c "tcpdump -i any -w /sdcard/capture.pcap"` — but even then, *other* devices' traffic
requires ARP spoofing or router cooperation as described above.

## v2 — zero-root, zero-network-change defense layer

Everything below works on a stock Redmi Note 14, no root, no router changes:

| Feature | How it works | File |
|---|---|---|
| Passive DNS capture | DNS queries pass through the tunnel as UDP; QNAMEs are parsed | `Dns.kt` |
| Endpoint reputation | Embedded tracker/crypto-miner lists + offline classification | `Intel.kt` |
| DGA detection | Shannon entropy + vowel-ratio heuristic on domain names | `Intel.kt` |
| ARP-spoofing / MITM watch | Gateway IP-MAC binding monitored continuously; a silent MAC change alerts | `ArpWatch.kt` |
| Continuous rogue-MAC watch | ARP table diffed every 15 s while VPN runs | `ArpWatch.kt` |
| Deep device fingerprinting | mDNS (NSD), SSDP/UPnP, HTTP banners, TLS cert CNs | `Discovery.kt` |

Flagged endpoints get a colored badge (TRACKER / MINER / MALWARE / DGA) and a
rate-limited notification (max 1 per item per 10 min, no spam).

### Why this is the realistic ceiling without root
- Phone traffic: fully visible (VPN engine) — DNS, SNI, ports, IPs.
- Other LAN devices: identities + open services visible (scan + fingerprinting),
  but their *traffic content* is invisible without ARP spoofing (root) or a
  router capture point.
- Off-network attackers: invisible to any phone; MITM on your own LAN is the
  attack this build actually detects (gateway MAC flip + rogue MAC join).

## v3 — whole-network endpoint visibility via router feed (Pi-hole / OpenWrt)

The Status tab now has a router-feed panel. Three sources, all read-only:

1. **Pi-hole v5**: paste API token -> app pulls `getAllQueries` every 60 s.
2. **Pi-hole v6**: paste FTL password -> session auth -> `/api/queries`.
3. **Syslog (OpenWrt or any router)**: enable log forwarding to the phone's IP;
   `SyslogListener` parses dnsmasq `query[A] ... from <client>`, gravity `blocked`
   lines, and `DHCPACK ... <ip> <mac>` leases (new leases auto-alert and are
   learned into the device store).

All router-fed entries are stored with `src = <client IP>`, shown in the
Endpoints tab as "from 192.168.1.x" with a "LAN (router)" tag, classified by
the same Intel engine as phone traffic, and counted separately (lanLogged).

Combined coverage: phone traffic (VPN engine), every other LAN device's DNS
endpoints (Pi-hole/syslog), rogue-device + MITM detection (ARP watch),
device fingerprinting (deep scan). That is the full realistic set for
no-root phone + read-only router access.

## I just want the APK (no tools installed)

Fastest path - GitHub builds it for you, free, ~8 minutes:

1. Create a free account at github.com, then **New repository** (any name, private is fine).
2. On the repo page choose **uploading an existing file**, drag the entire
   unzipped `NetSentry` folder contents in, Commit.
3. Open the **Actions** tab -> **Build NetSentry APK** -> **Run workflow**.
4. When the run finishes (green check), open it -> **Artifacts** ->
   download `NetSentry-APK.zip` -> extract `app-debug.apk`.
5. Copy the APK to your Redmi Note 14, tap it, allow "Install from unknown
   sources" when asked, install, open, press **Start monitoring VPN**, accept
   the VPN dialog, then set the app's Battery usage to **No restrictions**
   in MIUI settings so the background service survives.

The APK is a debug build signed with a debug key - fine for personal use.

Alternative: open the project in Android Studio and use
Build -> Build Bundle(s)/APK(s) -> Build APK(s).
