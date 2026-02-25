# Verifiable AI Agent Memory

**What if every AI agent could prove what it did, when it did it, and with whom?**

Not with editable logs. Not with screenshots. With cryptographic proof — signed, chained, tamper-proof. Forever.

This is what PiQrypt builds. For every agent. On every framework.

---

## The problem with AI agent memory today

When an AI agent acts, it leaves a trace. But that trace lives in a log file. And log files can be edited. Deleted. Fabricated.

```
Today's reality
─────────────────────────────────────────────────────

  Agent decides         Log file records it
  ──────────────        ────────────────────
  "BUY AAPL $182"  →   app.log: "trade executed"
                              ↑
                         anyone can edit this
                         anyone can delete this
                         no proof it happened
                         no proof WHO decided
                         no proof WHEN exactly
```

This is fine for debugging. It is not fine when:
- A trade goes wrong and regulators ask for proof
- An automated decision is challenged in court
- Two AI agents disagree about what was agreed
- A model update changes behavior and nobody can prove it

---

## The PiQrypt answer — Verifiable Memory

Every action becomes a **signed, chained, tamper-proof memory entry**.

```
PiQrypt memory entry
─────────────────────────────────────────────────────

  Agent decides         PiQrypt records it
  ──────────────        ────────────────────────────────────────
  "BUY AAPL $182"  →   {
                          agent_id:   "5Z8nY7KpL9mN..."  ← who
                          timestamp:  1740477600          ← when (RFC 3161)
                          payload:    hash("BUY AAPL")    ← what (never raw)
                          prev_hash:  "a3f9b2c1..."       ← chain continuity
                          signature:  Ed25519 sig         ← unforgeable proof
                        }
                              ↑
                         cannot be edited (breaks signature)
                         cannot be deleted (breaks chain)
                         cannot be fabricated (no private key)
                         cannot be backdated (RFC 3161 timestamp)
```

---

## Memory architecture — three layers

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│   Layer 3 — Session Memory                         │
│   Multi-agent · Co-signed · Shared session_id      │
│   "We agreed to work together. Here is the proof." │
│                                                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│   Layer 2 — Interaction Memory                     │
│   A2A handshake · Peer signature embedded          │
│   "I talked to Agent B. Agent B confirms."        │
│                                                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│   Layer 1 — Individual Memory                      │
│   Single agent · Signed · Hash-chained             │
│   "I did this. Here is my proof."                  │
│                                                     │
└─────────────────────────────────────────────────────┘
```

You can use any layer independently. They compose naturally.

---

## Layer 1 — Individual Memory

One agent. Every action signed and chained.

```
Agent lifecycle
──────────────────────────────────────────────────────────────

  identity created    first action       second action
  ────────────────    ────────────────   ────────────────
  [genesis event] ──▶ [event 1]       ──▶ [event 2]      ──▶ ...
        │                  │                   │
        ▼                  ▼                   ▼
   Ed25519 sig        Ed25519 sig         Ed25519 sig
   hash: "genesis"    hash: H(event0)     hash: H(event1)
                           ↑                   ↑
                      chain link          chain link
                    (modifying event1    (modifying event2
                     breaks event2)       breaks event3)
```

**What this proves:**
- The agent with this identity performed this action
- At this exact timestamp
- In this exact sequence
- Nothing was inserted, deleted, or modified

```python
import piqrypt as aiss

# One identity per agent — generate once, persist forever
private_key, public_key = aiss.generate_keypair()
agent_id = aiss.derive_agent_id(public_key)

# Stamp every significant action
event = aiss.stamp_event(private_key, agent_id, {
    "action": "trade_executed",
    "symbol": "AAPL",
    "quantity": 100,
})
aiss.store_event(event)

# Verify anytime
aiss.verify_chain(aiss.load_events())  # ✅ or raises
```

---

## Layer 2 — Interaction Memory (A2A)

Two agents. Co-signed proof of their interaction.

```
Agent A and Agent B interact
──────────────────────────────────────────────────────────────

  Agent A memory                  Agent B memory
  ─────────────────────           ─────────────────────
  {                               {
    event_type: "a2a_handshake"     event_type: "a2a_handshake"
    my_role: "initiator"            my_role: "responder"
    peer_id: B.agent_id             peer_id: A.agent_id
    peer_signature: B.sig    ←→     peer_signature: A.sig
    session_id: "sess_abc"          session_id: "sess_abc"
    signature: A.sig                signature: B.sig
  }                               }
       ↑                                ↑
  signed by A                     signed by B
  contains B's sig                contains A's sig
  stored in A's memory            stored in B's memory
