# Plan: Build the "Super Marketing Agent" in Dify (from scratch, UI walkthrough)

## Context

You already have:

- A self-hosted Dify instance running on a Vultr VPS, accessed through Twingate.
- An existing Dify app + Telegram bot (`@johnswarroom_bot`) powered by the **War Room Assistant** prompt.
- Four knowledge bases (MKB1–MKB4) created in Dify with content uploaded.
- A complete design at [MARKETING-AGENT-DESIGN.md](Dify/Super%20Marketing%20Agent/MARKETING-AGENT-DESIGN.md) describing **Option B**: a Chatflow that classifies user intent into 6 categories (STRATEGY, COPYWRITING, BRAND_REVIEW, RESEARCH, CAMPAIGN, GENERAL), retrieves from the right KBs per intent, and uses a persona-specific LLM for each branch.
- An importable YAML at [wr-marketing-agent.dify.yml](Dify/Super%20Marketing%20Agent/wr-marketing-agent.dify.yml) (you have chosen NOT to use it — building manually to learn the platform).

**Goal:** A new Dify Chatflow named **"WR Super Marketing Agent"** that turns the same Telegram bot into a senior marketing strategist, with classifier-driven persona routing, grounded in MKB1–MKB4. Once verified, repoint the Telegram plugin from the old app to this new one.

**Why manually:** you want to learn each Dify node hands-on, not just import a black-box DSL.

---

## Files referenced

- **Design source of truth:** `Dify/Super Marketing Agent/MARKETING-AGENT-DESIGN.md` — copy persona prompts verbatim from §"The Six Personas".
- **Existing assistant prompt (for diffing):** `Dify/Super Marketing Agent/war-room-assistant-system-prompt.md` — shows the `{{#context#}}` wiring pattern and Telegram formatting rules to carry forward.
- **Backup reference if you get stuck:** `Dify/Super Marketing Agent/wr-marketing-agent.dify.yml` — same Chatflow expressed as DSL; open in a text editor to verify field values when manual config feels off.

---

## Pre-flight checks (do these before touching the UI)

1. **Connect via Twingate.** Confirm your Dify Studio URL loads in the browser. (If not, see `war-room-security` skill.)
2. **Verify MKB1–MKB4 exist.** In Dify, click **Knowledge** in the top nav. Confirm 4 datasets are listed and each has at least 1 document indexed (green "Available" status).
3. **Capture each KB's dataset ID.** Click each KB → look at the URL (`.../datasets/<ID>/documents`) → save the 4 IDs in a scratch note. You'll select KBs by name in the UI, but having IDs handy helps if a node mis-binds.
4. **Verify your default model provider.** Settings → Model Provider. Confirm at least one of: OpenRouter / Anthropic / OpenAI is configured with credit. Recommended: **`anthropic/claude-haiku-4.5`** for the persona LLMs (cost/quality balance per design doc) and **`anthropic/claude-haiku-4.5`** also for the classifier (cheap classification is fine).
5. **Note your existing Telegram-connected app.** You'll repoint the plugin at the end — don't delete the old app until the new one is verified.

---

## Build steps (click-by-click)

### Step 1 — Create the Chatflow app

1. Dify Studio → **Studio** tab → click **Create App** (top right) → choose **Chatflow** (NOT Workflow, NOT Agent).
2. Name: `WR Super Marketing Agent`. Description: `Classifier-routed marketing strategist over MKB1–4. Connects to @johnswarroom_bot.`
3. Click **Create**. You land in the visual editor with a default `Start → LLM → Answer` skeleton.
4. Delete the default LLM node (right-click → Delete). Keep `Start` and `Answer`.

### Step 2 — Add the Question Classifier

