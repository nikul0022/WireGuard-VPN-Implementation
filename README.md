# WireGuard-VPN-Implementation
A point-to-point-style WireGuard VPN built and tested on a lab network with one server (acting as a hub) and two client peers, demonstrating encrypted tunnel setup, key-based authentication, and verified connectivity between all nodes.

## Overview

WireGuard is a modern, open-source VPN protocol that uses public-key cryptography (Curve25519) for authentication and a minimal codebase compared to legacy VPN protocols like OpenVPN or IPsec. Instead of complex routing tables, WireGuard uses **Cryptokey Routing**: each peer's public key is bound to a list of allowed IP addresses, so the interface knows exactly which peer to encrypt/decrypt traffic for.

## Network Architecture

| Node | Role | Tunnel IP | Public Endpoint |
|---|---|---|---|
| Kali VM | WireGuard server / hub | 10.0.0.1/24 | 172.16.0.124:51820 |
| Client 1 (Windows) | Peer | 10.0.0.2/24 | 172.16.0.123 |
| Client 2 (Windows) | Peer | 10.0.0.3/24 | 172.16.0.122 |

            [ Kali VM - wg0 server ]
               10.0.0.1/24 : 51820
                /              \
    VPN tunnel /                \ VPN tunnel
               /                  \
[ Client 1 ]                  [ Client 2 ]
10.0.0.2/24                   10.0.0.3/24


<img width="1280" height="722" alt="image" src="https://github.com/user-attachments/assets/6552bf93-b97f-40d8-a077-a437ac2ef8a5" />

## Server Configuration (`/etc/wireguard/wg0.conf`)

```ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <server-private-key-redacted>

[Peer]   # Client 1
PublicKey = h0zqusT4llvHjmBei8wdYeCtxHjgfcg14eoVuBxtcCw=
AllowedIPs = 10.0.0.2/32

[Peer]   # Client 2
PublicKey = lLvZ+Iycht9X2CQP4/4nJQINmI1T7NWT11yBDXUrA3Y=
AllowedIPs = 10.0.0.3/32
```

## Client Configuration (example — Client 2)

```ini
[Interface]
PrivateKey = <client2-private-key-redacted>
Address = 10.0.0.3/24

[Peer]
PublicKey = XTv4XyheqAbHsLkIrt1MmR/IHbd8MSyB/XdSG7419VQ=
AllowedIPs = 10.0.0.0/24
Endpoint = 172.16.0.124:51820
```

Client 1 follows the same pattern with its own key pair and `Address = 10.0.0.2/24`.

## Setup Steps

1. Installed WireGuard on the Kali VM (`apt install wireguard`) and generated a server key pair with `wg genkey | tee server_private.key | wg pubkey > server_public.key`.
2. Created `/etc/wireguard/wg0.conf` on the server defining the interface address, listen port, and one `[Peer]` block per client with its public key and allowed IP.
3. Installed the WireGuard desktop client on both Windows machines, generated a key pair per client, and created a tunnel config pointing `Endpoint` at the server's public IP and port.
4. Brought the interface up (`wg-quick up wg0` on the server; **Activate** in the Windows client), confirmed the handshake with `wg show`.
5. Verified full connectivity by pinging across the tunnel in both directions.

## Proof of Working Implementation

**Server-side `wg show` output** — confirms both peers completed a handshake and are actively passing traffic:

interface: wg0
public key: XTv4XyheqAbHsLkIrt1MmR/IHbd8MSyB/XdSG7419VQ=
listening port: 51820

peer: h0zqusT4llvHjmBei8wdYeCtxHjgfcg14eoVuBxtcCw= # Client 1
endpoint: 172.16.0.123:53007
allowed ips: 10.0.0.2/32
latest handshake: 1 minute, 40 seconds ago
transfer: 6.92 KiB received, 5.38 KiB sent

peer: lLvZ+Iycht9X2CQP4/4nJQINmI1T7NWT11yBDXUrA3Y= # Client 2
endpoint: 172.16.0.122:51966
allowed ips: 10.0.0.3/32
latest handshake: 1 minute, 26 seconds ago
transfer: 18.61 KiB received, 8.09 KiB sent

**Client 1 → Server and Client 2, over the tunnel:**

C:\Users\testuser2>ping 10.0.0.1 (server)
Reply from 10.0.0.1: bytes=32 time<1ms TTL=64 ... 0% loss

C:\Users\testuser2>ping 10.0.0.3 (Client 2)
Reply from 10.0.0.3: bytes=32 time=1ms TTL=127 ... 0% loss

**Client 2 → Server and Client 1, over the tunnel:**

C:\Users\vagrant>ping 10.0.0.1 (server)
Reply from 10.0.0.1: bytes=32 time=1ms TTL=64 ... 0% loss

C:\Users\vagrant>ping 10.0.0.2 (Client 1)
Reply from 10.0.0.2: bytes=32 time=1ms TTL=127 ... 0% loss


All three nodes could reach each other with 0% packet loss, confirming the tunnel was correctly routing encrypted traffic end-to-end.

## Key Takeaways

- Practiced asymmetric key generation and distribution for a real VPN deployment (not just theory)
- Applied Cryptokey Routing by scoping each peer's `AllowedIPs` rather than using broad routing rules
- Verified the implementation with live diagnostics (`wg show`, `ping`) rather than assuming success from configuration alone