```

**What this proves:**
- A and B mutually identified each other
- Neither can deny the interaction happened
- Both have independent, verifiable records
- The records corroborate each other

```python
from aiss.a2a import create_identity_proposal, perform_handshake

# Agent A proposes
proposal = create_identity_proposal(a_key, a_pub, a_id)

# Agent B responds — produces co-signed event in both chains
result = perform_handshake(b_key, b_pub, b_id, proposal)

# result["cosigned_event"] is now in B's memory
# A builds its own co-signed event from proposal + result["response"]
```

---

## Layer 3 — Session Memory (Multi-Agent)

N agents. One shared session. All pairs co-signed before any action.

```
Session with 3 agents: LLM · TradingBot · OpenClaw
──────────────────────────────────────────────────────────────

  session.start()
        │
        ├── LLM ↔ TradingBot      co-signed handshake ✅
        ├── LLM ↔ OpenClaw        co-signed handshake ✅
        └── TradingBot ↔ OpenClaw co-signed handshake ✅
              ↑
        3 agents = 3 handshakes
        N agents = N*(N-1)/2 handshakes
        all automatic, all co-signed


  Session running
  ───────────────

  LLM memory              TradingBot memory       OpenClaw memory
  ────────────────────    ────────────────────    ────────────────────
  session_start           session_start           session_start
  handshake↔TradingBot    handshake↔LLM           handshake↔LLM
  handshake↔OpenClaw      handshake↔OpenClaw      handshake↔TradingBot
  recommendation_sent ──▶ recommendation_rcvd
                          order_executed                          ──▶ report_generated
  session_end             session_end             session_end
        │                       │                       │
        └───────────────────────┴───────────────────────┘
                    all share session_id: "sess_a3f9..."
```

**What this proves:**
- All agents agreed to collaborate before acting
- Every interaction is recorded from both perspectives
- The same `interaction_hash` links both sides of an interaction
- No agent can claim ignorance of what another did

```python
from piqrypt_session import AgentSession

session = AgentSession([
    {"name": "llm",          "identity_file": "llm.json"},
    {"name": "trading_bot",  "identity_file": "trading-bot.json"},
    {"name": "openclaw",     "identity_file": "openclaw.json"},
])

session.start()
# ✅ 3 co-signed handshakes — all agents mutually identified

# Co-signed interaction — recorded in both memories
session.stamp("llm", "recommendation_sent", {
    "symbol": "AAPL",
    "action": "buy",
    "confidence": 0.87,
}, peer="trading_bot")

# Unilateral action — recorded in one memory
session.stamp("trading_bot", "order_executed", {
    "symbol": "AAPL",
    "price": 182.50,
    "quantity": 100,
})

session.end()
session.export("audit.json")
```

---

## What each memory contains — the full picture

```
A real session: LLM recommends, TradingBot executes, OpenClaw reports
──────────────────────────────────────────────────────────────────────────

LLM memory (~/.piqrypt/llm/events/2026-02.json)
────────────────────────────────────────────────
  ① session_start          session_id: sess_a3f9
  ② a2a_handshake          peer: trading_bot  peer_sig: ████
  ③ a2a_handshake          peer: openclaw     peer_sig: ████
  ④ recommendation_sent    interaction_hash: c7d2  peer: trading_bot
  ⑤ session_end            duration: 47s


TradingBot memory (~/.piqrypt/trading_bot/events/2026-02.json)
────────────────────────────────────────────────────────────────
  ① session_start              session_id: sess_a3f9
  ② a2a_handshake              peer: llm          peer_sig: ████
  ③ a2a_handshake              peer: openclaw     peer_sig: ████
  ④ recommendation_rcvd        interaction_hash: c7d2  peer: llm
     peer_sig: ████            ← LLM's signature embedded here
  ⑤ order_executed             symbol: AAPL  price_hash: ████
  ⑥ session_end                duration: 47s


