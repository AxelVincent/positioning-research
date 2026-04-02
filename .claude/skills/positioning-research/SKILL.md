---
name: positioning-research
description: Conduct deep market and positioning research for any product or idea. Works in two modes — give it a URL to analyze an existing product's website, or give it a short memo describing your idea. Uses WebSearch and WebFetch for most research (including Reddit via site:reddit.com + old.reddit.com, and G2/Capterra reviews). Browser tools (Chrome or playwright-cli) used only as fallback. Asks clarifying questions before diving in. Produces a comprehensive positioning document with concrete pricing recommendations. Use this whenever the user mentions market research, positioning, competitive analysis, or wants to understand how a product fits in its market.
argument-hint: "[URL or short product memo, e.g. 'https://linear.app' or 'A tool that helps sales teams automate follow-up emails based on CRM activity']"
---

# Positioning & Market Research

You are conducting positioning and market research. The user provided: **$ARGUMENTS**

## Research Philosophy

This skill exists to help someone make a high-stakes decision — whether to build, invest, pivot, or kill an idea. The worst outcome is false confidence. Every phase of research is oriented around **invalidation**: finding reasons the thesis is wrong, then seeing what survives.

**Source credibility tiers** — weight evidence accordingly:

1. **Tier 1 (strongest):** Churned user language ("I left X because..."), negative reviews, user complaints, support threads, "I switched from" posts
2. **Tier 2:** Job postings in the category (real demand signal), hiring velocity of competitors, actual usage data (GitHub stars trends, npm downloads, BuiltWith adoption)
3. **Tier 3:** Funding data, revenue estimates, analyst reports from credible firms (Gartner, Forrester — but note these firms sell reports, so size estimates skew high)
4. **Tier 4 (weakest):** Competitor marketing copy, press releases, Product Hunt launches, self-reported metrics

When Tier 4 sources contradict Tier 1-2 sources, Tier 1-2 wins. Always. A competitor's homepage says "loved by 10,000 teams" but G2 reviews say "we churned after 3 months" — lead with the G2 data.

## Step 0 — Create an Isolated Research Workspace

Before doing anything else, create a workspace folder for this research run inside the `research/` directory.

**Naming convention:** `{product-name}-{research-topic}` (lowercase kebab-case)

```bash
RESEARCH_DIR="research/{product-name}-{research-topic}"
mkdir -p "$RESEARCH_DIR"
RUN_ID="research-$(date +%Y%m%d-%H%M%S)"
```

All playwright-cli sessions must use session names prefixed with `${RUN_ID}`. **All files go into `$RESEARCH_DIR/`.** Nothing in the project root.

## Step 1 — Determine Input Mode

Look at `$ARGUMENTS`:

- **URL mode** — input contains a URL. Scrape the website to understand the product.
- **Memo mode** — input is text. Work from the description.

Carry any focus directives ("focus on the developer tooling angle") through the entire research.

## Step 2 — Understand the Product

### URL Mode: Scrape the Website

Use WebFetch to read the main page. If it fails (empty, 403, JS-only), fall back to browser.

Discover subpages from actual navigation links — don't guess paths like `/pricing`. Follow the 2-5 most useful ones.

### Memo Mode: Work from the Description

Extract: what it does, who it's for, how it works, what makes it different.

### Both Modes: Internal Summary

- What the product does (1 paragraph)
- Target audience
- Core value propositions (3-5 bullets)
- Claimed differentiators — flag these as *claimed*, not validated
- Pricing model (if known)
- Category terms to research

## Step 3 — Ask Clarifying Questions

Pause and ask 2-5 questions. Good ones:

- **Target audience**: "Who's the primary buyer — practitioners, team leads, or company-wide?"
- **Competitive awareness**: "Competitors you already know about?"
- **Differentiation**: "What's hardest for a competitor to replicate?"
- **Stage**: "Existing product you're repositioning, or new idea you're validating?"
- **Category**: "What would someone Google to find this?"
- **Priorities**: "What matters most — competitive intel, user language, positioning, or something else?"

**Wait for answers before proceeding.**

