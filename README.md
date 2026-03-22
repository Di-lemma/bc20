# bc20
### A RabbitMQ-backed RAT Masquerading as Adware

---

## Summary

This RAT enables arbitrary shell execution, used a robust multi-process watchdog architecture, and operated through a command-and-control (C2) infrastructure based on AMQP/RabbitMQ. The payload, identified as `bc20`, primarily monetized through ad injection while providing the operator with continuous remote shell access.

The C2 domain (`ogorax.xyz`) was inactive at the time of discovery, indicating the campaign may have been dormant for an extended period, possibly years.

---

## Infection Artifacts

### LaunchAgents (persistence layer)

| Plist | Binary | Role |
|---|---|---|
| `com.bc20.plist` | `~/Library/bc20/bc20` | Primary RAT agent |
| `com.rp2.plist` | `~/Library/rp2/rp2` | Watchdog / resurrector (30s interval) |
| `com.Amritsar.plist` | `~/Library/Application Support/com.Amritsar/Amritsar` | Secondary agent / dropper |
| `com.SettingsExtd.plist` | `~/Library/Application Support/com.SettingsExtd/SettingsExtd` | Updater (12h interval) |
| `com.described.plist` | `~/Library/Application Support/com.described/described` | Updater (12h interval) |
| `com.QuicklookPI.plist` | `~/Library/FileFinder/QuicklookPI/QuicklookPI` | Attempted root escalation |
| `com.LinkBranch2.plist` | `~/Library/FileFinder/LinkBranch/LinkBrancha` | Unknown / likely dead |
| `com.poultryproof.plist` | *(empty — 0 bytes)* | Dead dropper remnant |

### C2 Configuration

Found in `~/Library/bc20/` (campaign directory):

```json
{
  "rabbit_mq_address": "amqps://username:password@www.ogorax.xyz:443/biz"
}
```

- **Protocol**: AMQPS (AMQP over TLS)
- **Port**: 443 — deliberately chosen to blend with HTTPS traffic
- **Virtual host**: `/biz`
- **Domain**: `ogorax.xyz` — unresolvable at time of discovery (campaign dead)

---

## Architecture

### C2 Transport: Why RabbitMQ?

The choice of RabbitMQ as the C2 bus is sophisticated and deliberate. Most unsophisticated malware uses direct TCP callbacks or HTTP polling to a C2 server, which is trivially detectable. RabbitMQ inverts the connection model:

```
[ogorax.xyz RabbitMQ broker]
        |
        |  amqps:443 (outbound from victim — looks like HTTPS)
        |
[bc20 on victim machine]
   subscribes to queue, blocks waiting for messages
```

The victim machine initiates an **outbound** connection to the broker and holds it open. This means:

- No inbound connections — firewalls see nothing suspicious
- Port 443 — indistinguishable from normal web traffic at the network layer
- TLS encrypted — deep packet inspection sees gibberish
- RabbitMQ is legitimate enterprise software — not flagged by heuristics
- Operator pushes one message; thousands of bots receive it simultaneously

The `/biz` virtual host likely existed alongside others (`/telemetry`, `/update`, etc.) partitioning different message types within the same broker infrastructure.

### Shell Execution

Static analysis of the `bc20` binary revealed the strings `popen` and `/bin/sh`, confirming the execution primitive:

```c
FILE* pipe = popen("/bin/sh -c <command_from_queue>", "r");
```

This gives the operator a full interactive-equivalent shell running as the victim user. No privilege escalation required — user-space access was sufficient to reach all valuable assets.

### Persistence Architecture

The infection used a layered resilience model:

```
[com.rp2]  ←— watchdog, polls every 30 seconds
    |
    └── if bc20 not running → restart it
    
[com.bc20]  ←— primary agent, KeepAlive: true
    |
    └── launchd auto-restarts on crash
    
[com.SettingsExtd / com.described]  ←— phone home every 12h
    |
    └── pull updated config / new payload versions
```

`KeepAlive: true` in bc20's plist means launchd itself acts as a babysitter — if the process dies for any reason, it is restarted automatically. `rp2` provides a second layer of resurrection independent of launchd behavior.

### Privilege Escalation Attempt

`com.QuicklookPI.plist` contains `UserName: root`, requesting that launchd run the binary as root. From a LaunchAgent (user-space plist), this request is ignored by modern macOS — LaunchDaemons in `/Library/LaunchDaemons/` can legitimately run as root, but LaunchAgents in `~/Library/LaunchAgents/` are user-scoped. This either represents a failed escalation attempt, a misconfigured plist, or a technique that worked on older macOS versions present on the 2017 hardware.