OpenClaw memory (~/.piqrypt/openclaw/events/2026-02.json)
──────────────────────────────────────────────────────────
  ① session_start          session_id: sess_a3f9
  ② a2a_handshake          peer: llm          peer_sig: ████
  ③ a2a_handshake          peer: trading_bot  peer_sig: ████
  ④ task_reasoning         task_hash: ████    model: llama-3.2
  ⑤ tool_execution         tool: bash         input_hash: ████
  ⑥ tool_execution         tool: file_write   output_hash: ████
  ⑦ session_end            duration: 47s


What links them all
────────────────────
  session_id: sess_a3f9               → same in all memories
  interaction_hash: c7d2              → same in LLM④ and TradingBot④
  peer_sig in TradingBot④             → is LLM's signature from LLM④
  all events hash-chained             → modifying any breaks the chain
```

**An auditor can:**
1. Load all three memory files
2. Verify each chain independently
3. Cross-reference interaction hashes
4. Confirm peer signatures match
5. Prove the complete session cryptographically

No content was ever stored — only hashes.

---

## Framework support

PiQrypt works with every major AI agent framework.

```
Your framework          Install                         What gets stamped
────────────────────    ──────────────────────────      ──────────────────────────────
LangChain               pip install piqrypt-langchain   LLM calls · tool calls · chains
AutoGen (Microsoft)     pip install piqrypt-autogen     replies · code execution · group chat
CrewAI                  pip install piqrypt-crewai      tasks · crew kickoffs · decorators
OpenClaw                pip install piqrypt-openclaw    reasoning · bash · file ops
Multi-agent session     pip install piqrypt-session     handshakes · interactions · exports
Plain Python            pip install piqrypt             anything you want
```

All bridges are independent. All depend only on `piqrypt` core.  
Mix and match — a LangChain agent and an AutoGen agent can share a session.

---

## Memory properties

| Property | What it means |
|---|---|
| **Signed** | Each entry is Ed25519-signed by the agent's private key |
| **Chained** | Each entry references the hash of the previous one |
| **Tamper-evident** | Modifying any entry breaks all subsequent hashes |
| **Non-repudiable** | The agent cannot deny its actions — it signed them |
| **Co-signed** | Interactions between agents carry both signatures |
| **Local-first** | Memory lives on disk — no cloud, no network required |
| **Post-quantum** | Optional Dilithium3 (NIST FIPS 204) for 50-year proofs |
| **Privacy-preserving** | Raw content never stored — only SHA-256 hashes |

---

## Verify any memory

```bash
# Single agent
piqrypt verify agent-audit.json
# ✅ Chain integrity verified — 47 events, 0 forks

# Full session
piqrypt verify session-audit.json
# ✅ Session sess_a3f9b2c1d4e5f6a7
#    Agents    : 3
#    Handshakes: 3 co-signed
#    Events    : 89 total
#    Integrity : verified
#    Forks     : 0
```

---

## This is a first

No AI agent framework today provides:
- Cryptographic identity at the agent level
- Co-signed proof of agent-to-agent interactions
- A shared session context linking N agent memories
- Post-quantum readiness for long-term legal validity

**PiQrypt is the first implementation of Verifiable AI Agent Memory.**  
Built on [AISS v1.1](https://github.com/piqrypt/piqrypt) — Agent Identity & Signature Standard.

---

## Get started

```bash
# Core
pip install piqrypt

# With your framework
pip install piqrypt-langchain    # LangChain
pip install piqrypt-autogen     # AutoGen
pip install piqrypt-crewai      # CrewAI
pip install piqrypt-openclaw    # OpenClaw

# Multi-agent sessions
pip install piqrypt-session
```

→ **[Full documentation](https://github.com/piqrypt/piqrypt)**  
→ **[Quick Start](https://github.com/piqrypt/piqrypt/blob/main/QUICK-START.md)**  
→ **[Integration Guide](https://github.com/piqrypt/piqrypt/blob/main/INTEGRATION.md)**

---

*PiQrypt — Verifiable AI Agent Memory*  
*MIT License · Local-first · No cloud · No signup*