## Prerequisites

- **Chrome browser with claude-in-chrome extension** (preferred)
- **playwright-cli** (fallback) — must be in PATH

## Browser Tool Selection

1. Try `mcp__claude-in-chrome__tabs_context_mcp`. If it works, use Chrome.
2. Otherwise, fall back to playwright-cli.

## Research Tool Selection

Use the lightest tool that works:

| Task | Tool | Why |
|------|------|-----|
| Search queries | **WebSearch** | Fast, no browser |
| Read pages, blogs, docs | **WebFetch** | Direct content grab |
| Reddit discovery | **WebSearch** (`site:reddit.com`) | Google index finds threads |
| Reddit reading | **WebFetch** (`old.reddit.com/r/.../comments/ID.json`) | `.json` endpoint returns structured data; fall back to `old.reddit.com` HTML |
| Reddit fallback | **Chrome/playwright-cli** | Escalation only |
| G2 reviews | **WebSearch** + **WebFetch** | G2 acquired Capterra (Jan 2026) — review ecosystem is now consolidated. Search both but expect overlap |
| Google Trends | **Chrome** first, then proxy sources | Browser often fails; use explodingtopics.com or trend articles |
| Discord servers | **WebSearch** (`site:discord.com` or `"discord.gg"`) | Find public servers; can't read private channels |
| Job postings | **WebSearch** (`site:linkedin.com/jobs` or `site:greenhouse.io`) | Demand signal — companies hiring = real need |
| Glassdoor/Blind | **WebSearch** + **WebFetch** | Employee sentiment = competitor health signal |
| JS-only/403 sites | **Chrome/playwright-cli** | Last resort |

## Web Fetching Protocol — Haiku Summarization Layer

Every WebFetch in this skill goes through a summarization layer to control token cost. **No agent should WebFetch a page directly into its own context.**

Instead, for each URL that needs reading, spawn a **Haiku subagent** (`model: "haiku"`) with this pattern:

```
Fetch this URL and extract ONLY the following:
- URL: [the URL]
- Extract: [what you need — e.g., "user complaints about [product]", "pricing tiers and price points", "main arguments for/against [category]"]

Rules:
- Use WebFetch to read the page
- Extract only what was asked for — ignore navigation, ads, sidebars, unrelated content
- If the page is a Reddit thread, extract: top-level post summary, top 5-10 comments by relevance, any specific quotes about [topic]
- If the page is a review site, extract: rating, pros, cons, reviewer context, key quotes
- If the page is a pricing page, extract: tier names, prices, what's included, billing model
- Keep your response under 500 words
- Include the source URL at the top
- If the page fails to load or is paywalled, say so in one line
```

This means a Reddit thread that's 80K tokens raw becomes a 500-token extract. The Sonnet agent never sees the full page — only the Haiku-compressed summary.

**Batch fetches when possible.** If an agent needs to read 8 URLs, spawn 8 Haiku subagents in parallel, collect all summaries, then proceed with analysis.

**Cap: each Phase 2 agent should fetch at most 15 URLs total.** Prioritize ruthlessly — not every search result needs fetching. Read the WebSearch snippets first and only fetch pages where the snippet suggests high-value content.

## Phase 2 — Parallel Research

Launch **4 parallel agents** using `model: "sonnet"`. All agents use the Haiku summarization layer for WebFetch (see above). They use WebSearch directly.

### Agent 1: Competitive Reality Check (Third-Party Evidence Only)

This agent builds the competitive picture from *external evidence*, not competitor marketing copy. The goal is to understand what competitors actually deliver, not what they claim.

**Step 1 — Discover competitors via WebSearch:**
- `"[category]" alternative`
- `best [category] tools [current year]`
- `"[category]" vs` (comparison articles from third parties)
- `"[category]" review site:g2.com`

**Step 2 — For each competitor, gather third-party evidence:**
- G2/Capterra reviews (especially 2-3 star reviews — these are most informative)
- HN threads mentioning the competitor
- Reddit threads discussing the competitor
- Blog posts comparing competitors (from users, not the competitors themselves)

