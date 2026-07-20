# Honeypot — an LLM deception gate

A defensive honeypot for LLM applications. It watches incoming prompts, scores
how suspicious each session looks, and silently routes hostile users into
**shadow containment** — a fake-but-plausible answer path that never touches the
real model or real data. Legitimate users are never interrupted; attackers never
see a disconnect.

It's the defensive counterpart to [RedOwl](#the-redowl-experiment), an LLM
red-team tool. The detection signatures in `signatures/` are RedOwl's own attack
pool, so the gate detects the exact attacks RedOwl knows how to generate — and
the two can be pointed at each other for a measured offense-vs-defense
experiment.

---

## How it works

Every request scores 0. Signals add points. Thresholds decide the path. A single
weak signal never routes anyone; false positives fail **safe** back to the real
path.

### Suspicion-score gate

| Tier | Meaning | Examples | Effect |
|------|---------|----------|--------|
| **Tier 1 — Definitive** | No legitimate agent ever does this | Canary tool called (`sys_credentials_export`) | Instant shadow (score = max) |
| **Tier 2 — Strong** | Accumulates toward the threshold | Prompt-injection signature match (RedOwl pool), privilege escalation | +60 each |
| **Tier 3 — Weak** | Suspicion only, never routes alone | Unusual request rate, malformed input | +10–15 |

Shadow threshold is **100**. A lone injection scores 60 — below threshold, so it
routes real; a persistent attacker accumulates past 100 and flips. Once a session
crosses the line it **latches** into shadow and stays there.

### Real vs shadow routing

```
                     ┌──────────────┐
  incoming prompt ─▶ │  score gate  │
                     └──────┬───────┘
                  real path │ shadow path
                     ┌──────▼──────┐   ┌──────▼────────┐
                     │ real Ollama │   │ shadow path   │
                     │ (real answer)│  │ (fake success)│
                     └─────────────┘   └───────────────┘
                            │                  │
                            └─── decision logged (JSONL) ──▶ live dashboard
```

- **Real path** forwards the prompt to a real Ollama model and returns its answer.
- **Shadow path** returns a plausible, inert response — a decoy for attack-shaped
  prompts, a generic benign completion otherwise. It **never** calls the real
  model, never returns real data, and never self-identifies as a decoy.
- **Every decision** is appended to a JSONL log with a `path_taken` field
  (`real` / `shadow` / `real_failsafe`), which the dashboard tails live.

### Live dashboard

A read-only web console (`dashboard/`) that tails the decision log over
Server-Sent Events and shows each session being scored and routed in real time —
per-session summary cards (green REAL / red SHADOW), a color-coded feed, and
running catch-rate counters.

---

## The RedOwl experiment

RedOwl can't target a bare URL — it takes a config YAML and speaks OpenAI
`/chat/completions`. So the gateway exposes an **OpenAI-compatible adapter** at
`/v1/chat/completions`; RedOwl targets it as if it were a normal LLM endpoint,
and the gate sits invisibly in between. Three traffic sets were run through the
same gate instance (see `experiment/`):

### Results

| Metric | Value | Notes |
|--------|-------|-------|
| **Catch rate** (attacks → shadow) | **91%** (20/22) | every attack after a session's first |
| **False-positive rate** (clean → shadow) | **0%** (0/20) | no legit prompt ever shadowed |
| **Slip rate** (attacks → real) | **9%** (2/22) | *by design*, see below |
| Fail-safes triggered | **0** | gate never crashed or hung |

The 9% slip is not a miss. A single injection signature scores 60, below the
threshold of 100, so the **first** attack in an attacker session routes real; the
session then accumulates past 100 and every later attack is shadowed. This is the
deliberate trade-off that guarantees a legitimate user is never routed on a
single ambiguous signal.

**The story:** an LLM red-team tool and an LLM honeypot, pointed at each other.
The honeypot caught 91% of the attack traffic while passing 100% of clean traffic
— offense meets defense, measured with real numbers.

📸 `experiment/out/dashboard_live_traffic.jpg`

---

## Phase 3 — session-level containment

### 3a · Mid-session rerouting

A user who starts legitimate and turns hostile gets flipped to shadow
**permanently**, with no visible disconnect. Demonstrated over one session
(`experiment/run_session_reroute.py`):

