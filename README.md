<div align="center">

```
     _       _   _           _   _            _
    / \  ___| |_| |__   ___ | | | | ___  _ __(_)_______  _ __
   / _ \/ _ \ __| '_ \ / _ \| |_| |/ _ \| '__| |_  / _ \| '_ \
  / ___ \  __/ |_| | | |  __/|  _  | (_) | |  | |/ / (_) | | | |
 /_/   \_\___|\__|_| |_|\___||_| |_|\___/|_|  |_/___\___/|_| |_|
```

**Agentic AI Security & GRC Platform**

*Three autonomous agents. One unified platform. Built for enterprise GRC.*

---

![Python](https://img.shields.io/badge/Python-3.11%2B-3776AB?style=flat-square&logo=python&logoColor=white)
![Anthropic](https://img.shields.io/badge/Claude-Sonnet%204.5-00B4D8?style=flat-square)
![Pydantic](https://img.shields.io/badge/Pydantic-v2-E92063?style=flat-square)
![Async](https://img.shields.io/badge/Async-asyncio%20%2B%20anyio-4B8BBE?style=flat-square)
![Tests](https://img.shields.io/badge/Tests-pytest--asyncio-0A9EDC?style=flat-square)
![Typed](https://img.shields.io/badge/Typed-mypy%20strict-2ECC71?style=flat-square)

</div>

---

## What This Is

AetherHorizon is a **production-grade agentic AI security platform** that demonstrates how Claude's tool-use API enables autonomous, multi-step security and GRC workflows. Three specialised agents each run independent tool-use loops — making real decisions about what to investigate next, how to classify findings, and how to synthesise results into actionable intelligence.

This is **not** a wrapper around an LLM. It is a properly engineered agentic system:

- **Real tool-use loops** — Claude calls tools, receives results, reasons about them, calls more tools
- **Type-safe throughout** — Pydantic v2, mypy strict mode, no `Any` shortcuts
- **Async-first** — every I/O path is `async`, fleet-ready from day one
- **Privacy-preserving** — SecretsHunter sends *zero* real credential values to the API
- **Extensible** — swap simulated recon for real Shodan/Censys APIs with one env variable

---

## The Three Agents

### ◈ SurfaceMapper — Attack Surface Reconnaissance
*External passive recon. No credentials required.*

Claude autonomously drives a multi-stage reconnaissance campaign:
DNS enumeration → subdomain discovery (CT logs) → port scanning → web fingerprinting → cloud asset probing → risk scoring → threat brief synthesis.

The agent decides *which hosts to scan deeply* based on intermediate results — internet-facing subdomains get scanned; internal ones don't. This is the key distinction from a scripted approach.

**7 tools:** `dns_enumerate`, `discover_subdomains`, `scan_host`, `fingerprint_web`, `discover_cloud_assets`, `whois_lookup`, `score_asset`

---

### ◈ SecretsHunter — Credential Leak Detection
*Pattern matching + AI triage. Zero secrets in the API.*

A two-stage pipeline: local regex + entropy detection first, then Claude triages the *redacted* results. Covers 25+ secret types across AWS, GCP, Azure, GitHub, Stripe, Twilio, Vault, and more.

**Privacy architecture:** The pattern engine runs entirely locally. Claude receives `type`, `severity`, `entropy`, and a redacted partial value — never the real credential. Blast radius analysis and remediation steps are generated per finding.

**4 tools:** `scan_simulated_repo`, `scan_provided_content`, `get_secret_blast_radius`, `generate_ciso_brief_data`

---

### ◈ ComplianceEngine — GRC Assessment
*NIST CSF 2.0 × SOC 2 Type II. The centrepiece for GRC roles.*

Claude acts as a senior GRC analyst working through a complete compliance assessment:

1. Lists all controls for the requested frameworks
2. Assesses each critical control against the organisation's evidence
3. Maps security findings from SurfaceMapper and SecretsHunter to specific control violations
4. Scores overall compliance posture with maturity level
5. Generates a prioritised evidence collection workplan for the upcoming audit

**Frameworks:** NIST Cybersecurity Framework 2.0 · SOC 2 Type II (TSC 2022) · ISO 27001:2022 scaffold · PCI DSS v4.0 scaffold

**5 tools:** `list_framework_controls`, `assess_control`, `map_finding_to_controls`, `score_compliance_posture`, `generate_evidence_requirements`

---

## Architecture

```
aetherhorizon/
├── src/aetherhorizon/
│   ├── core/
│   │   ├── config.py          ← Pydantic Settings v2 — typed env management
│   │   ├── models.py          ← Base domain types: Finding, AgentResult, ScanMetadata
│   │   ├── agent.py           ← BaseAgent — tool-use loop, retry, streaming
│   │   └── console.py         ← Rich terminal UI — branded, consistent
│   │
│   ├── surfacemapper/
│   │   ├── models.py          ← DiscoveredAsset, SurfaceFinding, HostExposure
│   │   ├── tools.py           ← 7 recon tools (live + simulation)
│   │   ├── agent.py           ← SurfaceMapperAgent
│   │   ├── report.py          ← Terminal + JSON + HTML report generation
│   │   └── cli.py             ← ah-surface CLI (Typer)
│   │
│   ├── secretshunter/
│   │   ├── patterns.py        ← 25+ regex patterns, Shannon entropy, SecretMatch
│   │   └── agent.py           ← SecretsHunterAgent (privacy-preserving)
│   │
│   ├── compliance/
│   │   ├── frameworks.py      ← Machine-readable NIST CSF 2.0 + SOC 2 controls
│   │   └── agent.py           ← ComplianceEngineAgent
│   │
│   └── cli.py                 ← Unified platform CLI (aetherhorizon)
│
├── tests/
│   ├── conftest.py
│   ├── surfacemapper/test_tools.py
│   └── secretshunter/ (coming)
│
├── examples/
│   └── demo_scan.py
│
├── .github/workflows/ci.yml   ← Lint, type-check, test, build
├── pyproject.toml             ← hatchling build, ruff, mypy, pytest config
└── Makefile
```

### The Agent Loop

The `BaseAgent.run()` method implements the Anthropic multi-turn tool-use pattern:

```python
# Simplified from core/agent.py
while iteration < max_iterations:
    response = await self._call_claude(messages)  # tenacity retry

    for block in response.content:
        if block.type == "text":
            final_text = block.text               # Claude's reasoning / report

        elif block.type == "tool_use":
            result = await self._execute_tool(block.name, block.input)
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": serialise(result),
            })

    if no_tool_use:
        break  # Claude is done

    # Multi-turn: add assistant turn + tool results
    messages.append({"role": "assistant", "content": response.content})
    messages.append({"role": "user",      "content": tool_results})
```

Each agent subclass only needs to define `tools`, `system_prompt`, `build_initial_prompt()`, and `parse_results()`. The loop, retry logic, streaming, and serialisation are all in `BaseAgent`.

---

## Getting Started

### Prerequisites

- Python 3.11+
- An Anthropic API key ([console.anthropic.com](https://console.anthropic.com))

### Install

```bash
git clone https://github.com/YOUR_USERNAME/aetherhorizon.git
cd aetherhorizon

# Create venv (recommended)
python3 -m venv .venv && source .venv/bin/activate

# Install with dev dependencies
make dev

# Configure
cp .env.example .env
# Edit .env — add your ANTHROPIC_API_KEY
```

### Run an Agent

```bash
# Attack surface recon (simulation mode — no external API calls)
aetherhorizon surface acmecorp.io --simulate

# Credential leak detection
aetherhorizon secrets acmecorp

# GRC compliance assessment
aetherhorizon grc "AcmeCorp" --framework both --audit-type SOC2

# Full assessment (all three agents, findings flow between them)
aetherhorizon assess acmecorp.io --simulate
```

### Output Formats

```bash
# Terminal (rich, coloured — default)
aetherhorizon surface acmecorp.io --format terminal

# JSON (machine-readable — pipe to jq, ingest into SIEM)
aetherhorizon surface acmecorp.io --format json | jq '.findings[].severity'

# HTML (standalone report — email to client or PM to audit folder)
aetherhorizon surface acmecorp.io --format html
aetherhorizon surface acmecorp.io --format all   # all three
```

### Run Tests

```bash
make test

# With coverage report
make test-cov
```

---

## Key Design Decisions

### Why `BaseAgent` with an abstract interface?

The tool-use loop is identical across all agents — only the tools, system prompt, and result parsing differ. Centralising the loop in `BaseAgent` means:
- Retry logic (tenacity) is written once
- Message history construction is written once  
- New agents are ~100 lines, not ~400

### Why Pydantic v2 everywhere?

Every domain object (`DiscoveredAsset`, `Finding`, `Control`, `ScanMetadata`) is a Pydantic model:
- `model_dump(mode="json")` gives free JSON export with correct datetime serialisation
- `computed_field` handles derived properties (risk scores, counts) without separate logic
- `field_validator` enforces invariants at model construction time
- mypy strict mode catches type errors before runtime

### Why Shannon entropy for secret detection?

Regex alone has ~40% false positive rate for generic high-entropy patterns. The entropy gate (`shannon_entropy(value, B64_CHARSET) > 3.5`) filters out low-entropy strings like placeholder values (`"your-api-key-here"`) while keeping real secrets. The threshold is pattern-specific — PEM key headers don't need an entropy check; generic `api_key=` assignments do.

### Why never send real secrets to the API?

The SecretsHunter pattern engine runs entirely in the Python process. Claude receives:
```json
{
  "secret_type": "AWS_ACCESS_KEY",
  "severity": "CRITICAL",
  "redacted_value": "AKIA***EXAMPLE",
  "entropy": 3.87,
  "file": ".env.production",
  "line": 4
}
```
The real value `AKIAIOSFODNN7EXAMPLE` never leaves the local process. This is the architecture required for enterprise and regulated environments.

---

## Extending the Platform

### Add a New Agent

```python
# src/aetherhorizon/myagent/agent.py
from aetherhorizon.core.agent import BaseAgent, ToolDefinition
from aetherhorizon.core.models import AgentResult, AgentStep, ScanMetadata

class MyAgent(BaseAgent):

    @property
    def tools(self) -> list[ToolDefinition]:
        return [MY_TOOL_1, MY_TOOL_2]

    @property
    def system_prompt(self) -> str:
        return "You are MyAgent..."

    def build_initial_prompt(self, target: str, **kwargs) -> str:
        return f"Analyse {target}..."

    async def parse_results(self, steps, final_text, metadata) -> AgentResult:
        findings = []  # extract from tool outputs in steps
        return AgentResult(metadata=metadata, steps=steps, findings=findings, raw_report=final_text)
```

That's the entire contract. The loop, retry, console hooks, and serialisation come for free.

### Enable Live Recon (Shodan)

```bash
# .env
AETHERHORIZON_SIMULATION_MODE=false
SHODAN_API_KEY=your-shodan-api-key
```

Tools already have live implementations (`_live_dns_enumerate`, `_shodan_host_lookup`, `_ct_log_search`) that activate when `simulation_mode=False`.

### Integrate Real Vulnerability Scanners

```python
# src/aetherhorizon/surfacemapper/tools.py
async def scan_host(host: str, top_ports: int = 20) -> dict[str, Any]:
    if not settings.simulation_mode and settings.has_shodan:
        return await _shodan_host_lookup(host)
    # Swap for Tenable, Qualys, Rapid7:
    # return await _tenable_asset_lookup(host)
    return _simulate_host_scan(host, top_ports)
```

### Add a Framework to ComplianceEngine

```python
# src/aetherhorizon/compliance/frameworks.py
HIPAA_CONTROLS: list[Control] = [
    Control(
        id="§164.312(a)(1)", framework=FrameworkID.HIPAA,
        title="Access Control — Unique User Identification",
        description="Assign a unique name and/or number for identifying and tracking user identity.",
        category="PROTECT", subcategory="Technical Safeguards",
        evidence_types=("IAM audit", "User account inventory", "MFA enrollment"),
        keywords=("access control", "user identification", "authentication", "mfa"),
        weight=5,
    ),
    # ... more controls
]
```

---

## GRC Capability Map

This platform directly supports the following GRC activities:

| Activity | Agent | Output |
|---|---|---|
| External attack surface baseline | SurfaceMapper | Asset inventory, risk scores |
| Credential exposure assessment | SecretsHunter | Findings with blast radius, SLA |
| NIST CSF 2.0 readiness assessment | ComplianceEngine | Posture score, control gap report |
| SOC 2 Type II pre-audit assessment | ComplianceEngine | Control status, evidence requirements |
| Security finding → control mapping | ComplianceEngine | Finding-to-control matrix |
| Evidence collection workplan | ComplianceEngine | Prioritised audit prep checklist |
| CISO / board executive brief | All agents | Natural language narrative |
| Continuous compliance monitoring | All agents | Scheduled runs via CI/CD |

---

## NIST CSF 2.0 Coverage

| Function | Controls Implemented | Weight-Adjusted Coverage |
|---|:---:|:---:|
| GOVERN (GV) — *New in CSF 2.0* | 3 | GV.OC, GV.RM, GV.SC |
| IDENTIFY (ID) | 4 | AM-01/02/05, RA-01/05 |
| PROTECT (PR) | 6 | AA-01/05, DS-01/02/10, PS-01/03 |
| DETECT (DE) | 2 | CM-01, CM-09 |
| RESPOND (RS) | 1 | MA-01 |
| RECOVER (RC) | 1 | RP-01 |

---

## Roadmap

**Phase 1 — Foundation** ✅
- [x] BaseAgent with real Anthropic tool-use loop
- [x] SurfaceMapper (7 tools, live + simulation)
- [x] SecretsHunter (25+ patterns, privacy-preserving)
- [x] ComplianceEngine (NIST CSF 2.0 + SOC 2 Type II)
- [x] Unified platform CLI
- [x] JSON + HTML report export
- [x] CI/CD pipeline (GitHub Actions)

**Phase 2 — GRC Depth** 🚧
- [ ] ISO 27001:2022 full control mapping
- [ ] PCI DSS v4.0 full control mapping
- [ ] HIPAA Technical Safeguards module
- [ ] Evidence management system (store, version, track expiry)
- [ ] Continuous compliance scheduler (cron-based rescanning)

**Phase 3 — Enterprise Integration** 📋
- [ ] Jira/ServiceNow ticket creation from findings
- [ ] Slack notifications with severity routing
- [ ] SIEM export (Splunk HEC, Elastic, Sentinel)
- [ ] Shodan + Censys live recon
- [ ] Tenable / Qualys vulnerability feed ingestion
- [ ] Multi-tenant organisation support

---

## Security Notice

- **No credentials** in source code or committed files
- **Simulation mode** is the default — no external API calls without explicit opt-in
- **Secret values** are never transmitted to the Anthropic API (redacted at pattern-match time)
- **`.env` is gitignored** — only `.env.example` with placeholders is committed
- Run `bandit -r src/` for SAST scanning (included in CI)

---

<div align="center">

Built by **AetherHorizon Security** · Python 3.11+ · Claude Sonnet 4.5 · Pydantic v2

*Agentic AI for the engineers who run GRC programmes that actually work.*

</div>
