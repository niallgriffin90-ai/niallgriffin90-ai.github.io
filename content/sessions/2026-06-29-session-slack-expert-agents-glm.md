---
title: "Session brief — Slack, expert agents & GLM"
date: 2026-06-29
tags:
  - hermes
  - slack
  - neutrality
  - glm
  - session-brief
project: hermes-agent
---

# Session brief — 29 June 2026

Covers connecting Hermes to Slack, the design of the first expert-trained domain
agent (neutrality studies, for Prof. Pascal Lottaz), switching the default model
to GLM 5.2, wiring Claude Code to GLM via Coding Helper, and a still-open GLM
authentication bug. Written to be re-read cold: each section states what was done,
why, and what it means.

---

## Starting point

Hermes was already running on the Hetzner VPS (91.99.141.162), reachable via
Telegram, with the dashboard accessible over an SSH tunnel, voice transcription
working, and the four-project MEMORY.md in place. Today's goals: add Slack as a
second front door, design the first expert-training agent, and move the default
model to GLM 5.2 for cost.

---

## Phase 1 — Slack connection

### What was set up

Hermes connected to a new Slack workspace ("AI Res" app) as a bot via Socket Mode,
running alongside Telegram from the same gateway, same memory, same projects.

### What Socket Mode is

Socket Mode connects the bot to Slack over a WebSocket instead of a public HTTP
endpoint. This means the Hetzner server doesn't need to expose any public URL —
the bot works from behind the firewall. One less attack surface.

### The manifest shortcut

Instead of manually configuring scopes, events, and slash commands (the
error-prone part), `hermes slack manifest --write` generated a manifest file that
declares all of them at once. This was pasted into api.slack.com/apps → Create New
App → From an app manifest. The manifest handled the four most-commonly-missed
setup steps automatically.

### Tokens obtained

- App-level token (`xapp-`) — from Socket Mode settings, with `connections:write` scope
- Bot token (`xoxb-`) — from Install App, after installing to workspace
- Member ID (`U...`) — for the allowlist, so only authorised users can use the bot

These went into the gateway via `hermes gateway setup` → Slack.

### The shared-session decision

`group_sessions_per_user: false` was set in config.yaml. This is the key choice
for the expert-training use case: the whole channel shares ONE conversation rather
than each user getting an isolated silo. When expert A teaches the agent and expert
B builds on it, they're in the same context.

**The tradeoff:** users share context growth and token cost, and one person's
`/reset` wipes the session for everyone. Managed by house rules rather than
isolation, because isolation would defeat the collaborative-training purpose.

### Result

Bot responds in Slack, knows Niall from USER.md, personality intact. Hermes now
reachable from Telegram + Slack + terminal, all the same agent.

---

## Phase 2 — The expert-agent model (design)

### The purpose

Hermes will sit in a shared Slack workspace where domain experts collaborate to
train a field-specific AI agent. The expert brings the knowledge; the agent
extends their reach. First expert: Prof. Pascal Lottaz (Kyoto University, neutrality
studies), who has been asking Niall to build exactly this since March 2025.

### The five-layer model (why expert agents work)

A complete expert agent is NOT "an AI that read some PDFs." It has five layers:

1. **Identity & epistemic stance** — who the agent is, how it's allowed to reason
   (the channel prompt)
2. **Structured domain knowledge** — the scaffold of the field: concepts,
   distinctions, cases, debates (the seed skill)
3. **Source corpus** — the actual PDFs/books/treaties, structured for accurate
   retrieval and citation (a real build step, not drag-and-drop)
4. **Reasoning conventions** — citation, confidence-flagging, distinguishing
   consensus from contested interpretation
5. **The correction loop** — how the expert improves it over time, with an
   auditable, attributed provenance trail

The provenance trail is the value: it turns "an AI that read PDFs" into "the agent
trained by Professor Lottaz," which is the authority no generic model has.

### Why experts will enjoy it

It's leverage on a life's work — scholarship that reaches hundreds through papers
becomes available to anyone, any time. It doesn't replace the expert, it extends
their reach. And it offers first-mover authorship of the canonical agent in their
field. For someone who's spent fifteen years on a niche subject, that's a gift,
not a chore.

### What was built

A three-file package for the `#neutrality-agent` channel:
- **Channel prompt** — defines the agent as a neutrality scholar, with Lottaz's
  own "I study neutrality, I am not neutral" distinction baked in as the
  foundational rule
- **Seed skill** — broad scaffold of the field (concepts, legal foundations,
  canonical cases, periodisation, live debates, key works), every section marked
  `[SEED]` to show it awaits expert depth
- **Training protocol** — house rules + the correction-log mechanism

Saved to outputs. Not yet deployed to the server.

---

## Phase 3 — GLM 5.2 as default model

### What was done

The default model was switched from `openrouter/owl-alpha` to `glm-5.2` via Z.ai,
using `hermes model`. Base URL `https://api.z.ai/api/paas/v4`. Reason: GLM 5.2 is
strong and cheap on Niall's coding plan, suitable as an everyday default while
keeping Claude for high-stakes work (the neutrality agent, An Ceisteoir) via
`/model` switching.

