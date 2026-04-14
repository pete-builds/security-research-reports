---
title: "Prompt Injection Attacks Against Claude Code and Agentic Coding Assistants"
date: 2026-04-01
updated: 2026-04-01
summary: "Research findings on prompt injection vectors, published success rates, and defenses for Claude Code and agentic coding tools. Includes Anthropic system card data, MCP attack vectors, and academic research."
canary: RESEARCH-PI-20260401
---

# Prompt Injection Attacks Against Claude Code and Agentic Coding Assistants

## Current Status

- Anthropic publishes granular prompt injection success rates by surface (coding, tool use, browser, GUI) in system cards for Opus 4.5 and 4.6
- Claude Code in constrained coding environments shows **0% attack success rate** across 200 adaptive attempts (Opus 4.6 system card)
- MCP tool poisoning and cross-server exfiltration are the most serious emerging vectors for agentic coding tools
- Academic research finds Claude Code has the lowest vulnerability rating among major coding agents; Cursor is rated critical
- **The "1.6% prompt injection success rate" statistic could not be verified.** See Finding 1 for details.

---

## Finding 1: The 1.6% Statistic Does Not Appear in Published Sources

**Confidence: High**

After searching Anthropic system cards, blog posts, VentureBeat coverage, and third-party analyses, the specific claim that "Claude Code with Opus has a 1.6% prompt injection success rate" does not appear in any published source. The closest published figures are:

| Figure | Context | Source |
|--------|---------|--------|
| **0%** | Opus 4.6 in constrained coding environment, 200 adaptive attempts, no safeguards needed | Opus 4.6 System Card [source: [VentureBeat](https://venturebeat.com/security/prompt-injection-measurable-security-metric-one-ai-developer-publishes-numbers)] |
| **1%** | Opus 4.5 browser use, adaptive Best-of-N attacker, 100 attempts, with safeguards | Anthropic prompt injection defenses blog [source: [Anthropic Research](https://www.anthropic.com/research/prompt-injection-defenses)] |
| **1.4%** | Opus 4.5 Chrome extension, adaptive attacker, with new safeguards | Anthropic prompt injection defenses blog [source: [Anthropic Research](https://www.anthropic.com/research/prompt-injection-defenses)] |
| **4.7%** | Opus 4.5 combined direct+indirect injection, single attempt (k=1) | Opus 4.5 System Card [source: [Zvi Mowshowitz analysis](https://thezvi.substack.com/p/claude-opus-45-model-card-alignment)] |

The 1.6% figure may be a misremembering of the 1.4% browser use statistic, or may have originated from an informal discussion rather than a published source. It is not from the system cards.

---

## Finding 2: Anthropic's Published Prompt Injection Data by Surface

**Confidence: High**

Anthropic is the first major AI lab to publish granular, quantified prompt injection metrics broken out by deployment surface, attempt count, and safeguard configuration.

### Opus 4.6 System Card Results

| Surface | Safeguards | k=1 ASR | k=200 ASR |
|---------|-----------|---------|-----------|
| Constrained coding | None needed | 0% | 0% |
| GUI computer use (extended thinking) | None | 17.8% | 78.6% |
| GUI computer use (extended thinking) | With safeguards | -- | 57.1% |
| Browser use | With safeguards | ~2% failure | -- |

Key insight: coding environments are dramatically more resistant than GUI/browser surfaces because the attack surface is narrower (no visual rendering, no arbitrary web content).

### Extended Thinking Trade-off

On the Gray Swan ART benchmark, enabling extended thinking increased prompt injection success rates from 14.8% to 21.7%. Extended thinking gives the model more "reasoning space" that attackers can exploit. [source: [Zvi Mowshowitz](https://thezvi.substack.com/p/claude-opus-46-system-card-part-1)]

---

## Finding 3: Claude Code's Defense Architecture

**Confidence: High**

Anthropic documents a multi-layered defense system across three engineering blog posts and the official security docs.

### Permission-Based Architecture
- Read-only by default; sensitive operations require explicit approval
- Write access restricted to working directory and subdirectories
- Command blocklist blocks `curl`, `wget` by default
- Fail-closed matching: unrecognized commands default to requiring approval

[source: [Claude Code Security Docs](https://code.claude.com/docs/en/security)]

### Auto Mode Two-Layer Defense
1. **Input layer**: Server-side prompt injection probe scans all tool outputs (file reads, web fetches, shell output) before they enter the agent context. Suspicious content gets a warning annotation rather than blocking.
2. **Output layer**: Transcript classifier (running on Sonnet 4.6) evaluates each proposed action. Uses two stages: fast single-token filter, then chain-of-thought reasoning when flagged. Critically, assistant messages and tool results are stripped from what the classifier sees, preventing payload exposure.

Real-world performance on internal traffic: 0.4% false positive rate, 5.7% false negative rate on synthetic exfiltration attempts. [source: [Anthropic Engineering: Auto Mode](https://www.anthropic.com/engineering/claude-code-auto-mode)]

### Sandboxing
- Uses OS-level primitives (Linux bubblewrap, macOS seatbelt)
- Filesystem isolation: read/write only to working directory
- Network isolation: outbound connections only to approved servers via proxy
- Reduces permission prompts by 84% while enhancing security
- Key claim: "a compromised Claude Code can't steal your SSH keys, or phone home to an attacker's server"

[source: [Anthropic Engineering: Sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing)]

### Isolated Context for Web Fetches
WebFetch uses a separate context window to prevent injecting malicious prompts into the main agent context. [source: [Claude Code Security Docs](https://code.claude.com/docs/en/security)]

---

## Finding 4: MCP Server Attack Vectors

**Confidence: High**

Three major categories of MCP-based prompt injection attacks have been documented:

### Tool Poisoning (Invariant Labs)
Malicious instructions embedded in MCP tool descriptions are invisible to users but visible to AI models. A technique called "shadowing" allows a malicious server to modify how agents interact with other trusted servers. Successfully demonstrated exfiltration of SSH keys and MCP credentials from Cursor. [source: [Invariant Labs](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks)]

### MCP Sampling Attacks (Palo Alto Unit 42)
Three vectors via MCP sampling protocol:
1. **Resource theft**: Hidden instructions cause extra generation, draining API credits
2. **Conversation hijacking**: Persistent instructions injected via initial request affect all subsequent turns
3. **Covert tool invocation**: Prompt injection triggers unauthorized tool execution (e.g., file writes with sensitive data)

Core issue: MCP sampling lets servers control prompt content and request LLM completions directly, reversing the typical client-server trust model. [source: [Unit 42](https://unit42.paloaltonetworks.com/model-context-protocol-attack-vectors/)]

### GitHub MCP Cross-Repository Exfiltration (Docker)
An AI agent legitimately accessing a public repository gets prompt-injected via a crafted GitHub issue, then uses the same credentials to access and exfiltrate data from private repositories. [source: [Docker Blog](https://www.docker.com/blog/mcp-horror-stories-github-prompt-injection/)]

---

## Finding 5: Academic Research on Coding Agent Vulnerabilities

**Confidence: High**

### "Your AI, My Shell" (Liu et al., 2025)
First systematic red-teaming of agentic coding editors using MITRE ATT&CK framework. 314 unique attack payloads covering 70 techniques.

**Platform success rates:**
| Platform | ASR |
|----------|-----|
| Cursor Auto Mode | 83.4% |
| Cursor + Claude 4 | 69.1% |
| Cursor + Gemini 2.5 Pro | 76.8% |
| Copilot + Claude 4 | 52.2% |
| Copilot + Gemini 2.5 Pro | 41.1% |

Primary vector: poisoned `.cursor/rules` files that developers import. Attacks included credential theft via regex grep for API keys, user creation, and service manipulation. [source: [arXiv:2509.22040](https://arxiv.org/html/2509.22040v1)]

### Maloyan & Namiot SoK Paper (January 2026)
Systematic analysis synthesizing 78 studies (2021-2026). Three-dimensional taxonomy: delivery vectors, attack modalities, propagation behaviors.

**Platform vulnerability ratings:**
| Platform | Rating | Key Factor |
|----------|--------|------------|
| Claude Code | Low | Mandatory tool confirmation, no auto-approve, sandboxed MCP |
| Cursor | Critical | Auto-approve available, unsandboxed MCP, unvalidated .cursorrules |
| Copilot | High | CVE-2025-53773 privilege escalation via config poisoning |

Finding: "Attack success rates against state-of-the-art defenses exceed 85% when adaptive attack strategies are employed." Claude Code's mandatory confirmation loop is the primary differentiator. [source: [arXiv:2601.17548](https://arxiv.org/html/2601.17548v1)]

---

## Finding 6: Known CVEs Against Claude

**Confidence: Medium** (limited detail available on the CVE page)

CVE-2025-54794 and CVE-2025-54795 ("InversePrompt") describe a technique for manipulating Claude by crafting prompts that subvert safety mechanisms through inverted or contradictory instructions. Published by Cymulate in August 2025, modification date March 2026. Patch status unclear from available sources. [source: [Cymulate](https://cymulate.com/blog/cve-2025-547954-54795-claude-inverseprompt/)]

---

## Confidence Assessment

| Finding | Confidence | Basis |
|---------|-----------|-------|
| 1.6% statistic not found | High | Searched system cards, blog posts, third-party coverage across both Opus 4.5 and 4.6 |
| Anthropic published success rates | High | Verified against multiple independent analyses of system cards |
| Claude Code defense architecture | High | Directly from Anthropic engineering blogs and official docs |
| MCP attack vectors | High | Published by Palo Alto Unit 42, Invariant Labs, Docker |
| Academic vulnerability ratings | High | Peer-reviewed/preprint research with specific methodology |
| InversePrompt CVEs | Medium | CVE assigned but full technical detail not extracted |

---

## Sources

### Anthropic Official
1. [Claude Code Security Documentation](https://code.claude.com/docs/en/security) - Official security architecture and prompt injection defenses
2. [Mitigating the Risk of Prompt Injections in Browser Use](https://www.anthropic.com/research/prompt-injection-defenses) - 1% ASR with safeguards, defense techniques
3. [Claude Code Auto Mode: A Safer Way to Skip Permissions](https://www.anthropic.com/engineering/claude-code-auto-mode) - Two-layer defense system, 0.4% FPR, 5.7% FNR
4. [Making Claude Code More Secure and Autonomous (Sandboxing)](https://www.anthropic.com/engineering/claude-code-sandboxing) - OS-level sandboxing, 84% reduction in permission prompts
5. [Claude Opus 4.5 System Card (PDF)](https://assets.anthropic.com/m/64823ba7485345a7/Claude-Opus-4-5-System-Card.pdf) - 4.7% ASR single attempt, 0% in coding

### Security Research
6. [Unit 42: New Prompt Injection Attack Vectors Through MCP Sampling](https://unit42.paloaltonetworks.com/model-context-protocol-attack-vectors/) - Three MCP attack categories
7. [Invariant Labs: MCP Tool Poisoning Attacks](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks) - Shadowing technique, cross-server exfiltration
8. [Docker: MCP Horror Stories - GitHub Prompt Injection Data Heist](https://www.docker.com/blog/mcp-horror-stories-github-prompt-injection/) - Cross-repository exfiltration via GitHub issues
9. [Cymulate: InversePrompt CVE-2025-54794 & CVE-2025-54795](https://cymulate.com/blog/cve-2025-547954-54795-claude-inverseprompt/) - Prompt inversion attack against Claude

### Academic Papers
10. [Liu et al., "Your AI, My Shell" (arXiv:2509.22040)](https://arxiv.org/html/2509.22040v1) - Red-teaming Cursor and Copilot with MITRE ATT&CK framework
11. [Maloyan & Namiot, "Prompt Injection Attacks on Agentic Coding Assistants" (arXiv:2601.17548)](https://arxiv.org/html/2601.17548v1) - SoK synthesizing 78 studies, platform vulnerability ratings

### Analysis & Coverage
12. [VentureBeat: Anthropic Published Prompt Injection Failure Rates](https://venturebeat.com/security/prompt-injection-measurable-security-metric-one-ai-developer-publishes-numbers) - Coverage of Opus 4.6 system card metrics
13. [Zvi Mowshowitz: Claude Opus 4.6 System Card Analysis](https://thezvi.substack.com/p/claude-opus-46-system-card-part-1) - Detailed breakdown of prompt injection tables
14. [The Decoder: Claude Opus 4.5 Resists Prompt Injections Better Than Rivals](https://the-decoder.com/claude-opus-4-5-resists-prompt-injections-better-than-rivals-but-still-falls-to-strong-attacks-alarmingly-often/) - Comparative analysis

---

## Update History

| Date | Change |
|------|--------|
| 2026-04-01 | Initial research report |
