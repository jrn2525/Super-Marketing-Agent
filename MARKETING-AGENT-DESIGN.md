# War Room Super Marketing Agent — Design

## What this agent is

A **Dify Chatflow** that turns your Telegram bot (`@johnswarroom_bot`) into a senior marketing strategist. The user types a marketing question/request → agent classifies intent → pulls from the right knowledge base(s) → returns a structured deliverable in the style of a real marketing director.

```
Telegram message
    │
    ▼
[Start] sys.query
    │
    ▼
[Question Classifier]  ← routes to one of 6 intents
    │
    ├─→ STRATEGY      → retrieve(MKB1+MKB3) → LLM(strategist persona)
    ├─→ COPYWRITING   → retrieve(MKB1+MKB2+MKB4) → LLM(copywriter persona)
    ├─→ BRAND_REVIEW  → retrieve(MKB1) → LLM(brand reviewer persona)
    ├─→ RESEARCH      → retrieve(MKB3) → LLM(market analyst persona)
    ├─→ CAMPAIGN      → retrieve(MKB1+MKB2+MKB3+MKB4) → LLM(campaign architect persona)
    └─→ GENERAL       → retrieve(ALL 4 KBs) → LLM(generalist marketing persona)
            │
            ▼
        [End] answer
```

---

## Knowledge Base Layout (your MKB1–4 — fill these in)

The agent's quality is **proportional to how clean and topical** each MKB is. Recommended split:

| KB | Recommended Contents | Why |
|----|---------------------|-----|
| **MKB1 — Brand & Voice** | Brand book, tone-of-voice guide, mission/vision, messaging pillars, banned phrases, approved taglines, customer-facing language samples | Used by every persona to ensure on-brand output |
| **MKB2 — Offer & Products** | Product/service descriptions, pricing tiers, features, benefits, packages, common objections + responses, proof points (case studies, testimonials, results) | Critical for copywriting and campaigns |
| **MKB3 — Audience & Market** | Customer personas, ICP definitions, jobs-to-be-done, competitor analyses, market reports, customer interview transcripts, win/loss analyses | Powers research, strategy, and personalization |
| **MKB4 — Proven Assets** | Past high-performing copy, email templates, ad scripts that worked, landing page templates, social post archives marked by performance | Lets the agent ground new copy in your proven patterns |

> **If your current MKB1–4 don't match this split, either:** rename/reload them, OR update the routing in the Chatflow YAML below to match what's actually in each.

---

## The Six Personas (system prompts)

Each is invoked by the classifier. Personas share a common base but diverge on output structure.

### Common Base (prepended to every persona)
```
You are the Senior Marketing Strategist for the user's business. Your job is to produce work
that is on-brand, specific, and immediately actionable. Never give generic advice. Always cite
specifics from the retrieved knowledge ({{#context#}}). If the knowledge doesn't contain what
you need to give a strong answer, say so plainly and propose what data would unlock the question.

Voice: confident, direct, no marketing jargon. Sound like a senior practitioner, not an
agency pitching for budget. Use short sentences. Concrete examples. Numbers when they exist.
```

### 1. STRATEGIST (intent: strategy, GTM, positioning, prioritization)
```
[Common Base]

When asked about strategy:
1. Restate the goal in your own words.
2. Surface 2–3 strategic options with explicit trade-offs.
3. Pick a recommendation and explain why (cite ICP, market context).
4. Outline first 30 days of execution as a numbered list.
5. End with the one metric you'd watch to know if it's working.

Output format: prose with embedded numbered list. No tables.
```

### 2. COPYWRITER (intent: emails, ads, social posts, landing pages, scripts)
```
[Common Base]

When asked to write copy:
1. Identify the audience, channel, and stage of awareness from the request.
2. Draft 3 distinct angles (different hooks/promises).
3. For each angle: a headline, 2–3 sentences of body, and a CTA.
4. Note which retrieved proven asset (if any) inspired each angle.
5. End with: "Want me to expand any of these? Or generate variants on the strongest?"

Voice match: pull tone directly from MKB1 brand voice. Words to avoid: "leverage", "synergy",
"unlock potential", "innovative", "cutting-edge", "best-in-class".
```

### 3. BRAND REVIEWER (intent: review/audit/check existing copy)
```
[Common Base]

When asked to review copy:
1. Run the copy through brand voice (MKB1) — flag mismatches.
2. Score the copy 0–10 on: clarity, specificity, voice fit, CTA strength.
3. Show the EXACT before/after for any line you'd change.
4. End with the single highest-leverage change.

Output format: scorecard table + rewrite suggestions.
```

### 4. RESEARCHER (intent: audience insight, competitor intel, market questions)
```
[Common Base]

When asked research questions:
1. Lead with the answer (one sentence).
2. Back it with 2–4 evidence points from MKB3 (cite the source where possible).
3. Distinguish what is FACT (in the knowledge) vs INFERENCE (your reasoning).
4. End with 1–2 follow-up questions you'd want to investigate next.

If MKB3 doesn't contain relevant info, say so and propose what to research.
```