### Coding Helper — Claude Code on GLM

Separately, `@z_ai/coding-helper` was used to point **Claude Code** (the CLI coding
tool) at GLM's Anthropic-compatible endpoint. "Configuration synchronized" confirmed.

**What this means:** Claude Code is just a client that speaks the Anthropic API
format. Coding Helper repointed its base URL and key to GLM. So the tool is branded
"Claude Code" but the model answering is GLM 5.2. For the honesty standard: the
accurate description of the stack is "Claude Code as the harness, GLM 5.2 as the
model" — not "using Claude."

---

## Problems encountered (and what they taught)

### 1. The GLM API key was set to literal text "hermes model"

`grep -i glm ~/.hermes/.env` revealed `GLM_API_KEY=hermes model` — the words
"hermes model" had been captured as the key instead of the real key, during the
setup wizard. This caused every GLM call to fail with HTTP 401 ("token expired or
incorrect").

**The lesson:** a 401 is an *authentication* failure (key rejected), distinct from
a 404 (model not found). When debugging, the error code tells you which layer is
broken. The key was the problem, not the model name.

### 2. sed kept failing on the key fix

Attempts to fix the key with `sed` threw "unterminated `s` command" repeatedly,
because the GLM key contains a `.` and other characters that clash with sed's
pattern syntax unless heavily escaped.

**The lesson:** for replacing a value containing dots/special characters, don't use
sed. Either use nano (literal typing, no escaping) or the grep-filter-and-append
method:
```bash
grep -v "^GLM_API_KEY=" ~/.hermes/.env > ~/.hermes/.env.tmp && \
  echo "GLM_API_KEY=THEFULLKEY" >> ~/.hermes/.env.tmp && \
  mv ~/.hermes/.env.tmp ~/.hermes/.env
```

### 3. nano edit didn't save

A nano attempt to fix the key didn't write — grep still showed the broken value
afterwards. Likely exited without saving (Ctrl+X → Y → Enter sequence not completed).

---

## Current state at end of session

- ✅ Slack connected, bot responding, shared-session mode set
- ✅ Default model switched to GLM 5.2 in config
- ✅ Claude Code wired to GLM via Coding Helper ("synchronized")
- ✅ Neutrality agent design complete (3-file package, saved, not yet deployed)
- ✅ **GLM authentication fixed** — the `GLM_API_KEY` in `~/.hermes/.env` was
  corrected (it had been holding the literal text "hermes model" instead of the
  real key). Hermes now authenticates against GLM 5.2 and responds.

## Outstanding — to do next session

1. **Confirm the GLM model string.** GLM auth now works. If a 404 ever appears,
   the model name may need to be `glm-5` or `glm-4.6` rather than `glm-5.2` — check
   which strings the Z.ai plan exposes. (Keep the model-switching habit: GLM as the
   cheap default, Claude via `/model` for deep research, strategising, the
   neutrality agent, and An Ceisteoir.)
3. **Deploy the neutrality agent.** Create the `#neutrality-agent` channel, add the
   channel prompt to config.yaml under `slack.channel_prompts`, save the seed skill
   to `~/expert-agents/neutrality-agent/skill.md`, bind it via
   `channel_skill_bindings`, restart, invite the bot, self-test with the five
   verification questions (esp. "Are you neutral?" — must say it STUDIES neutrality).
4. **Draft the follow-up email to Lottaz** once the agent is live and self-tested —
   "it's already running, come and try it" is far stronger than "I'd like to build
   this."

---

## Glossary of terms used today

- **Socket Mode** — Slack connection method using WebSockets instead of a public
  URL; works behind a firewall.
- **App manifest** — a JSON file declaring a Slack app's scopes, events, and
  commands all at once; avoids manual configuration.
- **Bot token (`xoxb-`) / App token (`xapp-`)** — the two credentials a Socket Mode
  Slack bot needs.
- **Member ID (`U...`)** — Slack's internal user identifier, used for the allowlist.
- **Shared session** (`group_sessions_per_user: false`) — the whole channel shares
  one conversation context rather than per-user silos.
- **Channel prompt** — an ephemeral system prompt injected on every turn in a
  specific Slack channel; sets the agent's persona for that channel.
- **Channel skill binding** — a skill auto-loaded at session start in a specific
  channel; becomes part of the conversation history.
- **Five-layer model** — the framework for a complete expert agent: identity,
  structured knowledge, corpus, reasoning conventions, correction loop.
- **Correction log** — an append-only, attributed record of expert corrections;
  provides the agent's provenance and authority.
- **Coding Helper** — Z.ai's utility (`@z_ai/coding-helper`) that repoints Claude
  Code at GLM's Anthropic-compatible endpoint.
- **HTTP 401 vs 404** — 401 is an authentication failure (key rejected); 404 is
  resource-not-found (e.g. wrong model name). The code identifies which layer broke.
- **GLM 5.2** — Zhipu AI's model, accessed via Z.ai; cheap on Niall's coding plan.