1. Click the `+` after the **Start** node → **Question Classifier**.
2. **Input Variable:** `sys.query` (the user's Telegram message).
3. **Model:** `anthropic/claude-haiku-4.5`, temperature `0.0`.
4. **Classes:** add 6 classes — copy the labels and descriptions exactly:

| Class ID | Label | Description (paste into the "description" field) |
|----------|-------|-----|
| 1 | `STRATEGY` | Questions about go-to-market, positioning, prioritization, strategic options, what to do next, audience targeting at the strategic level. |
| 2 | `COPYWRITING` | Requests to write or draft copy: emails, ads, social posts, landing pages, scripts, headlines, hooks, CTAs. |
| 3 | `BRAND_REVIEW` | Requests to review, audit, critique, or fix existing copy. User is sharing copy and asking for feedback. |
| 4 | `RESEARCH` | Questions about audience, ICP, personas, competitors, market size, customer pain, jobs-to-be-done. |
| 5 | `CAMPAIGN` | Requests to plan a full campaign or launch — multi-channel, multi-week, with KPIs and timeline. |
| 6 | `GENERAL` | Anything else marketing-related that doesn't cleanly fit the above. Use as the catch-all. |

5. Save the node. You should now see 6 output handles on the right edge of the classifier.

### Step 3 — Build one branch end-to-end (STRATEGY) as the template

This is the pattern you'll repeat 6×. Build it once carefully, then duplicate.

**3a. Knowledge Retrieval node**

1. Drag from the classifier's `STRATEGY` handle → drop → choose **Knowledge Retrieval**.
2. **Query Variable:** `sys.query`.
3. **Knowledge Bases:** click **+ Add** → select **MKB1** and **MKB3** (per design doc — strategy uses Brand and Audience).
4. **Retrieval Setting:** keep **Hybrid** (vector + keyword). Top K = `5`. **Score threshold = 0.3** (per `war-room-assistant-system-prompt.md` — default 0.5 is too strict for these KBs).
5. Save.

**3b. LLM node**

1. Drag from Knowledge Retrieval's output → **LLM**.
2. **Model:** `anthropic/claude-haiku-4.5`, temperature `0.4`.
3. **Context:** in the right panel find the **Context** section → toggle ON → variable selector → pick the upstream Knowledge Retrieval node's `result`. (This is what makes `{{#context#}}` resolve. If you skip this, the prompt will literally print `{{#context#}}` in the reply — see [war-room-assistant-system-prompt.md:51-62](Dify/Super%20Marketing%20Agent/war-room-assistant-system-prompt.md).)
4. **System prompt:** paste the **Common Base** + **STRATEGIST persona** + retrieval block, copied verbatim from [MARKETING-AGENT-DESIGN.md:48-71](Dify/Super%20Marketing%20Agent/MARKETING-AGENT-DESIGN.md). End the system prompt with:
   ```
   === RETRIEVED CONTEXT ===
   {{#context#}}
   === END CONTEXT ===

   Telegram formatting rules:
   - No wide tables. Use list-of-lines instead ("Clarity: 7/10 — ...").
   - If your output would exceed ~1500 chars, end with a question that invites the user to drill into one section.
   ```
5. **User prompt:** `{{#sys.query#}}`.
6. Save.

**3c. Answer node**

1. Drag from the LLM node → **Answer**.
2. **Answer:** `{{#llm.text#}}` (pick the LLM node's text output via the variable selector).
3. Save.

### Step 4 — Repeat for the other 5 branches

For each of `COPYWRITING`, `BRAND_REVIEW`, `RESEARCH`, `CAMPAIGN`, `GENERAL`:

1. Right-click the STRATEGY chain (Knowledge Retrieval + LLM + Answer) → **Duplicate**. Drag the cloned chain near the matching classifier handle. Connect.
2. Edit the **Knowledge Retrieval** node's KB list to match the design doc:

| Branch | KBs to select |
|---|---|
| COPYWRITING | MKB1, MKB2, MKB4 |
| BRAND_REVIEW | MKB1 |
| RESEARCH | MKB3 |
| CAMPAIGN | MKB1, MKB2, MKB3, MKB4 |
| GENERAL | MKB1, MKB2, MKB3, MKB4 |

3. Edit the **LLM** node's system prompt — replace the persona section with the matching one from `MARKETING-AGENT-DESIGN.md` §"The Six Personas":
   - COPYWRITING → §73-86 (COPYWRITER)
   - BRAND_REVIEW → §88-99 (BRAND REVIEWER)
   - RESEARCH → §101-112 (RESEARCHER)
   - CAMPAIGN → §114-128 (CAMPAIGN ARCHITECT)
   - GENERAL → §130-138 (GENERAL)
4. Re-bind the LLM node's **Context** to *its own* Knowledge Retrieval node (not STRATEGY's — easy mistake when duplicating).

> **Optional cost optimization:** for **CAMPAIGN** only, swap the model to `claude-sonnet-4.5` — the design doc calls this out as the highest-stakes persona worth the ~5× cost.

### Step 5 — Visual layout cleanup

You should now have: `Start → Question Classifier → (6 branches) → 6 Answers`. Total ~20 nodes. Drag them into a tidy fan-out so the canvas reads left-to-right. Save.

### Step 6 — Test inside Dify Studio (before touching Telegram)

1. Top right → **Debug and Preview** (the chat icon).
2. Run one prompt per intent and confirm routing + retrieval. Suggested test set:
   - STRATEGY: *"Should we double down on email or pivot to LinkedIn for Q3?"*
   - COPYWRITING: *"Write 3 cold-email subject lines for our flagship offer."*
   - BRAND_REVIEW: *"Review this: 'Unlock cutting-edge synergy with our innovative platform.'"* (should flag banned words)
   - RESEARCH: *"What do we know about our ICP's biggest objection?"*
   - CAMPAIGN: *"Plan a 4-week launch campaign for the new offer."*
   - GENERAL: *"What's a good marketing book?"*
3. After each response, click the relevant LLM node in the canvas → **Inputs** panel → confirm `context` is populated with retrieved chunks (not empty). If empty: lower `score_threshold` further on that branch's Knowledge Retrieval node, or verify the right MKBs are selected.
4. Confirm the classifier routed correctly — click the Question Classifier node → **Outputs** → see which class it picked.

### Step 7 — Publish

1. Top right → **Publish** → **Publish** (creates v1).
2. Copy the new app's **API endpoint** and **API key** from the **API Access** tab on the left sidebar — you'll need both for the Telegram plugin.

### Step 8 — Repoint the Telegram plugin

1. Open your existing Telegram bot configuration (wherever the plugin lives — Dify's Tools/Plugins panel, or n8n if that's how you wired it).
2. Replace the API endpoint + API key with the **WR Super Marketing Agent** values from Step 7.
3. **Do NOT delete the old "War Room Assistant" app yet.** Keep it as a fallback for 1 week.
4. Send a test Telegram message to `@johnswarroom_bot`. Confirm the response style now matches one of the 6 personas, not the old generic assistant.

---

## Verification (end-to-end)

| Check | Pass criteria |
|---|---|
| Studio Debug — all 6 intents | Each test prompt routes to the right class (visible in classifier Outputs) and produces persona-shaped output (e.g. STRATEGY ends with "the one metric to watch", BRAND_REVIEW shows scorecard) |
| Context wiring | Every LLM node's `context` Input is populated with non-empty chunks during a test run |
| Telegram round-trip | A real message to `@johnswarroom_bot` returns a Super Marketing Agent response (not the old War Room Assistant tone) within ~5s |
| Cost sanity | Settings → Billing/Usage shows ~$0.008–0.04/turn depending on which persona ran (per design doc estimates) |
| Fallback works | Switching the Telegram plugin back to the old app's API key still works (proves you didn't break the old app) |

---

## After it's live

- Use it daily for a week. Note which intents you actually use and where output is weak — that tells you which persona prompts to tighten and which MKBs need more content.
- Per the design doc Iteration Plan: Week 3 is per-intent prompt tuning; Week 4 is adding `/feedback` Telegram command. Both are out of scope for this plan.
- If you want the old War Room Assistant *and* the Super Marketing Agent both reachable, set up a `/marketing` Telegram command that routes to the new app while the default behavior keeps the old one. (Separate plan.)

---

## Rollback

If anything misbehaves after Step 8:

1. In the Telegram plugin config, paste back the old app's API endpoint + key.
2. The old "War Room Assistant" app keeps running unchanged — no data loss, no migration.
3. Diagnose the new app inside Dify Studio's Debug pane without users hitting it.