---

## Ad Injection Mechanism (Monetization Layer)

The adware component operated via a local HTTPS proxy:

1. Install proxy server on `localhost:<port>`
2. Rewrite system network proxy settings to route all traffic through it
3. Install a rogue root certificate into the system keychain to enable TLS MITM
4. Intercept HTTP responses and inject `<script>` tags before delivery to the browser
5. Browser renders page + injected ad content, sees valid certificate, shows green padlock

At time of analysis, `scutil --proxy` showed no active proxy configuration and the system keychain contained no rogue certificates, suggesting the network injection component had either been partially removed previously or failed to install correctly.

---

## Threat Assessment

The ad injection is the monetization mechanism. The shell access is the capability. These are orthogonal — the operator could have used the same `popen("/bin/sh")` primitive to:

- Exfiltrate files (`tar -czf - ~/Documents | curl -F ...`)
- Harvest browser credentials from profile directories
- Capture keystrokes
- Pivot to other machines on the local network
- Deploy ransomware
- Use the machine as a proxy node for further attacks

No evidence of any of the above was found, consistent with an adware-focused campaign. However, the infrastructure made no architectural distinction between "show an ad" and "exfiltrate a file" — both are simply messages on the queue.

---

## Timeline

| Date | Event |
|---|---|
| ~2020–2021 | Initial infection, based on plist timestamps |
| Unknown | `ogorax.xyz` domain lapses, C2 goes dark |
| Mar 13, 2026 | LaunchAgents directory inspected, bc20 discovered |
| Mar 14, 2026 | RabbitMQ config JSON extracted from bc20 campaign directory |
| Mar 14, 2026 | `ogorax.xyz` confirmed dead (no DNS A record) |
| Mar 14, 2026 | All LaunchAgents identified; remediation initiated |

---

## Remediation

```bash
# Unload all agents
for plist in com.bc20 com.rp2 com.Amritsar com.SettingsExtd \
             com.described com.QuicklookPI com.LinkBranch2 com.poultryproof; do
    launchctl unload ~/Library/LaunchAgents/$plist.plist 2>/dev/null
    rm ~/Library/LaunchAgents/$plist.plist 2>/dev/null
done

# Remove binaries
rm -rf ~/Library/bc20
rm -rf ~/Library/rp2
rm -rf ~/Library/FileFinder
rm -rf ~/Library/Application\ Support/com.Amritsar
rm -rf ~/Library/Application\ Support/com.SettingsExtd
rm -rf ~/Library/Application\ Support/com.described

# Verify no processes remain
ps aux | grep -E "bc20|rp2|Amritsar|QuicklookPI|SettingsExtd|described|LinkBranch"

# Verify proxy clean
scutil --proxy

# Verify keychain clean
security find-certificate -a -p | openssl x509 -noout -subject 2>/dev/null | grep -v Apple
```

---

## Indicators of Compromise

- `~/Library/bc20/` directory
- `~/Library/rp2/` directory  
- `~/Library/FileFinder/` directory
- `~/Library/Application Support/com.Amritsar/`
- `~/Library/Application Support/com.SettingsExtd/`
- `~/Library/Application Support/com.described/`
- C2: `amqps://*@www.ogorax.xyz:443/biz`
- LaunchAgent labels: `bc20`, `rp2`, `QuicklookPI`, `LinkBranchb`, `com.Amritsar`, `com.SettingsExtd`, `com.described`

---

## Conclusion

bc20 appears to be a RAT-class backdoor or botnet agent that combined adware-style monetization with arbitrary shell command execution in user space. The recovered artifacts show durable persistence, a resilient watchdog/update structure, and AMQPS-based C2 using RabbitMQ on port 443. While some higher-level campaign behavior remains inferential, the execution primitive and persistence model are sufficient to treat the infection as a serious compromise rather than mere nuisance adware. 

The most operationally significant finding is that **the infection had been present since approximately 2020–2021 and went undetected for 4–5 years**, surviving multiple macOS updates and routine system use. This is not a criticism of the user — XProtect is signature-based, the agent used outbound-only connections on port 443, and the Pirrit family of adware is specifically engineered to avoid behavioral detection heuristics.

*Discovered and analyzed by sernyl / 養心堂, March 2026.*
