<p align="center">
  <a href="https://github.com/occludra/gateway">
    <img alt="Occludra — AI Security Gateway" src="https://occludra.ai/og.png" width="600" />
  </a>
</p>

<h3 align="center">The Open-Source AI Firewall & LLM Proxy</h3>

<p align="center">
  Drop-in AI security proxy with DLP, SSO, RBAC, SIEM, and Hybrid VPC. Redacts PII, blocks prompt injection, enforces governance — before prompts reach any LLM.<br />
  <strong>OpenAI SDK compatible. Enterprise-ready. Two lines of code.</strong>
</p>

<p align="center">
  <a href="https://github.com/occludra/gateway"><strong>Get Started</strong></a> ·
  <a href="https://occludra.ai/docs"><strong>Docs</strong></a> ·
  <a href="https://occludra.ai/open-source"><strong>OSS vs Cloud</strong></a> ·
  <a href="https://occludra.ai/docs/hybrid-vpc-deployment"><strong>Hybrid VPC</strong></a> ·
  <a href="https://occludra.ai"><strong>Managed Cloud (1M free credits)</strong></a>
</p>

<p align="center">
  <a href="https://github.com/occludra/gateway/blob/main/LICENSE"><img src="https://img.shields.io/badge/License-Apache_2.0-blue?style=for-the-badge" alt="Apache 2.0" /></a>&nbsp;
  <a href="https://github.com/occludra/gateway"><img src="https://img.shields.io/badge/Docker-Quickstart-2496ED?style=for-the-badge&logo=docker&logoColor=white" alt="Docker" /></a>&nbsp;
  <a href="https://occludra.ai/docs/openai-compatible-proxy"><img src="https://img.shields.io/badge/OpenAI_SDK-Compatible-10a37f?style=for-the-badge&logo=openai&logoColor=white" alt="OpenAI Compatible" /></a>
</p>

---

Every LLM application we audited had the same problem: sensitive data flowing directly from user prompts to third-party AI providers, unfiltered.

Occludra (AI Security Gateway) is the **control layer** that sits between your application and any LLM provider — scanning every request for PII, secrets, and prompt injection attacks before anything reaches the model.

```
    ┌──────────┐           ┌─────────────────────────────┐           ┌──────────────┐
    │          │  POST     │        Occludra Gateway      │           │              │
    │ Your App ├──────────▸│  1. Auth (API key)           │──────────▸│ LLM Provider │
    │          │           │  2. DLP scan (Presidio)      │           │ (8 supported) │
    │          │◂──────────│  3. Block or redact PII      │◂──────────│              │
    └──────────┘  response │  4. Forward to upstream      │  response └──────────────┘
                           │  5. Return with metadata     │
                           └─────────────────────────────┘
```

---

## What It Does

- **PII Redaction** — 13 entity types out of the box: emails, phone numbers, credit cards, SSNs, names, locations, IP addresses, and more
- **Secret Detection** — API keys (OpenAI, Anthropic, Google, AWS), GitHub tokens, private keys, Slack webhooks
- **Prompt Injection Blocking** — jailbreaks, DAN variants, instruction overrides, system prompt extraction, developer mode exploits
- **OpenAI SDK Compatible** — drop-in replacement, change one line of code
- **Multi-Provider Routing** — 8 providers: OpenAI, Anthropic, Groq, Together, Gemini, Mistral, DeepInfra, xAI — BYOK, swap in config
- **Fail-Closed Security** — if the safety layer is down, requests are **blocked**, never forwarded unscanned
- **Zero Cloud Dependencies** — runs entirely on your infrastructure via Docker
- **No Telemetry** — zero external calls, no analytics, no phone-home
- **SAML SSO** — Okta, Azure AD, Google Workspace, any SAML 2.0 IdP — auto-provisioning + enforced SSO (cloud)
- **RBAC** — 4-tier role hierarchy (Owner / Admin / Member / Viewer) with 17 granular permissions (cloud)
- **SIEM Connectors** — stream security events to Splunk HEC, Datadog Logs, or Microsoft Sentinel in real-time (cloud)
- **Hybrid VPC** — compiled Go proxy in your VPC, prompts never leave your network. Cloud dashboard for policies (cloud)

