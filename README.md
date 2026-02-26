# piqrypt-session

**Multi-agent cryptographic session for PiQrypt.**

[![PyPI](https://img.shields.io/pypi/v/piqrypt-langchain)](https://pypi.org/project/piqrypt-langchain/)
[![Downloads](https://img.shields.io/pypi/dm/piqrypt-langchain)](https://pypi.org/project/piqrypt-langchain/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![PiQrypt](https://img.shields.io/badge/powered%20by-PiQrypt-blue)](https://github.com/piqrypt/piqrypt)

Establishes co-signed A2A handshakes between all agents before any action takes place. Each agent keeps its own verifiable memory. Every interaction references a shared session — provable, tamper-proof, non-repudiable.

```bash
pip install piqrypt-session
```

Works with any framework — LangChain, AutoGen, CrewAI, OpenClaw, or plain Python.

---

## The idea

When multiple AI agents collaborate, today's systems record what each agent did — but not that they **agreed to work together**, and not that one agent's action was **seen and acknowledged** by another.

`piqrypt-session` fills that gap:

```
Before any action:
  LLM ↔ TradingBot    → co-signed handshake ✅
  LLM ↔ OpenClaw      → co-signed handshake ✅
  TradingBot ↔ OpenClaw → co-signed handshake ✅

During the session:
  LLM recommends BUY AAPL
    → LLM memory:         "I sent this recommendation to TradingBot"
    → TradingBot memory:  "I received this recommendation from LLM"
    → Both events share the same interaction_hash
    → Neither can deny it happened
```

---

## Quickstart

```python
from piqrypt_session import AgentSession

# Define your agents — any framework, any setup
session = AgentSession([
    {"name": "llm",          "identity_file": "llm.json"},
    {"name": "trading_bot",  "identity_file": "trading-bot.json"},
    {"name": "openclaw",     "identity_file": "openclaw.json"},
])

# Start — performs all pairwise handshakes automatically
session.start()
# [PiQrypt Session] ✅ Session started: sess_a3f9b2c1d4e5f6a7
#   Agents    : llm, trading_bot, openclaw
#   Handshakes: 3 co-signed
#   Timestamp : 1740477600

# Stamp a co-signed interaction between two agents
session.stamp("llm", "recommendation_sent", {
    "symbol": "AAPL",
    "action": "buy",
    "confidence": 0.87,
    "reasoning": "Strong Q4 earnings, positive momentum",
}, peer="trading_bot")
# → event in LLM memory:         "I sent recommendation to TradingBot"
# → event in TradingBot memory:  "I received recommendation from LLM"
# → both share interaction_hash

# Stamp a unilateral action
session.stamp("trading_bot", "order_executed", {
    "symbol": "AAPL",
    "action": "buy",
    "price": 182.50,
    "quantity": 100,
    "order_id": "ORD-20260225-001",
})

# End session
session.end()

# Export full audit — all agents, all events, all handshakes
session.export("trading-session-audit.json")
# $ piqrypt verify trading-session-audit.json
```

---

## What each agent's memory contains

**LLM memory** (`~/.piqrypt/llm/events/`):
```json
{ "event_type": "session_start",         "session_id": "sess_a3f9..." }
{ "event_type": "a2a_handshake",         "peer_agent_id": "trading_bot_id", "peer_signature": "..." }
{ "event_type": "a2a_handshake",         "peer_agent_id": "openclaw_id",    "peer_signature": "..." }
{ "event_type": "recommendation_sent",   "interaction_hash": "c7d2...", "peer_agent_id": "trading_bot_id" }
{ "event_type": "session_end",           "session_id": "sess_a3f9..." }
```

**TradingBot memory** (`~/.piqrypt/trading_bot/events/`):
```json
{ "event_type": "session_start",                    "session_id": "sess_a3f9..." }
{ "event_type": "a2a_handshake",                    "peer_agent_id": "llm_id", "peer_signature": "..." }
{ "event_type": "recommendation_sent_received",     "interaction_hash": "c7d2...", "peer_agent_id": "llm_id" }
{ "event_type": "order_executed",                   "session_id": "sess_a3f9..." }
{ "event_type": "session_end",                      "session_id": "sess_a3f9..." }
```

**What links them:**
- Same `session_id` across all memories
- Same `interaction_hash` in co-signed events
- Each memory has the **peer's signature embedded**

---

## Framework-agnostic

`piqrypt-session` works independently of any agent framework.
Combine with framework bridges for full coverage:

```python
# LangChain agent + AutoGen agent in the same session
from piqrypt_langchain import PiQryptCallbackHandler
from piqrypt_autogen import AuditedAssistant
from piqrypt_session import AgentSession

# Each framework stamps its own actions
lc_handler = PiQryptCallbackHandler(identity_file="langchain-agent.json")
ag_assistant = AuditedAssistant(name="analyst", llm_config=llm_config,
                                 identity_file="autogen-agent.json")

# Session links them together
session = AgentSession([
    {"name": "langchain_agent", "identity_file": "langchain-agent.json"},
    {"name": "autogen_agent",   "identity_file": "autogen-agent.json"},
])
session.start()

# All individual actions are stamped by the bridges
# All interactions between agents are co-signed by the session
session.stamp("langchain_agent", "analysis_sent", {
    "result": analysis_output
}, peer="autogen_agent")
```

---

## N agents — automatic handshakes

| Agents | Handshakes |
|---|---|
| 2 | 1 |
| 3 | 3 |
| 4 | 6 |
| 5 | 10 |
| N | N*(N-1)/2 |

All handshakes happen automatically on `session.start()`.

---

## Verify

```bash
piqrypt verify trading-session-audit.json
# ✅ Session sess_a3f9b2c1d4e5f6a7
#    Agents    : 3
#    Handshakes: 3 co-signed
#    Events    : 14 total
#    Integrity : verified
#    Forks     : 0
```

---

## Scope

| Use case | Profile |
|---|---|
| Development / PoC | AISS-1 (Free, included) |
| Non-critical production | AISS-1 (Free) |
| Regulated production | AISS-2 (Pro) |

---

## Links

- **PiQrypt core:** [github.com/piqrypt/piqrypt](https://github.com/piqrypt/piqrypt)
- **LangChain bridge:** [piqrypt-langchain-integration](https://github.com/piqrypt/piqrypt-langchain-integration)
- **AutoGen bridge:** [piqrypt-autogen-integration](https://github.com/piqrypt/piqrypt-autogen-integration)
- **CrewAI bridge:** [piqrypt-crewai-integration](https://github.com/piqrypt/piqrypt-crewai-integration)
- **OpenClaw bridge:** [piqrypt-openclaw](https://github.com/piqrypt/piqrypt-openclaw-integration)
- **Issues:** piqrypt@gmail.com

---

*PiQrypt — Verifiable AI Agent Memory*  