**Step 3 — Only then, check pricing pages:**
Read competitor pricing pages via WebFetch — but treat pricing as factual data, not their marketing narrative. Capture tiers, price points, model type.

**Step 4 — Pricing landscape analysis:**
- Map price ranges across the category
- Identify pricing model patterns (per-seat, usage, flat, freemium)
- Where are the gaps?
- What does pricing signal about willingness-to-pay?

**Step 5 — Job posting scan (demand signal):**
- `site:linkedin.com/jobs "[category]"` or `site:greenhouse.io "[category term]"`
- Count open roles that would use tools in this category
- Note which companies are hiring — this is a real demand signal, harder to fake than market reports

Save to `$RESEARCH_DIR/competitive-landscape.md` (**max 2,000 words**): competitor table (name, what users actually say, pricing, G2 rating, URL), pricing landscape, positioning gaps, job posting demand signals. Be dense — tables over prose.

**What NOT to include:** competitor taglines, hero copy, or self-described positioning. If you reference what a competitor claims, explicitly frame it as "Competitor X claims..." and contrast it with user evidence.

### Agent 2: User Pain Points & Switching Signals

This agent gathers authentic user language. It is the most important agent — user voice is the foundation of everything else.

**Layer 1 — Reddit via WebSearch + WebFetch (primary):**

```
WebSearch: site:reddit.com "[category]" pain OR frustration OR switched OR alternative
WebSearch: site:reddit.com "[competitor name]" review OR experience OR "switched from"
```

Read threads via WebFetch using `old.reddit.com`. For structured data, try the `.json` endpoint:
```
WebFetch: https://old.reddit.com/r/SUBREDDIT/comments/THREAD_ID/.json
```
Falls back to HTML if `.json` is blocked.

Run 8-12 searches across 3-5 subreddits.

**Layer 2 — G2 reviews (focus on negative/mixed):**

G2 acquired Capterra in January 2026 — the review ecosystem is consolidated. Focus on 2-3 star reviews (most informative) and "switched from" mentions.

```
WebSearch: site:g2.com "[competitor name]" reviews
WebSearch: "[competitor name]" "switched from" OR "moved to" OR "pros and cons"
```

**Layer 3 — Discord (important for dev tools and community-led products):**

Many authentic discussions now happen in Discord, not Reddit. Public servers are discoverable:

```
WebSearch: "[category]" discord server OR discord.gg
WebSearch: site:discord.com "[category]"
```

Note which Discord communities exist and their size. If threads are publicly indexed, read them.

**Layer 4 — HN and community forums:**
```
WebSearch: site:news.ycombinator.com "[category]" OR "[competitor]"
WebSearch: "[category]" forum discussion pain OR frustration
```

**Layer 5 — Blind/Glassdoor (employee sentiment as competitive intel):**

Competitor employee sentiment reveals things marketing never will — low morale, talent flight, leadership chaos, product direction disagreements.

```
WebSearch: site:glassdoor.com "[competitor name]" reviews
WebSearch: site:teamblind.com "[competitor name]"
WebSearch: "[competitor name]" glassdoor OR blind employee review
```

This is Tier 2 evidence — treat it seriously.

