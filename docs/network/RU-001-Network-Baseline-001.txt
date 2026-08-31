RU-001-Network-Baseline-001.md
Date / time: 2026-08-11 12:14:29 (Local Time)
EMS-001 identification: 

    Host Name: BEELINK5800H
    OS: Microsoft Windows 11 Pro (Build 26200 / 25H2)
    System Type: x64-based PC (Beelink 5800H)

Interface state
Interface
	
Description
	
Status
	
Link Speed
Ethernet 4
	
Realtek PCIe GbE Family Controller
	
Up
	
1 Gbps
Беспроводная сеть 2
	
MediaTek Wi-Fi 6 MT7920
	
Down (APIPA)
	
N/A
Сетевое подключение Bluetooth 2
	
Bluetooth Device (Personal Area Network)
	
Down (APIPA)
	
N/A
Подключение по локальной сети* 1/2
	
Virtual / Wi-Fi Direct adapters
	
Down (APIPA)
	
N/A
IPv4 configuration

    Active Interface: Ethernet 4
    IP Address: 192.168.0.102
    Prefix Length: 24 (Subnet 192.168.0.0/24)
    Prefix Origin: Dhcp (или Static, в зависимости от настроек роутера, факт: адрес стабилен).
    Note: All other interfaces hold APIPA addresses (169.254.x.x), confirming they are disconnected from Layer 2.

IPv6 configuration

    Global IPv6: Absent.
    Link-Local IPv6: Present on all interfaces (fe80::/64).
    Status: IPv6 stack is enabled in the OS, but no global addresses are assigned by the ISP or router.

Default gateway

    IPv4 Default Gateway: 192.168.0.1 via Ethernet 4 (Metric: 256).
    IPv6 Default Gateway: None (::/0 route is missing).

DNS

    Active DNS Servers (Ethernet 4): 8.8.8.8, 1.1.1.1 (Public resolvers).
    Wi-Fi DNS: 192.168.0.1 (Inactive).
    Virtual Interfaces DNS: fec0:0:0:ffff::1, ::2, ::3 (Standard Windows IPv6 site-local stubs).
    Suffix Search List: Empty (UseSuffixSearchList : False).
    Resolution Capability: Fully functional for IPv4 (A records).

Routing table

    IPv4: Standard clean table. Local subnet (192.168.0.0/24), Loopback (127.0.0.0/8), Multicast (224.0.0.0/4), Broadcast (255.255.255.255/32). Single default route via 192.168.0.1. No static overrides or policy routing.
    IPv6: Only Link-Local (fe80::/64), Loopback (::1/128), and Multicast (ff00::/8). No global routes.

Public IPv4

    Status: Present and active.
    Value: PUBLIC_IPV4_REDACTED (Verified via api.ipify.org and whoami.akamai.net).

Public IPv6

    Status: Absent / Not routed.
    Value: None (Verified via api6.ipify.org - timeout/failure).

HTTPS connectivity

    General HTTPS: Functional.
    GitHub HTTPS: github.com:443 -> TcpTestSucceeded : True. Remote Address: 140.82.121.4.

GitHub HTTPS connectivity

    DNS Resolution (A): Successful (140.82.121.4).
    DNS Resolution (AAAA): Returns SOA record (GitHub does not serve AAAA records for the main domain).
    TCP 443: Open and reachable.
    Note: Git push/pull not executed per TZ restrictions.

DNS test

    Default Resolver Test: Query to whoami.akamai.net successfully returned PUBLIC_IPV4_REDACTED.
    Direct OpenDNS Test: Query to resolver1.opendns.com returned no output (silent drop/timeout).

IPv4/IPv6 observations

    Network transit is strictly IPv4. 
    IPv6 is enabled at the OS level but completely inert at the network layer (no global IP, no default route, no IPv6 DNS).
    Wi-Fi and Bluetooth radios are powered on but not connected to any network.

Potential leaks (Baseline State)

    DNS Exposure (High): DNS queries are sent in plaintext (UDP 53) directly to public resolvers (8.8.8.8, 1.1.1.1). The real public IPv4 is exposed to these third-party entities.
    IPv4 Exposure (High): Direct internet access without overlay/proxy. Destination servers see the real ISP-issued public IPv4.
    IPv6 Exposure (None): Since there is no global IPv6 routing, IPv6 leaks are currently impossible.
    Radio Exposure (Low/Medium): Wi-Fi and Bluetooth adapters are enabled. Even without an active connection, they periodically broadcast probe requests (Wi-Fi) or discovery packets (BT), which can be used for local tracking.

Anomalies

    OpenDNS Silent Drop: Direct UDP 53 queries to 208.67.222.222 (resolver1.opendns.com) yield no response, unlike queries to Google/Cloudflare. Possible ISP DPI filtering or routing blackhole.
    WMP Prompt in Explorer: Accessing SMB shares via the Windows "Network" (WSD) UI node triggers a false prompt to install Windows Media Player. Bypassed by using direct UNC paths (\\IP\Share).

Engineer conclusion
Baseline successfully captured. EMS-001 operates as a standard single-homed IPv4 client. The network stack is clean, with no hidden routes or unauthorized overlays. 
Critical finding for RU-001: The current configuration provides zero privacy or security against network-level monitoring. DNS and IP are fully exposed. 
The system is ready for the architectural selection of the RU-001 network layer (System VPN / Proxy / Overlay).
