---
title: "Axios Package Compromise"
date: 2026-04-09
updated: 2026-04-09T17:06:05-04:00
summary: "On March 31, 2026, North Korean threat actors (UNC1069/Sapphire Sleet) compromised the npm account of the lead Axios maintainer and published two backdoored versions (1.14.1 and 0.30.4) containing a cross-platform Remote Access Trojan delivered via a malicious dependency called plain-crypto-js."
---

# Axios npm Package Supply Chain Compromise

## Current Status

**CONFIRMED.** The attack is over. Both malicious versions have been removed from npm. The incident is in the post-mortem and long-term remediation phase. Anyone who installed axios@1.14.1 or axios@0.30.4 between March 31, 2026 00:21 UTC and approximately 03:15 UTC should assume their system was compromised.

**Confidence: HIGH** (corroborated by 15+ independent security vendors, the maintainer's own post-mortem, and attribution from Google Threat Intelligence Group and Microsoft Threat Intelligence).

---

## Findings

### What Happened

On March 31, 2026, a threat actor compromised the npm account of **jasonsaayman**, the lead maintainer of the Axios JavaScript HTTP client library. The attacker changed the account email to an attacker-controlled ProtonMail address (ifstap@proton.me) and used the hijacked account to publish two malicious versions of Axios to npm [source: https://www.aikido.dev/blog/axios-npm-compromised-maintainer-hijacked-rat].

Axios is one of the most popular packages in the npm ecosystem, with over 100 million weekly downloads and approximately 175,000 dependent projects [source: https://www.csoonline.com/article/4152696/attackers-trojanize-axios-http-library-in-highest-impact-npm-supply-chain-attack.html].

The attacker did not modify any Axios source code. Instead, they injected a new runtime dependency called **plain-crypto-js@4.2.1** into the package.json file. When developers ran `npm install axios`, npm automatically resolved and installed this malicious dependency, which executed a postinstall hook that deployed a cross-platform Remote Access Trojan (RAT) [source: https://unit42.paloaltonetworks.com/axios-supply-chain-attack/].

### Technical Analysis

**Attack Vector:** Compromised npm maintainer account credentials (long-lived access token). The malicious versions were published directly via the npm CLI, bypassing the project's GitHub Actions OIDC-signed CI/CD pipeline. Neither version had a corresponding commit, tag, or GitHub release [source: https://phoenix.security/axios-supply-chain-compromise-npm-rat-2026/].

**Malicious Dependency Chain:**
1. Compromised axios versions added `plain-crypto-js@4.2.1` as a runtime dependency
2. plain-crypto-js contained an obfuscated `setup.js` postinstall script
3. The script used two-layer encoding: reversed Base64 + XOR cipher with key `OrDeR_7077`
4. On execution, it detected the host OS and downloaded platform-specific payloads from C2 infrastructure
5. After payload delivery, the dropper performed anti-forensic cleanup: deleted setup.js, removed the postinstall hook, and replaced the corrupted package.json with a clean decoy file called package.md

[source: https://socket.dev/blog/axios-npm-package-compromised]

**Platform-Specific RAT Payloads:**

| Platform | Payload Location | Mechanism |
|----------|-----------------|-----------|
| macOS | `/Library/Caches/com.apple.act.mond` | C++ Mach-O binary, executed via AppleScript and /bin/zsh |
| Windows | `%PROGRAMDATA%\wt.exe` (disguised PowerShell), `%TEMP%\6202033.ps1` | VBScript launcher, persistence via registry Run key |
| Linux | `/tmp/ld.py` | Python RAT, executed via nohup in background |

[source: https://unit42.paloaltonetworks.com/axios-supply-chain-attack/]

**RAT Capabilities (WAVESHAPER.V2):**
- System fingerprinting (hostname, username, OS version, running processes, directory listings)
- HTTP POST beaconing every 60 seconds with Base64-encoded JSON data
- Command execution via four commands: `kill`, `runscript`, `peinject`, `rundir`
- Binary injection capability
- Hardcoded user-agent spoofing Internet Explorer 8 on Windows XP

[source: https://unit42.paloaltonetworks.com/axios-supply-chain-attack/]

**Social Engineering Origin:** The Axios team confirmed the attack began with a targeted social engineering campaign. The maintainer's npm access token was stolen, likely through a fake Microsoft Teams error message used as a phishing lure [source: https://www.clearphish.ai/news/axios-npm-hack-fake-teams-error-supply-chain-attack]. The attacker pre-staged the clean version of plain-crypto-js (v4.2.0) 18 hours before publishing the poisoned Axios versions, demonstrating operational planning [source: https://www.sans.org/blog/axios-npm-supply-chain-compromise-malicious-packages-remote-access-trojan].

### Timeline (reverse chronological, times converted to ET where applicable)

| When (ET) | Event |
|-----------|-------|
| Apr 2, 2026 | VentureBeat reports Huntress confirmed infections within 89 seconds of install |
| Apr 1, 2026 6:26 PM | Microsoft publishes mitigation blog, attributes attack to Sapphire Sleet (North Korea) |
| Apr 1, 2026 5:04 AM | SecurityWeek reports attribution to North Korean actors via stolen long-lived access token |
| Apr 1, 2026 | Socket.dev publishes updated deep-dive analysis |
| Mar 31, 2026 ~11:15 PM | npm removes both malicious versions (~03:15 UTC). Malicious versions were live for approximately 3 hours |
| Mar 31, 2026 ~4:49 PM | Bleeping Computer publishes report confirming RAT delivery |
| Mar 31, 2026 ~2:35 AM | The Hacker News publishes initial report |
| Mar 31, 2026 ~8:05 PM (Mar 30) | Socket.dev automated scanner flags axios@1.14.1 within 6 minutes of publication (00:05:41 UTC Mar 31) |
| Mar 30, 2026 ~9:00 PM | axios@0.30.4 published to npm (01:00 UTC Mar 31) |
| Mar 30, 2026 ~8:21 PM | axios@1.14.1 published to npm (00:21 UTC Mar 31) |
| Mar 30, 2026 ~7:59 PM | plain-crypto-js@4.2.1 (malicious) published to npm (23:59:12 UTC Mar 30) |
| Mar 30, 2026 ~early morning | plain-crypto-js@4.2.0 (clean staging version) published ~18 hours before attack |

### Discovery

The compromise was first detected by **Socket.dev's automated scanner** within six minutes of the first malicious version being published (00:05:41 UTC on March 31, 2026) [source: https://socket.dev/blog/axios-npm-package-compromised]. **StepSecurity's AI Package Analyst** and **Harden-Runner** tool also independently detected the compromise and flagged anomalous C2 contact in over 12,000 projects [source: https://www.stepsecurity.io/blog/axios-compromised-on-npm-malicious-versions-drop-remote-access-trojan].

Elastic Security Labs also detected the compromise through automated supply-chain monitoring on March 30, 2026 (UTC) [source: https://www.elastic.co/security-labs/axios-one-rat-to-rule-them-all].

### Response

- npm removed both malicious versions approximately 3 hours after publication [source: https://www.tenable.com/blog/faq-about-the-axios-npm-supply-chain-attack-by-north-korea-nexus-threat-actor-unc1069]
- The Axios maintainer (Jason Saayman) published a post-mortem on GitHub Issue #10636 confirming the account compromise and remediation steps [source: https://github.com/axios/axios/issues/10636]
- Major security vendors (Microsoft, Google, Palo Alto, Cisco Talos, Elastic, Sophos, Trend Micro, Bitdefender, Snyk, Socket, Huntress, StepSecurity, Wiz, Orca) published advisories and detection guidance

---

## Indicators of Compromise

### Network IOCs

| Type | Value |
|------|-------|
| C2 Domain | `sfrclak[.]com` |
| C2 IP | `142.11.206[.]73` |
| Secondary C2 Domain | `callnrwise[.]com` |
| C2 Port | `8000` |
| C2 URL (macOS) | `packages[.]npm[.]org/product0` (POST body) |
| C2 URL (Windows) | `packages[.]npm[.]org/product1` (POST body) |
| C2 URL (Linux) | `packages[.]npm[.]org/product2` (POST body) |
| Dropper URL | `http://sfrclak[.]com:8000/6202033` |

### File System IOCs

| Platform | Path |
|----------|------|
| macOS | `/Library/Caches/com.apple.act.mond` |
| Windows | `%PROGRAMDATA%\wt.exe` |
| Windows | `%APPDATA%\Local\Temp\6202033.ps1` |
| Windows | `%APPDATA%\Local\Temp\6202033.vbs` |
| Linux | `/tmp/ld.py` |
| All | `node_modules/plain-crypto-js/` directory |

### Package IOCs

| Package | Versions |
|---------|----------|
| axios | 1.14.1, 0.30.4 |
| plain-crypto-js | 4.2.0, 4.2.1 |
| @shadanai/openclaw | 2026.3.31-1, 2026.3.31-2 |
| @qqbrowser/openclaw-qbot | 0.0.130 |

### SHA256 Hashes (partial)

- `ad8ba560ae5c4af4758bc68cc6dcf43bae0e0bbf9da680a8dc60a9ef78e22ff7`
- `fcb81618bb15edfdedfb638b4c08a2af9cac9ecfa551af135a8402bf980375cf`

[source: https://socket.dev/blog/axios-npm-package-compromised, https://unit42.paloaltonetworks.com/axios-supply-chain-attack/]

### Attacker Account IOCs

- npm account email changed to: `ifstap@proton[.]me`
- User-agent string: Internet Explorer 8 on Windows XP (hardcoded in RAT)
- XOR key used in obfuscation: `OrDeR_7077`

---

## Affected Versions and Safe Versions

| Branch | Compromised Version | Safe Version (rollback to) |
|--------|-------------------|---------------------------|
| 1.x | 1.14.1 | 1.14.0 or 1.6.5 |
| 0.x | 0.30.4 | 0.30.3 or 0.28.0 |

[source: https://www.tenable.com/blog/faq-about-the-axios-npm-supply-chain-attack-by-north-korea-nexus-threat-actor-unc1069]

---

## Remediation Steps

### Immediate (if you installed a compromised version)

1. **Check your lockfiles** for axios@1.14.1, axios@0.30.4, or plain-crypto-js in any version
2. **Search systems for file artifacts**: `/Library/Caches/com.apple.act.mond` (macOS), `%PROGRAMDATA%\wt.exe` (Windows), `/tmp/ld.py` (Linux)
3. **Isolate affected systems** immediately if IOCs are found
4. **Assume all secrets on affected machines are stolen**: rotate npm tokens, SSH keys, AWS keys, CI/CD secrets, API keys, database credentials
5. **Rebuild from known-good state** rather than attempting in-place cleanup
6. **Block C2 traffic** to `sfrclak[.]com`, `callnrwise[.]com`, and `142.11.206.73` at network perimeter

### Preventive (for all Axios users)

1. **Downgrade** to safe versions: axios@1.14.0 (or 1.6.5) / axios@0.30.3 (or 0.28.0)
2. **Pin versions** in package-lock.json / yarn.lock / pnpm-lock.yaml
3. **Clear package caches**: `npm cache clean --force`, equivalent for yarn/pnpm
4. **Use `--ignore-scripts`** during CI/CD installations to prevent postinstall hooks
5. **Deploy EDR** on developer workstations
6. **Migrate secrets** from developer machines to secure vaults
7. **Audit npm token lifetimes**: enforce short-lived tokens and enable 2FA on publishing

---

## Confidence Assessment

| Claim | Confidence | Basis |
|-------|-----------|-------|
| Axios versions 1.14.1 and 0.30.4 were compromised | HIGH | Confirmed by maintainer post-mortem, npm removal, 15+ vendor reports |
| Attack vector was compromised maintainer account | HIGH | Confirmed in maintainer post-mortem (GitHub #10636) |
| Malicious dependency was plain-crypto-js | HIGH | Confirmed by Socket, Unit42, Snyk, StepSecurity, and others |
| RAT delivered to macOS, Windows, Linux | HIGH | Independent analysis by Elastic, Unit42, Huntress, Trend Micro |
| Attribution to North Korean actors (UNC1069/Sapphire Sleet) | HIGH | Attributed by Google Threat Intelligence Group (GTIG) and Microsoft Threat Intelligence based on WAVESHAPER.V2 malware and infrastructure overlaps |
| Social engineering via fake Teams error | MEDIUM | Reported by ClearPhish, referenced in maintainer post-mortem, but full details of the phishing lure not independently verified |
| Malicious versions live for ~3 hours | HIGH | Consistent across Socket, Tenable, Wiz, and npm records |
| 12,000+ projects had anomalous C2 contact | MEDIUM | Single source (StepSecurity); not independently confirmed by other vendors |

---

## Open Questions

1. **Exact download count**: How many developers actually installed the compromised versions during the 3-hour window? Estimates vary widely. StepSecurity reported 12,000+ projects with anomalous C2 contact, but the total number of affected machines is unknown.
2. **Token theft method**: The exact mechanism by which the long-lived npm token was stolen is partially confirmed (social engineering, likely fake Teams error) but the full kill chain from phishing to token exfiltration has not been fully published.
3. **Secondary payloads**: Were any post-exploitation payloads delivered after the initial RAT beaconing? No vendor has published evidence of lateral movement or data exfiltration beyond the initial C2 contact.
4. **CVE assignment**: No CVE has been referenced in any of the sources reviewed. It is unclear whether one will be assigned for this supply chain compromise.
5. **npm platform changes**: What specific technical changes will npm implement to prevent future account takeovers of high-impact packages?
6. **Related packages**: The @shadanai/openclaw and @qqbrowser/openclaw-qbot packages were flagged as related. The full scope of connected malicious packages may not yet be known.

---

## Sources

### Primary / Vendor Analysis
- Socket.dev: https://socket.dev/blog/axios-npm-package-compromised
- Palo Alto Unit 42: https://unit42.paloaltonetworks.com/axios-supply-chain-attack/
- Microsoft Security: https://www.microsoft.com/en-us/security/blog/2026/04/01/mitigating-the-axios-npm-supply-chain-compromise/
- Google Cloud (GTIG): https://cloud.google.com/blog/topics/threat-intelligence/north-korea-threat-actor-targets-axios-npm-package
- Cisco Talos: https://blog.talosintelligence.com/axois-npm-supply-chain-incident/
- Elastic Security Labs: https://www.elastic.co/security-labs/axios-one-rat-to-rule-them-all
- Wiz: https://www.wiz.io/blog/axios-npm-compromised-in-supply-chain-attack
- Snyk: https://snyk.io/blog/axios-npm-package-compromised-supply-chain-attack-delivers-cross-platform/
- StepSecurity: https://www.stepsecurity.io/blog/axios-compromised-on-npm-malicious-versions-drop-remote-access-trojan
- Huntress: https://www.huntress.com/blog/supply-chain-compromise-axios-npm-package
- Tenable: https://www.tenable.com/blog/faq-about-the-axios-npm-supply-chain-attack-by-north-korea-nexus-threat-actor-unc1069
- Sophos: https://www.sophos.com/en-us/blog/axios-npm-package-compromised-to-deploy-malware
- Bitdefender: https://www.bitdefender.com/en-us/blog/businessinsights/technical-advisory-axios-npm-supply-chain-attack-cross-platform-rat-deployed-compromised-account
- Trend Micro: https://www.trendmicro.com/en_us/research/26/c/axios-npm-package-compromised.html

### Maintainer Post-Mortem
- GitHub Issue #10636: https://github.com/axios/axios/issues/10636

### News Coverage
- The Hacker News: https://thehackernews.com/2026/03/axios-supply-chain-attack-pushes-cross.html
- The Hacker News (UNC1069): https://thehackernews.com/2026/04/unc1069-social-engineering-of-axios.html
- Bleeping Computer: https://www.bleepingcomputer.com/news/security/hackers-compromise-axios-npm-package-to-drop-cross-platform-malware/
- The Register: https://www.theregister.com/2026/03/31/axios_npm_backdoor_rat/
- Dark Reading: https://www.darkreading.com/application-security/axios-npm-package-compromised-precision-attack
- SecurityWeek: https://www.securityweek.com/axios-npm-package-breached-in-north-korean-supply-chain-attack/
- Malwarebytes: https://www.malwarebytes.com/blog/news/2026/03/axios-supply-chain-attack-chops-away-at-npm-trust
- CSO Online: https://www.csoonline.com/article/4152696/attackers-trojanize-axios-http-library-in-highest-impact-npm-supply-chain-attack.html
- VentureBeat: https://venturebeat.com/security/axios-npm-supply-chain-attack-rat-maintainer-token-2026
- InfoQ: https://www.infoq.com/news/2026/04/axios-supply-chain/
- Cybernews: https://cybernews.com/security/axios-npm-critical-supply-chain-compromise/

### Guides and Overviews
- SOCRadar: https://socradar.io/blog/axios-npm-supply-chain-attack-2026-ciso-guide/
- SANS Institute: https://www.sans.org/blog/axios-npm-supply-chain-compromise-malicious-packages-remote-access-trojan
- Aikido: https://www.aikido.dev/blog/axios-npm-compromised-maintainer-hijacked-rat
- Picus Security: https://www.picussecurity.com/resource/blog/axios-npm-supply-chain-attack-cross-platform-rat-delivery-via-compromised-maintainer-credentials

---

## Update History

| Date | Update |
|------|--------|
| 2026-04-09 5:06 PM ET | Initial report published |

---

## How This Report Was Generated

This report was compiled on April 9, 2026 using Claude Code's research agent. Sources were gathered via SearXNG deep search, news search, and technical search across multiple engines (Brave, DuckDuckGo, Bing). Key articles from Socket.dev, Unit42, Tenable, Microsoft, and the maintainer's GitHub post-mortem were fetched and analyzed for technical details. All claims are grounded in cited sources. Confidence levels reflect the number of independent corroborating sources for each claim.