**What to extract across all layers:**
- How users describe the problem (their words, not the product's words)
- What they've tried and what failed
- Switching triggers — the specific moment they decided to leave
- What they wish existed
- Employee sentiment about key competitors (morale, direction, talent retention)

**Citation rule: Every quote MUST have a direct URL.** No secondhand quotes. If you can't link to the original, mark as `[unverified — secondhand source]`.

Save to `$RESEARCH_DIR/user-signals.md` (**max 2,500 words**): user language, pain points ranked by frequency, switching triggers, competitor employee sentiment, key quotes with URLs. Prioritize direct quotes over summaries — quotes are the raw material the synthesis needs.

### Agent 3: Trend Validation & Structural Shifts

**Google Trends — try browser first, fall back to proxies:**

Using Chrome:
```
mcp__claude-in-chrome__tabs_create_mcp → get tabId
mcp__claude-in-chrome__navigate(tabId, "https://trends.google.com/trends/explore?q=TERM1,TERM2,TERM3&hl=en")
mcp__claude-in-chrome__read_page(tabId)
mcp__claude-in-chrome__javascript_tool(tabId, "window.close()")
```

If browser fails, use proxies:
```
WebSearch: "google trends" "[category term]" growth OR rising OR declining
WebSearch: site:explodingtopics.com "[category term]"
```

**Funding signals:**
```
WebSearch: "[category]" funding OR raised OR series [current year]
WebSearch: "[category]" acquisitions OR shutdown OR pivot [current year]
```

Funding tells you where money flows. But also search for **shutdowns and pivots** — a competitor leaving the category is a signal too, and not always a positive one.

**Structural shifts — the most valuable part of this agent:**

Look for forces reshaping the segment. Not "the market is growing" but "the rules are changing."

```
WebSearch: "[category]" disrupted OR commoditized OR "paradigm shift" OR "future of" [current year]
WebSearch: "[category]" AI impact OR automation OR "no longer need" [current year]
WebSearch: "[adjacent technology]" replacing "[category]"
```

For each shift: what's changing, who wins/loses, timeline, evidence, implication for the product.

**Analyst coverage (contextualize, don't trust blindly):**
```
WebSearch: "[category]" Gartner OR Forrester OR "magic quadrant" [current year]
```

If Gartner has a Magic Quadrant → established category. Market Guide → emerging. Nothing → either too niche or too new.

Save to `$RESEARCH_DIR/trends-validation.md` (**max 1,500 words**): trend data with sources, funding/shutdown signals, structural shifts with evidence and timeline, analyst coverage context.

### Agent 4: Market Sizing & Invalidation

This agent does two things: size the market **and** actively look for reasons the thesis is wrong.

**Step 1 — Bottom-up market sizing (preferred over top-down):**

The best market sizing is traceable. Instead of citing a $4.2B TAM from Grand View Research, try to count real entities:

```
WebSearch: "how many [target users/companies]" [relevant geography]
WebSearch: "[target role]" linkedin results [geography]
WebSearch: site:linkedin.com/jobs "[category term]" — count job postings as demand proxy
```

If you can estimate: (number of potential customers) × (realistic price point) = a grounded SAM. Use 0.5-2% adoption rate, not 10%.

**Step 2 — Top-down as supplement (not primary):**

If analyst reports exist, cite them but add context:
```
WebSearch: "[category]" market size [current year]
```

Flag confidence level honestly. Grand View Research, Mordor Intelligence, etc. sell reports — their TAM estimates are marketing for the reports themselves. Treat them as upper bounds, not expected values.

**Step 3 — Platform risk scan:**
```
WebSearch: "[big player]" "[category feature]" announced OR launched OR acquired
```

Check if any major platform has shipped or announced overlapping features.

**Step 4 — Invalidation search (critical):**

Actively search for reasons this market/product might not work:

```
WebSearch: "[category]" failed OR shutdown OR "didn't work" OR "waste of money"
WebSearch: "[category]" "solution looking for a problem" OR "nobody needs"
WebSearch: site:news.ycombinator.com "[category]" dead OR overhyped OR "who uses"
WebSearch: "[similar product that failed]" post-mortem OR "lessons learned"
```

Look for:
- Startups that tried this and failed — why?
- HN threads where practitioners are skeptical
- Categories that sound big but have low actual adoption
- "Graveyard" signals — lots of funded companies, few survivors

This is the most important step. If you can't find reasons the thesis might be wrong, you haven't looked hard enough.

Save to `$RESEARCH_DIR/market-sizing-gtm.md` (**max 2,000 words**): bottom-up sizing with math shown, top-down as supplement with confidence flags, platform risk, **invalidation findings** (failed companies, skepticism, adoption barriers).

## Phase 3 — Deep Dives (Opt-In)

Deep dives are **not automatic**. After Phase 2 completes, review the intermediate reports and ask the user:

> "Phase 2 is done. Here's what I found: [2-3 sentence summary of key findings and gaps]. I'd recommend a deep dive into [specific question]. Want me to go deeper, or should I proceed to synthesis?"

If the user says proceed, skip to Phase 4. If they want deep dives, launch **1-2 agents** (not 3) using `model: "sonnet"`. Each deep dive uses the Haiku summarization layer for fetches. Choose based on what Phase 2 surfaced:

- **Deep-dive into the #1 threat** — actual user sentiment, employee morale, growth trajectory
- **Deep-dive into the invalidation case** — if Phase 2 found red flags, investigate further
- **Deep-dive into user pain** — more Reddit/G2/Discord threads, specific workflows
- **Deep-dive into adjacent market** — tools in neighboring categories

### Deep Dive Quality Criteria

Each deep dive must:
1. Answer a specific question (not just "research competitor X" — but "can competitor X's enterprise moat be broken?")
2. Use Tier 1-2 evidence primarily
3. Include at least 10 numbered citations with URLs
4. End with a clear "so what" — what this means for the product
5. Be honest about what it couldn't find or verify

No word minimums. Answer the question thoroughly, then stop. A focused 1,500-word deep dive that answers the question is better than a 5,000-word one that doesn't.

Save each to `$RESEARCH_DIR/` with descriptive filenames.

## Phase 4 — Synthesize

Write the final document to `$RESEARCH_DIR/positioning-research.md`.

### Citation System

Numbered references throughout, like a research paper.

- Inline: `[1]`, `[2]`, `[1][2]`
- References section at the end with full URL, title, author, date
- Every factual claim needs a reference
- Every user quote needs a direct URL

### Source Credibility Annotations

When citing sources, annotate credibility where it matters:

- `[3, competitor claim]` — this is what the company says about itself
- `[7, user review]` — this is what a user actually experienced
- `[12, analyst estimate]` — this is a paid report's projection

This helps the reader weight evidence themselves.

### Document Structure

```markdown
# [Product Name] — Positioning Research

**Date:** [today]
**Input:** [URL / Memo]
**Verdict:** [One sentence: go / no-go / conditional go / needs validation]

---

## 1. The Case Against

Lead with why this might not work. This section earns the reader's trust — if you can articulate the strongest reasons to walk away, the positive findings that follow carry more weight.

- Failed or struggling companies in this space and why [N]
- Strongest skeptic arguments found in practitioner communities [N]
- Market risks: timing, adoption barriers, category confusion
- Platform risk: which big players could enter and ship this as a feature [N]
- What this research couldn't answer — data gaps and unknowns

**Verdict on survivability:** After considering all the above, does the opportunity still hold? What specifically survives scrutiny?

## 2. Market Reality

### What Users Actually Say
- Pain points ranked by frequency and intensity [N]
- User language vs. product language — do they match?
- Key quotes with source URLs [N]
- Switching signals: why users leave current tools, what triggers the switch [N]

### What Users Have Tried
- Tools and workarounds mentioned, ranked by frequency
- What works for them, what doesn't
- The gap: what they wish existed but can't find

### Employee Sentiment (Competitor Health)
- Glassdoor/Blind signals on key competitors [N]
- Talent flow direction — who's hiring, who's losing people
- What this reveals about competitor stability and product direction

## 3. Competitive Landscape

### Competitor Overview
- Table: name, what users say about them (not their tagline), pricing, G2 rating, funding, URL
- Positioning map showing clusters and white space

### Pricing Landscape
- Price range across category
- Model comparison (per-seat, usage, flat, freemium)
- Pricing gaps and what they signal about willingness-to-pay

### Most Dangerous Competitors
- 1-2 competitors that matter most, based on user evidence not marketing claims
- What they do well (from user reviews), where they're weak (from negative reviews)
- **Decision point:** Can you win against them, and on what axis?

## 4. Market Sizing

### Bottom-Up (Primary)
- Number of potential customers in target segment, with sources [N]
- Realistic price point × realistic adoption rate (0.5-2%) = grounded estimate
- Math shown, assumptions explicit

### Top-Down (Supplement)
- Analyst estimates with confidence flags [N]
- Context: who produced the report and why their estimates may be inflated

### Demand Signals
- Job postings in the category [N]
- Funding velocity — direction and what it means [N]
- Shutdowns and pivots — what failed companies signal about the market [N]

### Market Timing
- Emerging / growing / crowded / consolidating?
- Trend data with source [N]
- **Decision point:** Is the window open, closing, or not yet open?

## 5. Structural Shifts

Forces reshaping this segment — not trends, but paradigm changes.

For each shift:
- **The shift:** What's changing and why
- **Evidence:** Concrete signals [N]
- **Winners and losers**
- **Timeline:** Now / 1-2 years / 3-5 years
- **Implication:** Does this create opportunity, threaten the premise, or require repositioning?

## 6. Positioning Strategy

- The core tension (if any)
- Recommended positioning and why — grounded in user language from Section 2, not aspirational marketing
- One-liner
- Differentiation from nearest neighbors — which differentiators are real (evidence-backed) vs. claimed
- Value props ranked by: market need (from user research), defensibility, evidence strength

## 7. Pricing Recommendation

- Recommended price point(s) with rationale tied to competitive gaps in Section 3
- Pricing model with justification
- Tier structure if applicable
- Evidence basis: which data points support this price
- **What's NOT validated:** explicitly flag that willingness-to-pay is inferred, not tested. Recommend specific validation steps

## 8. What To Do Next

- Prioritized action list
- What to validate before committing (specific experiments: landing page tests, interview targets, pre-sale approaches)
- What to kill or deprioritize based on findings

---

## References

[1] Author. "Title." *Source.* Date. URL
[2] ...
```

### Writing Rules

**Audience: Decision makers who need the truth, not reassurance.**

- If the product is poorly positioned, say so. If a differentiator is weak, call it out.
- Surface hard truths in Section 1, not buried in Section 7.
- Every section answers "so what?" — not just describes the landscape.
- No filler, no hedging, no "it could be argued that."
- Challenge assumptions. In URL mode, compare website claims against user evidence.
- **Rank everything.** Don't present options without a recommendation. Take a position even when evidence is mixed — and state the risk of being wrong.
- **Distinguish claimed vs. evidenced.** When a finding comes from competitor marketing, say so. When it comes from user reviews, say so. Let the reader see the evidence quality.

### Evidence Standards

- Quantify: "42 comments," "337 upvotes," "$299-999/mo"
- Quote directly when user words are more powerful than summary
- Rank by frequency: pain points in 5 threads > pain points in 1
- Prefer negative/critical evidence — it's rarer and more informative than positive
- If a market size number comes from a report publisher that sells reports, flag it

## Browser Protocol Reference

**If an agent needs browser access**, it must read `.claude/skills/positioning-research/BROWSER_PROTOCOL.md` before using any browser tool.

## Self-Review Checklist

Before saving the final document:

- [ ] Section 1 ("The Case Against") is substantive — it would make an optimist uncomfortable
- [ ] Failed companies or skeptic arguments are cited, not just generic risks
- [ ] Market sizing uses bottom-up math, not just analyst report numbers
- [ ] Analyst estimates are flagged with confidence level and publisher context
- [ ] User quotes have direct URLs — no secondhand quotes without `[unverified]` marking
- [ ] Competitive landscape is built from user/third-party evidence, not competitor marketing copy
- [ ] Competitor taglines and hero copy are absent (or explicitly marked as competitor claims)
- [ ] G2 reviews are included — especially 2-3 star reviews
- [ ] Switching signals are captured with specific triggers
- [ ] Pricing recommendation exists with concrete price, evidence basis, and unvalidated flags
- [ ] Every major section ends with a decision point or implication
- [ ] Source credibility is annotated where it matters
- [ ] The document takes a clear position on positioning
- [ ] Risks are concrete, specific, and honest — not generic disclaimers
- [ ] Next steps include specific validation experiments, not just "do more research"
- [ ] All browser sessions are closed and artifacts cleaned up
- [ ] All files are in `$RESEARCH_DIR/`, nothing in project root