---

## Quickstart (60 seconds)

```bash
git clone https://github.com/occludra/gateway.git
cd gateway
cp .env.example .env        # add your provider key
docker compose up --build   # gateway + presidio
```

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Authorization: Bearer change-me-to-a-real-secret" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-4-maverick",
    "messages": [{"role": "user", "content": "My email is alice@acme.com and SSN is 123-45-6789"}]
  }'
```

The gateway redacts the email and SSN before forwarding. The response includes `aisg_metadata.pii_detected: true`.

### Python SDK

```bash
pip install aisg
```

```python
from aisg import AISG

client = AISG(api_key="your-gateway-key", base_url="http://localhost:8000/v1")

response = client.chat.create(
    model="llama-4-maverick",
    messages=[{"role": "user", "content": "My email is alice@acme.com"}],
)
print(response.aisg_metadata.pii_detected)        # True
print(response.aisg_metadata.entity_types_detected) # ["EMAIL_ADDRESS"]
```

Typed responses, structured errors (`DLPBlockError`, `BudgetExhaustedError`, `LoopDetectedError`), async support, and model discovery. Works with self-hosted and [managed cloud](https://occludra.ai). → [Full SDK docs](https://github.com/occludra/gateway/tree/main/sdk/python)

---

## What Gets Detected

| PII (Presidio built-ins) | Developer Secrets (custom) | Prompt Injection |
|---|---|---|
| `EMAIL_ADDRESS` | `API_KEY` (OpenAI, Anthropic, GCP) | Ignore previous instructions |
| `PHONE_NUMBER` | `AWS_ACCESS_KEY` | Disregard your rules |
| `CREDIT_CARD` | `PRIVATE_KEY` (RSA, EC, etc.) | System prompt extraction |
| `US_SSN` | `GITHUB_TOKEN` (PAT, OAuth) | DAN / jailbreak attempts |
| `PERSON`, `LOCATION` | `SLACK_WEBHOOK` | Developer mode exploits |
| `IP_ADDRESS` | | SYSTEM OVERRIDE impersonation |

**13 entity types** self-hosted — the [managed cloud](https://occludra.ai) extends this to **30+ entity types** with OCR image scanning, street addresses, crypto addresses, medical identifiers, and more.

---

## Security Model

- **Fail-closed by default** — if Presidio is unreachable, requests are **blocked**, never forwarded unscanned
- **Auth by default** — API key authentication enabled out of the box
- **No telemetry** — zero external calls, no analytics, no phone-home
- **Secret scrubbing** — structured logs automatically mask API keys and tokens
- **Rate limiting** — token bucket per API key (default 10 req/sec)

Designed for teams building **GDPR**, **HIPAA**, and **SOC 2**-compliant AI applications. Prompts are never stored.

---

## OSS vs Managed Cloud

This repo gives you the core AI security proxy. The managed [Occludra Cloud](https://occludra.ai) adds everything you need to run it across teams at scale.

| | OSS (this repo) | [Cloud](https://occludra.ai) |
|---|:---:|:---:|
| PII detection & redaction (text) | 13 entity types | 30+ entity types |
| Providers | 8 (OpenAI, Anthropic, Groq, Together, Gemini, Mistral, DeepInfra, xAI) | 8+ with managed keys |
| OCR image scanning | — | Yes |
| Secret leak prevention | 5 recognizers | Extended (incl. Groq, AWS Secret Key, crypto, MAC) |
| Prompt injection blocking | 5 core patterns | Extended library: mid-context override, system-prompt extraction, refusal suppression, SYSTEM OVERRIDE — FP-tuned |
| Routing | Header-based (`x-provider`) | Smart Router + real-time pricing |
| Failover | — | Automatic intelligent chains |
| Cost optimization | — | Automatic (cheapest per request) |
| Budget enforcement | — | Per-project caps + alerts + analytics |
| Model discovery API | — | `GET /v1/models` with 600+ models |
| Self-hosted | Yes | Managed |
| Multi-project management | — | Yes |
| Project-level DLP policies | — | Yes |
| Dashboards, leak reports & analytics | — | Yes |
| Real-time model pricing registry | — | Yes |
| Managed provider keys (no BYOK required) | — | Yes |
| SAML SSO (Okta, Azure AD, Google Workspace) | — | Yes |
| RBAC (Owner/Admin/Member/Viewer, 17 permissions) | — | Yes |
| SIEM connectors (Splunk, Datadog, Sentinel) | — | Yes |
| Hybrid VPC (prompts stay in your network) | — | Yes |
| MCP Gateway (policy enforcement on the agent tool plane) | — | Enterprise — private beta |
| Semantic caching (DLP-aware) | — | Yes |
| Recursive loop protection (agent retry kill) | — | Yes |
| Webhook security alerts (HMAC-signed) | — | Yes |
| EU AI Act compliance logging (hash-chained) | — | Yes |
| SLA & support | Community | Yes |

**Why are loop protection, EU AI Act logging, and semantic caching cloud-only?** These aren't artificial paywalls — each genuinely requires distributed infrastructure:

- **Loop protection** needs shared state (Redis) across horizontally-scaled proxy instances to detect cross-instance agent loops. A single-instance approximation would miss the exact failure mode you'd want to catch.
- **EU AI Act logging** needs managed WORM storage with tamper-evident hash chains, retention policies, and secure export — the operational burden regulated teams pay to avoid.
- **Semantic caching** needs a distributed cache backend with TTL management and cross-instance coherence. A local cache only helps one instance.

The OSS is a complete, production-ready security proxy. If a feature doesn't require distributed infrastructure, it belongs in the OSS. The 8-provider expansion (from 2 at launch) is one example of this commitment.

**Enterprise features** (SAML SSO, RBAC, SIEM connectors) require centralized identity management, organization-level metadata, and persistent event streaming — inherently multi-tenant services. **Hybrid VPC** bridges the gap: prompts stay in your network while the cloud manages policies, SSO, and analytics via metadata-only telemetry.

**MCP Gateway** (Enterprise, *private beta*) extends the same policy engine to the agent **tool plane** — the surface prompt-only gateways miss. It scans MCP tool descriptions for poisoning at catalog time, DLP-scans tool-call **arguments** (block) and tool **results** (redact/block), and enforces default-deny tool allowlists. Runs in both managed Cloud and Hybrid VPC.

> **Skip the setup?** [occludra.ai](https://occludra.ai) — everything here plus SSO, RBAC, SIEM connectors, Hybrid VPC, dashboards, smart cost routing, and 600+ models. 1M free credits, no credit card.

---

<p align="center">
  <a href="https://theresanaiforthat.com/ai/aisecuritygateway/?ref=featured&v=7352275" rel="nofollow noopener noreferrer">
    <img width="200" src="https://media.theresanaiforthat.com/featured-on-taaft.png?width=600" alt="Featured on There's An AI For That" />
  </a>
</p>

---

<p align="center">
  <a href="https://github.com/occludra/gateway"><strong>⭐ Star the repo</strong></a> ·
  <a href="https://occludra.ai/open-source"><strong>Learn more</strong></a> ·
  <a href="https://occludra.ai"><strong>Try the managed cloud free</strong></a>
</p>

<p align="center">
  <sub>
    <a href="https://occludra.ai/security">Security</a> ·
    <a href="https://github.com/occludra/gateway/blob/main/LICENSE">License (Apache 2.0)</a> ·
    <a href="https://www.crunchbase.com/organization/ai-security-gateway">Crunchbase</a> ·
    <a href="https://linkedin.com/company/ai-security-gateway">LinkedIn</a> ·
    <a href="https://x.com/occludra">X / Twitter</a> ·
    <a href="https://www.youtube.com/@AISecurityGateway">YouTube</a>
  </sub>
</p>

<p align="center">
  <sub>Built by <a href="https://datumfuse.com">Datum Fuse LLC</a> — making AI safe by default.</sub>
</p>
