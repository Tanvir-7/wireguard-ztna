# Project 3 — Zero-Trust Remote Access

> Built on top of [Project 2 — Enterprise Monitoring & Observability Stack](../project-2)

A fully self-hosted, production-grade remote access system built on WireGuard VPN and pfSense. Implements Zero-Trust principles through per-device cryptographic identity, role-based firewall enforcement, and least-privilege network segmentation — all running on a Proxmox homelab.

---

## Table of Contents

- [What Was Built](#what-was-built)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Zero-Trust Design Principles](#zero-trust-design-principles)
- [WireGuard Server — Tunnel Configuration](#wireguard-server--tunnel-configuration)
- [Per-Device Key Management](#per-device-key-management)
- [Role-Based Access Control — Firewall Rules](#role-based-access-control--firewall-rules)
- [Client Configuration](#client-configuration)
- [Validation](#validation)
- [Challenges Solved](#challenges-solved)
- [What I Learned](#what-i-learned)

---

## What Was Built

A complete Zero-Trust remote access platform that allows trusted devices to securely connect to the homelab infrastructure from anywhere on the internet:

| Capability | Implementation | Status |
|---|---|---|
| **Encrypted VPN Tunnel** | WireGuard on pfSense | Live |
| **Per-Device Identity** | Unique key pair per device | Live |
| **Role-Based Access** | pfSense firewall rules per VLAN | Live |
| **Server VLAN Access** | Allowed via WIREGUARD interface rules | Live |
| **User VLAN Isolation** | Blocked via WIREGUARD interface rules | Live |
| **Least-Privilege Enforcement** | Firewall default-deny with explicit allow | Live |
| **Encrypted Transport** | WireGuard ChaCha20 + Poly1305 | Live |

---

## Tech Stack

| Technology | Role |
|---|---|
| **Proxmox VE** | Hypervisor hosting the pfSense VM |
| **pfSense** | Firewall, VPN server, VLAN routing, RBAC enforcement |
| **WireGuard** | Modern VPN protocol — encrypted tunnel layer |
| **Android (WireGuard app)** | VPN client device |

---

## Architecture

```
+------------------------------------------------------------------+
|                        Proxmox Homelab                           |
|                                                                  |
|   +------------------------------------------------------------+ |
|   |                     pfSense VM                             | |
|   |                                                            | |
|   |   WireGuard Tunnel (tun_wg0)   Listen Port: 51820/UDP     | |
|   |   VPN Subnet: 10.0.99.0/24     Server IP:   10.0.99.1     | |
|   |                                                            | |
|   |   WIREGUARD Interface Rules:                               | |
|   |     [PASS]  10.0.99.0/24  -->  Server VLAN                | |
|   |     [BLOCK] 10.0.99.0/24  -->  User VLAN                  | |
|   |                                                            | |
|   |   WAN Rule:                                                | |
|   |     [PASS]  Any  -->  WAN:51820/UDP                       | |
|   +------------------------------------------------------------+ |
|          |                        |                              |
|          v                        v                              |
|   +--------------+        +--------------+                       |
|   | Server VLAN  |        |  User VLAN   |                       |
|   | (accessible) |        |  (blocked)   |                       |
|   +--------------+        +--------------+                       |
+------------------------------------------------------------------+
          ^
          | Encrypted WireGuard Tunnel (internet)
          |
+--------------------------+
|   Android Client         |
|   VPN IP: 10.0.99.2/32   |
|   WireGuard App          |
+--------------------------+
```

**Traffic flow:**
```
Android Device (mobile data / any network)
  |-- WireGuard handshake --> pfSense WAN :51820/UDP
  |-- Encrypted tunnel established
  |-- [ALLOWED] --> Server VLAN resources
  +-- [BLOCKED] --> User VLAN resources
```

---

## Zero-Trust Design Principles

This project implements the core tenets of Zero-Trust networking:

| Principle | Implementation |
|---|---|
| **Never trust, always verify** | Every device must present a valid cryptographic key to establish any connection |
| **Per-device identity** | Each device has a unique key pair — no shared credentials |
| **Least privilege** | VPN clients can only reach what firewall rules explicitly allow |
| **Micro-segmentation** | Server VLAN and User VLAN are treated as separate trust zones |
| **Assume breach** | User VLAN is blocked even for authenticated VPN clients |
| **Encrypted transport** | All traffic is encrypted end-to-end regardless of the underlying network |

---

## WireGuard Server — Tunnel Configuration

WireGuard was installed on pfSense via the Package Manager and configured as the VPN server endpoint.

| Property | Value |
|---|---|
| Listen Port | `51820` (UDP) |
| VPN Subnet | `10.0.99.0/24` |
| Server Interface Address | `10.0.99.1/24` |
| Protocol | WireGuard (UDP) |
| pfSense Interface | WIREGUARD (assigned from `tun_wg0`) |

The WireGuard interface was assigned as a dedicated pfSense interface (`WIREGUARD`) so that granular firewall rules could be applied specifically to VPN traffic — separating it cleanly from LAN, WAN, and VLAN traffic.

Screenshot: `wireguard-tunnel.png` — pfSense WireGuard tunnel configuration showing listen port and interface address



---

## Per-Device Key Management

Every client device receives a **unique cryptographic key pair** — public and private. The server stores only the public key. The private key never leaves the client device.

**Key generation workflow:**

1. A key pair is generated on the client device (WireGuard app)
2. The client's **public key** is registered as a peer in pfSense
3. Each peer is assigned a **dedicated IP** within the VPN subnet
4. The pfSense server's **public key** is embedded in the client config

**Peer assignment table:**

| Device | VPN IP | Allowed IPs |
|---|---|---|
| Android Phone | `10.0.99.2` | `10.0.99.2/32` |
| *(future device)* | `10.0.99.3` | `10.0.99.3/32` |
| *(future device)* | `10.0.99.4` | `10.0.99.4/32` |

Using `/32` per peer means each device is individually identifiable at the firewall level. Any single device can be revoked by deleting its peer entry without affecting any other device.

Screenshot: `wireguard-peers.png` — pfSense Peers tab showing the Android peer with its assigned VPN IP and public key

---

## Role-Based Access Control — Firewall Rules

Firewall rules on the `WIREGUARD` interface enforce which network segments authenticated VPN clients can reach. Rules are processed top-to-bottom — first match wins.

| Priority | Action | Source | Destination | Purpose |
|---|---|---|---|---|
| 1 | **Pass** | `10.0.99.0/24` | Server VLAN | Allow VPN clients to reach servers |
| 2 | **Block** | `10.0.99.0/24` | User VLAN | Prevent lateral movement to user network |

**WAN rule** (separate tab):

| Action | Protocol | Destination | Port | Purpose |
|---|---|---|---|---|
| **Pass** | UDP | WAN Address | `51820` | Accept inbound WireGuard connections |

This rule ordering ensures that even a fully authenticated VPN client cannot reach the User VLAN — authentication grants network entry, not blanket access. Each VLAN is treated as a separate trust boundary.

Screenshot: `firewall-wireguard-rules.png` — pfSense Firewall > Rules > WIREGUARD tab showing the pass and block rules in order

Screenshot: `firewall-wan-rule.png` — pfSense Firewall > Rules > WAN tab showing the UDP 51820 allow rule

---

## Client Configuration

The Android client was configured using the WireGuard app with the following parameters:

**Interface section:**

| Field | Value |
|---|---|
| Private Key | *(generated on device — never shared)* |
| Address | `10.0.99.2/32` |
| DNS | `1.1.1.1` |

**Peer section:**

| Field | Value |
|---|---|
| Public Key | pfSense WireGuard server public key |
| Endpoint | `<public-ip>:51820` |
| Allowed IPs | `0.0.0.0/0` (full tunnel — all traffic routed through VPN) |
| Persistent Keepalive | `25` seconds |

Setting `AllowedIPs = 0.0.0.0/0` routes all device traffic through the encrypted tunnel, ensuring no traffic leaks to the local network while connected.

Screenshot: `android-wireguard-config.png` — WireGuard app on Android showing the configured tunnel interface and peer settings

Screenshot: `android-wireguard-config.png` — WireGuard app on Android showing the configured tunnel interface and peer settings


---

## Challenges Solved

Real problems encountered and resolved during this project:

### WireGuard tunnel failing to activate on Android — "unable to turn tunnel on"
The tunnel interface had no IP address assigned. pfSense's WireGuard package requires the **Interface Address** to be explicitly added under the tunnel configuration (`10.0.99.1/24`). Without it, the tunnel has no local endpoint to bind to and the client cannot complete the handshake. Fixed by adding the interface address directly in the tunnel settings.

### pfSense WireGuard peer page has no public key generate button
Unlike some WireGuard implementations, pfSense does not generate client key pairs server-side. The public/private key pair must be generated on the **client device first**, with only the public key pasted into pfSense. Resolved by generating the key pair in the Android WireGuard app and registering the public key as a peer in pfSense.

### VPN connection failing from inside the home network
Attempting to connect to the public IP endpoint (`<public-ip>:51820`) from a device on the same LAN as pfSense failed due to NAT loopback not being supported. This is expected behaviour — VPN connectivity must be tested from outside the network (mobile data or a different location). Confirmed working correctly once tested on mobile data.

### WIREGUARD interface not enabled after assignment
After assigning the `tun_wg0` interface in pfSense, the interface was saved but the **Enable** checkbox was not checked. This caused the WIREGUARD firewall tab to be inactive and rules to have no effect. Fixed by opening the interface settings and explicitly enabling it before saving.

---

## What I Learned

- How **WireGuard's cryptographic key model** enforces device identity — the server only ever holds public keys, making per-device revocation clean and immediate
- The difference between **authentication** (proving identity via keys) and **authorization** (firewall rules determining what that identity can access) — Zero-Trust requires both
- How pfSense **interface assignment** works and why it is necessary to apply granular firewall rules to a VPN tunnel rather than relying on generic LAN rules
- How **firewall rule ordering** in pfSense determines policy — and why the block rule must come before any broad allow rule to be effective
- That **NAT loopback limitations** are a normal part of home network architecture, not a misconfiguration — and how to account for this when testing
- The practical meaning of **least-privilege network access** — a device being authenticated to the VPN does not mean it should reach every part of the network
- How **per-device VPN IPs (`/32`)** enable individual device tracking and revocation at the firewall level without touching any other device's access

---

> **Project 2:** [Enterprise Monitoring & Observability Stack](../project-2)
> **Project 4:** Coming soon
