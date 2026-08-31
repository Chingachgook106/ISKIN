# RU-001 — Independent Network Architecture Review

## Role

You are an independent network/security architect reviewing the proposed network architecture for a single Windows 11 Pro engineering node.

Do not assume that any particular solution is preferred.

Your task is to independently evaluate three architectural approaches:

1. System-level VPN
2. Proxy-based architecture
3. Overlay-network architecture

The objective is not to recommend a commercial product at this stage.

The objective is to determine which architectural class best satisfies the requirements below.

---

# 1. System under review

Node:

- EMS-001 / "Baby"
- Windows 11 Pro 25H2
- x64 Beelink 5800H
- Ethernet is the only active network interface
- Wi-Fi and Bluetooth are currently disconnected
- Current network transit is IPv4-only
- No global IPv6 address
- No IPv6 default route
- Single IPv4 default route through the local gateway
- Git and VS Code are installed and operational
- GitHub HTTPS connectivity is currently functional
- No VPN, proxy, or overlay is currently installed

The attached document:

`RU-001-Network-Baseline-001.md`

is the authoritative source for the current network state.

Do not invent configuration details that are not present in the baseline.

---

# 2. Architectural objective

RU-001 is a relocation profile.

The node must remain an independent engineering node when physically moved between different networks and network environments.

The architecture must not depend on:

- a particular ISP;
- a particular local network;
- a particular IP address;
- a particular physical location;
- a particular application-specific VPN implementation.

The network security/privacy layer should exist below the application layer whenever technically practical.

Git, VS Code and Terminal should not each require their own independent network-security configuration.

---

# 3. Primary requirement

The most important requirement is:

## FAIL-CLOSED

If the protected network path becomes unavailable, the node must NOT silently fall back to direct Internet connectivity.

Required behaviour:

```text
Protected path available
        ↓
Internet traffic allowed through protected path


Protected path unavailable
        ↓
Direct Internet route must NOT become available
        ↓
Internet traffic blocked
```

The reviewer must explain exactly how this property can be implemented and, more importantly, how it can be empirically verified.

Do not treat "VPN disconnected" as equivalent to "fail-closed".

---

# 4. Security and privacy requirements

Evaluate each architecture for:

### Network security

- protection of traffic in transit;
- resistance to route manipulation;
- resistance to accidental direct Internet fallback;
- interaction with Windows firewall/routing;
- attack surface introduced by the architecture.

### Privacy

Evaluate:

- public IPv4 exposure;
- DNS exposure;
- IPv6 exposure;
- metadata exposure;
- destination visibility;
- local-network exposure.

Do NOT equate privacy with anonymity.

Do NOT claim that VPN/proxy/overlay automatically provides anonymity.

---

# 5. DNS

This is a critical evaluation area.

The current baseline shows that DNS queries are sent directly to public resolvers.

Determine for each architecture:

- where DNS queries originate;
- which resolver receives them;
- whether DNS can bypass the protected path;
- whether DNS behaviour changes during connection loss;
- whether DNS can continue working while Internet traffic is blocked;
- whether encrypted DNS can be enforced;
- whether Windows DNS behaviour creates possible leakage paths.

Explicitly analyse Windows name-resolution behaviour.

---

# 6. IPv4 and IPv6

Evaluate IPv4 and IPv6 separately.

The current node has no global IPv6 connectivity, but the architecture must be evaluated against the possibility that IPv6 becomes available after moving the node to another network.

Determine:

- whether the architecture controls IPv4;
- whether it controls IPv6;
- whether IPv6 can bypass the protected path;
- what happens if IPv6 becomes available unexpectedly;
- whether fail-closed remains valid in a future dual-stack network.

---

# 7. Routing

For each architecture, explain:

- where routing decisions are enforced;
- whether the physical interface retains a usable default route;
- whether split tunnelling is possible;
- whether force tunnelling is possible;
- how exceptions are handled;
- how the protected endpoint itself remains reachable;
- what happens to routes after reconnect;
- what happens after reboot;
- what happens when the physical network changes.

Identify all potentially dangerous routing states.

---

# 8. Application independence

Evaluate whether the following can operate normally without application-specific network configuration:

- Git
- VS Code
- PowerShell
- Command Prompt
- SSH
- HTTPS/API clients
- ordinary Windows services

The desired architecture is:

```text
             Network Security Layer
                       │
        ┌──────────────┼──────────────┐
        │              │              │
       Git          VS Code        Terminal
        │              │              │
        └──────────────┼──────────────┘
                       │
                 protected path
                       │
                    Internet
```

Explain where each candidate does and does not satisfy this model.

---

# 9. Failure analysis

For every candidate, analyse at minimum:

1. Protected tunnel/path unavailable at boot
2. Protected path drops after successful connection
3. Authentication failure
4. DNS failure
5. DNS resolver unavailable
6. Internet unavailable
7. Local gateway unavailable
8. Network changes from one Wi-Fi/Ethernet environment to another
9. IPv6 unexpectedly becomes available
10. Application starts before the protected path is established
11. Windows restarts networking
12. System reboot
13. Sleep/wake
14. Protected service crashes
15. Configuration corruption

For every case answer:

- What happens?
- Is traffic blocked?
- Can direct traffic escape?
- Does the system recover automatically?
- Does the user receive a clear indication?
- Is manual intervention required?

---

# 10. Mobility

RU-001 must work as a relocation profile.

Evaluate behaviour when EMS-001 moves between:

- home LAN;
- office LAN;
- public network;
- mobile hotspot;
- restricted network;
- network with different DNS;
- network with IPv6 enabled.

Do not assume that the current baseline network will remain available.

---

# 11. Compute Node compatibility

The future ISKIN architecture will contain:

- EMS-001 as Control / Orchestration Node;
- external Compute Node as an independent expandable compute resource.

The network architecture should therefore allow a future secure communication path between EMS-001 and a Compute Node.

Do not design the current solution specifically for a Compute Node.

Instead determine whether each candidate provides a clean architectural extension point.

---

# 12. Operational complexity

Evaluate:

- installation complexity;
- configuration complexity;
- maintenance;
- diagnostics;
- recovery;
- dependency on third-party software;
- dependency on external infrastructure;
- Windows integration;
- reproducibility;
- documentation requirements.

---

# 13. Security boundary

For each candidate identify exactly where the security boundary exists.

For example:

```text
Application
    ↓
Operating System
    ↓
Network Security Layer
    ↓
Physical Interface
    ↓
Local Network
    ↓
ISP
    ↓
Internet
```

Mark the exact layer at which the architecture enforces its guarantees.

If a guarantee cannot actually be enforced at that layer, state this explicitly.

---

# 14. Evidence requirement

Separate your conclusions into:

### A. Established facts

Claims supported by official technical documentation or experimentally verified behaviour.

### B. Engineering inference

Conclusions derived from known mechanisms but not directly experimentally verified on EMS-001.

### C. Assumption

A condition that must be verified before implementation.

### D. Unknown

A question that cannot currently be answered without testing.

Do not present assumptions as facts.

Prefer primary technical documentation over marketing material.

For Windows-specific behaviour, consult Microsoft technical documentation where applicable.

---

# 15. Comparison matrix

Produce a comparison table:

| Criterion | System VPN | Proxy | Overlay |
|---|---|---|---|
| Full system coverage | | | |
| DNS protection | | | |
| IPv4 protection | | | |
| IPv6 protection | | | |
| Fail-closed | | | |
| Routing control | | | |
| Application independence | | | |
| Windows integration | | | |
| Reboot behaviour | | | |
| Network mobility | | | |
| Failure recovery | | | |
| Local LAN compatibility | | | |
| Security boundary clarity | | | |
| Operational complexity | | | |
| Future Compute Node path | | | |
| Verification difficulty | | | |
| Main failure mode | | | |

Use:

- Strong
- Adequate
- Weak
- Unsuitable
- Unknown

Do not assign numerical scores unless you can justify the scoring methodology.

---

# 16. Adversarial review

After evaluating all three candidates, attempt to disprove your own preferred solution.

Identify:

- hidden assumptions;
- dangerous edge cases;
- misleading security guarantees;
- possible bypasses;
- operational failure modes;
- situations in which the architecture appears secure but is not.

Then state:

> "The strongest argument against my preferred architecture is..."

This section is mandatory.

---

# 17. Final recommendation

Only after completing the independent analysis, provide:

1. Preferred architectural class
2. Second-best alternative
3. Rejected alternative
4. Why
5. Conditions under which the preferred architecture should NOT be selected
6. What must be experimentally verified on EMS-001 before implementation

Do not recommend a specific commercial VPN provider unless necessary to demonstrate a technical property.

Do not provide implementation commands at this stage.

The output is an architectural research report, not an installation guide.

---

# 18. Important restriction

Do not assume that:

- VPN = fail-closed;
- encrypted tunnel = privacy;
- proxy = system-wide protection;
- overlay = security boundary;
- absence of IPv6 today means IPv6 can be ignored permanently.

Each claim must be examined technically.

---

# 19. Deliverable

Return a structured technical report suitable for comparison with two independent expert reports.

Recommended structure:

1. Executive conclusion
2. Current-state interpretation
3. System VPN analysis
4. Proxy analysis
5. Overlay analysis
6. Failure-mode analysis
7. DNS analysis
8. IPv4/IPv6 analysis
9. Mobility analysis
10. Compute Node implications
11. Comparison matrix
12. Adversarial critique
13. Recommendation
14. Experimental verification plan
15. Open questions
16. Sources

Do not modify the attached baseline.
Do not propose implementation changes to EMS-001.
Do not execute any commands.