| # | prompt | score | route |
|---|--------|-------|-------|
| 1–3 | clean | 0 | REAL (real answers) |
| 4 | injection | 60 | REAL (below threshold) |
| 5 | injection | 120 | **SHADOW (flip)** |
| 6–7 | clean | 120 | SHADOW (sticky) |

After the flip, even clean prompts hit shadow — and get a generic, plausible
answer (not the attack-flavored decoy), so a contained attacker who returns to
innocent questions notices nothing.

📸 `experiment/out/dashboard_session_reroute.jpg`

### 3b · Multi-user dashboard

Multiple concurrent sessions, each on its own score track. A two-session demo
(`experiment/run_multiuser_demo.py`) runs a legit user and an attacker
interleaved in real time:

```
Time  Session       Prompt                              Expected
─────────────────────────────────────────────────────────────────
 1    legit-user    "What is the capital of Japan?"      REAL
 2    attacker      "How does a car engine work?"        REAL
 3    legit-user    "Explain how rainbows form"          REAL
 4    attacker      [injection attack]                   REAL (score 60)
 5    legit-user    "What's the tallest mountain?"       REAL
 6    attacker      [second injection attack]            SHADOW (flip)
 7    legit-user    "Tell me about the solar system"     REAL
 8    attacker      "What is 2 + 2?"                     SHADOW (sticky)
 9    legit-user    "How do airplanes fly?"              REAL
10    attacker      "Name three oceans"                  SHADOW (sticky)
```

Net result per session:

```
attacker     real=2  shadow=3    (clean → injection@60 → injection@120 flip → sticky)
legit-user   real=5  shadow=0    (100% real throughout)
```

The dashboard shows both side by side — one session green throughout, the other
flipping red mid-conversation and staying red — with per-session summary cards
and color-coded feed rows.

📸 `experiment/out/dashboard_multiuser.jpg`

---

## Quick start

```bash
pip install -r requirements.txt          # gate + experiment: stdlib + pyyaml
pip install fastapi uvicorn              # dashboard only

# 1) Phase 1 demo — attack pool + benign prompts, prints catch/slip/false-positive
python main.py --demo --deception

# 2) Gate in front of Ollama (deception ON), logging where the dashboard tails
python gateway.py --serve --deception --port 9100 --log experiment/out/decisions.jsonl

# 3) Live dashboard (separate terminal)
cd dashboard && python server.py --log ../experiment/out/decisions.jsonl --port 9101
#   open http://127.0.0.1:9101

# 4) Run the experiment sets, then summarize
python experiment/replay_attacks.py --gate http://127.0.0.1:9100   # Set A: attacks
python experiment/run_clean.py       --gate http://127.0.0.1:9100  # Set B: clean
python experiment/run_multiuser_demo.py --gate http://127.0.0.1:9100
python experiment/summarize.py --log experiment/out/decisions.jsonl --by-session

# tests
python -m unittest discover tests -v
```

Deception is **off by default** — without `--deception` the gate scores and logs
every signal but routes everything to the real path. Omitting the flag is the
kill switch.

---

## Layout

```
signatures/   RedOwl attack YAMLs used as detection signatures (11)
gate/
  signals.py       canary detection + injection-signature matcher
  scoring.py       suspicion scores, threshold, evaluate()
  router.py        detect → score → route → respond → log
  shadow.py        false tool success + shadow LLM responses
  decision_log.py  JSONL writer (includes path_taken)
gateway.py    Phase 2: score gate in front of Ollama; /generate +
              OpenAI /v1/chat/completions adapter (RedOwl-facing)
main.py       Phase 1 CLI + demo harness
tests/        gateway routing, fail-safe, adapter, and reroute tests
dashboard/    live decision-log console (server.py, index.html)
experiment/   RedOwl-vs-gate: adapter config, run scripts, summarize.py,
  out/           decision logs + dashboard screenshots
logs/         decisions.jsonl (created at runtime)
```

## Safety boundaries

- Deception off by default; a broken detector fails safe to the real path
  (logged as `real_failsafe`).
- Shadow state is disposable and never reads or writes real state.
- Benign prompts fail safe to the real path; one weak signal never routes anyone.
- Bounded lab module against the RedOwl harness — not an always-on production
  filter.

---

<sub>This is a public showcase — documentation and measured results only; the
source is kept in a separate private repository.</sub>

**Repository:** [github.com/KeithSistrunk/honeypot](https://github.com/KeithSistrunk/honeypot)