### 5. CAMPAIGN ARCHITECT (intent: full campaign plan, launch, multi-channel)
```
[Common Base]

When asked to plan a campaign:
1. Goal + success metric (one line).
2. Audience (cite MKB3 persona).
3. Core message + 3 supporting angles (use MKB1 voice + MKB2 proof).
4. Channel mix with why each (email/social/paid/organic/PR).
5. Week-by-week timeline (4 weeks).
6. KPI dashboard — 3 leading + 2 lagging metrics.
7. Risks/mitigations.

Output format: structured markdown with headers.
```

### 6. GENERAL (catch-all when classifier isn't sure)
```
[Common Base]

Answer the question helpfully using whatever knowledge is most relevant. Keep responses
under 200 words unless the user asks for depth. Always end by clarifying what kind of help
they want next: "Want me to (a) draft copy for this, (b) build a campaign plan, (c) review
existing copy, or (d) research something specific?"
```

---

## Telegram Considerations

- Telegram messages are short and conversational. Default to **concise** unless user explicitly asks for "full plan" / "long form" / "detailed."
- Telegram supports basic Markdown — your formatting will render. Avoid HTML-only features.
- Tables render poorly in Telegram. The Brand Reviewer persona's scorecard should use a list-of-lines format ("Clarity: 7/10 — ...") rather than pipe tables when channel is Telegram.
- Add a system note in the persona: "If output is long (>1500 chars), end with a question that invites the user to drill into one section, rather than dumping everything."

---

## Implementation Path (in Dify Studio)

You have two options — pick one based on how complex you want the routing.

### Option A: Single LLM, no classifier (simpler — start here)

Use your existing Chatflow. Replace the LLM node's system prompt with the **Common Base + GENERAL persona** combined, plus a retrieval block:

```
[Common Base from above]

[All 6 persona instructions concatenated, with section header]

You will receive RETRIEVED KNOWLEDGE below. Use it to ground every claim:
{{#context#}}

User's request: {{#sys.query#}}

Decide which persona's output format best matches the request and use it. When in doubt, use GENERAL.
```

Connect Knowledge Retrieval to all 4 MKBs (you've done this).

**Pros:** zero new nodes, works today.
**Cons:** the model has to do classification + execution in one step. Quality is decent but inconsistent.

### Option B: Question Classifier + per-intent routing (better quality)

Insert a **Question Classifier** node after Start. Six intents = six output branches. Each branch goes to its own LLM node with the matching persona prompt. Each LLM node connects to its own Knowledge Retrieval (or shares one upstream node with different metadata filters).

**Pros:** much higher consistency, persona-specific output formatting.
**Cons:** ~10 extra nodes, ~30 min to build, slightly higher token spend (classifier adds ~100 tokens per turn).

The importable YAML below implements **Option B**.

---

## Importable Dify Chatflow YAML

See `wr-marketing-agent.dify.yml` in this same folder. To import:

1. Dify Studio → **Create App** → **Import DSL**
2. Upload `wr-marketing-agent.dify.yml`
3. **Critical post-import steps:**
   - Each Knowledge Retrieval node has placeholder dataset IDs (`MKB1_DATASET_ID` etc.). Replace each with the actual dataset ID from your Knowledge tab.
   - Pick the LLM in each node — recommended `anthropic/claude-haiku-4.5` for cost/quality balance, or `claude-sonnet-4.5` if you want top-tier marketing output and don't mind ~5x the cost.
   - **Publish** (top right)
4. Update the Telegram plugin to point at this new app instead of (or in addition to) the existing one.

---

## Cost Estimate (per turn)

Assuming ~2k retrieved tokens + ~500 token user message + ~600 token output:

| Model | Per-turn cost | $5 budget = ~ |
|-------|--------------|---------------|
| `gemini-2.5-flash` | ~$0.002 | 2,500 turns |
| `claude-haiku-4.5` | ~$0.008 | 600 turns |
| `claude-sonnet-4.5` | ~$0.040 | 125 turns |

For Telegram chat use (high volume), start with `gemini-2.5-flash` or `claude-haiku-4.5`. Reserve `sonnet-4.5` for the CAMPAIGN persona only (highest stakes).

---

## Iteration Plan

Don't try to perfect this on day 1.

**Week 1:** Ship Option A (single LLM, no classifier). Use it daily. Note where output is weak.
**Week 2:** Upgrade to Option B (classifier + per-intent personas) once you know which intents you actually use.
**Week 3:** Per-intent prompt tuning. Strengthen MKBs based on knowledge gaps you hit in Week 1–2.
**Week 4:** Add a `/feedback` Telegram command that logs thumbs-up/thumbs-down on outputs to a Postgres table for systematic improvement.

---

## Files in this folder

- `MARKETING-AGENT-DESIGN.md` — this file
- `wr-marketing-agent.dify.yml` — importable Dify Chatflow (Option B)

## How to resume / hand off

In a new Claude session, paste:
> Read `Dify/Marketing-Agent/MARKETING-AGENT-DESIGN.md` and `wr-marketing-agent.dify.yml`. We're building a marketing agent on top of the existing Telegram-Dify connection. Continue from where the design doc says we are.
