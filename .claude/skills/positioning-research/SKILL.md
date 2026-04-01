---
name: positioning-research
description: Conduct deep market and positioning research for any product or idea. Works in two modes — give it a URL to analyze an existing product's website, or give it a short memo describing your idea. Uses WebSearch and WebFetch for most research (including Reddit via site:reddit.com + old.reddit.com, and G2/Capterra reviews). Browser tools (Chrome or playwright-cli) used only as fallback. Asks clarifying questions before diving in. Produces a comprehensive positioning document with concrete pricing recommendations. Use this whenever the user mentions market research, positioning, competitive analysis, or wants to understand how a product fits in its market.
argument-hint: "[URL or short product memo, e.g. 'https://linear.app' or 'A tool that helps sales teams automate follow-up emails based on CRM activity']"
---

# Positioning & Market Research

You are conducting positioning and market research. The user provided: **$ARGUMENTS**

## Step 0 — Create an Isolated Research Workspace

Before doing anything else, create a workspace folder for this research run inside the `research/` directory. The folder name should be auto-generated from the product name and research topic, using lowercase kebab-case. It should be human-readable and explicit — someone browsing the folder should immediately understand what the research is about.

**Naming convention:** `{product-name}-{research-topic}`

Examples:
- `linear-positioning`
- `stripe-competitive-analysis`
- `notion-market-research`
- `acme-saas-positioning`

```bash
# Derive RESEARCH_DIR from the product name and research topic
RESEARCH_DIR="research/{product-name}-{research-topic}"
mkdir -p "$RESEARCH_DIR"

# Also create a unique run ID for playwright session scoping
RUN_ID="research-$(date +%Y%m%d-%H%M%S)"
```

All playwright-cli sessions for this run must use session names prefixed with the run ID (e.g., `-s=${RUN_ID}-reddit`). **All files — intermediate reports, agent outputs, and the final synthesis — go into `$RESEARCH_DIR/`.** Nothing should be written to the project root.

## Step 1 — Determine Input Mode

Look at `$ARGUMENTS` to determine which mode to use:

- **URL mode** — the input contains a URL (starts with `http://` or `https://`). You'll scrape the website to understand the product.
- **Memo mode** — the input is a text description, brief, or set of bullet points. You'll work from that description.

The rest of the input (beyond the URL or memo) may contain focus directives like "focus on the developer tooling angle" — carry these through the entire research.

## Step 2 — Understand the Product

### URL Mode: Scrape the Website

Use WebFetch to read the product's main page first. If WebFetch returns meaningful content, continue with it for subpages. If it fails (empty, 403, JS-only), fall back to the browser tool.

**Step 1 — Read the main page** via WebFetch (or browser as fallback).

**Step 2 — Discover subpages from the actual content.** Don't guess paths like `/pricing` or `/features` — they often don't exist. Instead, look at the navigation links, footer links, and CTAs present in the page content you just fetched. Follow the ones that look most useful for understanding the product (pricing, features, about, blog, docs, use cases, etc.). Only visit pages that actually exist on the site.

**Step 3 — Read the most promising subpages** (typically 2-5, depending on what's available).

If using the browser for discovery, close the tab/session when done.

### Memo Mode: Work from the Description

Read the memo carefully. Extract everything you can about:
- What the product does
- Who it's for
- How it works
- What makes it different

### Both Modes: Produce an Internal Summary

After gathering initial information, produce a short internal summary:
- What the product does (1 paragraph)
- Target audience
- Core value propositions (3-5 bullets)
- Key differentiators
- Pricing model (if known)
- Key category terms to research

## Step 3 — Ask Clarifying Questions

Before launching into research, pause and ask the user 2-5 targeted questions to fill gaps in your understanding. The goal is to avoid wasting research time on wrong assumptions.

Good questions to consider (pick the ones that matter most given what you already know):

- **Target audience**: "Who do you see as the primary buyer — is this for individual practitioners, team leads, or a company-wide purchase?"
- **Competitive awareness**: "Are there specific competitors you already know about or are watching?"
- **Differentiation**: "What do you think is the hardest thing for a competitor to replicate about this?"
- **Stage**: "Is this an existing product you're repositioning, or a new idea you're validating?"
- **Market category**: "If someone Googled for this product, what would they search? What category do you think this fits in?"
- **Priorities**: "What's most important to you from this research — competitive intel, user language, positioning angle, keyword strategy, or something else?"

Keep it conversational. Don't dump all questions at once — ask the 2-5 most important ones based on what's missing. If the user's input was already very detailed, you might only need 1-2 clarifications.

**Wait for the user's answers before proceeding to Phase 2.** Their responses will shape which subreddits to search, which competitors to investigate, and which angles to prioritize.

## Prerequisites

- **Chrome browser with claude-in-chrome extension** (preferred) — provides browser automation via MCP tools
- **playwright-cli** (fallback) — must be installed and available in PATH (`which playwright-cli`). Used only when claude-in-chrome is not available.

## Browser Tool Selection

At the start of the run, determine which browser tool is available:

1. **Check for claude-in-chrome first:** Try calling `mcp__claude-in-chrome__tabs_context_mcp`. If it succeeds, use chrome tools for all browser tasks.
2. **Fall back to playwright-cli:** If chrome tools are not available (MCP not connected), use playwright-cli instead.

Set a variable to track which browser backend to use, and pass it to all agents that need browser access.

## Research Tool Selection

Use the lightest tool that gets the job done:

| Task | Tool | Why |
|------|------|-----|
| Search queries (competitors, categories, alternatives, HN) | **WebSearch** | Fast, no browser needed |
| Read competitor pages, blogs, articles, docs | **WebFetch** | Grabs page content directly |
| Reddit thread discovery | **WebSearch** (`site:reddit.com`) | Finds threads reliably via Google index |
| Reddit thread reading | **WebFetch** (`old.reddit.com`) | old.reddit.com is scrapable; new Reddit blocks |
| Reddit thread reading (if WebFetch fails) | **Chrome browser** (or playwright-cli) | Escalation only — browser hits CAPTCHAs frequently |
| G2/Capterra competitor reviews | **WebSearch** + **WebFetch** | Competitor reviews have verified user language |
| Google Trends | **Chrome browser** first, then proxy sources | Try browser; if 429/CAPTCHA, use explodingtopics.com or trend articles |
| Any site that WebFetch fails on (403, empty, JS-only) | **Chrome browser** (or playwright-cli fallback) | Last resort when static fetch doesn't work |

**Key lesson from past runs:** Reddit browser automation and Google Trends fail ~90% of the time. WebSearch `site:reddit.com` + WebFetch `old.reddit.com` is far more reliable for Reddit. For Trends, proxy sources (explodingtopics.com, trend articles, funding signals) provide equivalent data.

## Phase 2 — Parallel Research (15-20 min)

Launch **4 parallel agents** using `model: "sonnet"`. These agents do search-and-summarize work that doesn't need Opus — save the expensive model for the final synthesis. All agents use WebSearch/WebFetch as primary tools. Agents 2 and 3 may escalate to browser only if WebFetch fails on specific URLs.

### Agent 1: Competitive Landscape & Pricing (WebSearch + WebFetch)

**Step 1 — Discover competitors via WebSearch:**
- `"[category]" platform`
- `"[category]" tool`
- `[category] alternative`
- `best [category] tools [current year]`

**Step 2 — Read each competitor's homepage and pricing via WebFetch:**
For each direct and adjacent competitor:
- Capture the H1/hero tagline
- Note their positioning category
- Identify their target audience
- **Read their pricing page** — capture tiers, price points, what's included at each level, free trial availability

If WebFetch returns empty or blocked content for a specific site, note it and move on.

**Step 3 — Check who ranks for category terms via WebSearch:**
- Search the core category terms and note the top organic results

**Step 4 — Pricing strategy analysis:**
- Map the pricing landscape: who's cheap, who's premium, where the gaps are
- Identify pricing models (per-seat, usage-based, flat, freemium)
- Note any signals about willingness-to-pay (pricing page testimonials, case studies mentioning ROI)
- Look for Crunchbase/funding data that hints at revenue scale

Save report to `$RESEARCH_DIR/competitive-landscape.md`: competitor table (name, tagline, category, target, pricing, URL), pricing landscape analysis, positioning gaps, white space.

### Agent 2: User Pain Points & Switching Signals — Reddit + Review Sites

This agent gathers authentic user language from Reddit, G2, Capterra, and community forums. Past runs show that Reddit browser automation fails ~90% of the time (CAPTCHAs, rate limits). Use a multi-layered approach instead of relying on browser access alone.

**Layer 1 — Reddit via WebSearch + WebFetch (primary, most reliable):**

Reddit threads are indexed by Google. Use WebSearch to find them, then WebFetch to read individual threads (old.reddit.com works better than new Reddit for scraping):

```
# Find threads via Google
WebSearch: site:reddit.com "[category]" pain OR frustration OR switched OR alternative
WebSearch: site:reddit.com "[competitor name]" review OR experience OR switched from
WebSearch: site:reddit.com r/[relevant_subreddit] "[search term]"

# Read individual threads — use old.reddit.com for better scraping
WebFetch: https://old.reddit.com/r/SUBREDDIT/comments/THREAD_ID/...
```

This approach reliably returns thread content with comments. Run 8-12 searches across 3-5 subreddits.

**Layer 2 — Browser for Reddit (escalation only):**

Only use browser if WebFetch returns empty/blocked for specific threads you've already identified via WebSearch. Don't start with browser — it's slow and frequently hits CAPTCHAs.

Using Chrome:
```
mcp__claude-in-chrome__tabs_create_mcp → get tabId
mcp__claude-in-chrome__navigate(tabId, "https://www.reddit.com/r/SUBREDDIT/comments/...")
mcp__claude-in-chrome__read_page(tabId)
mcp__claude-in-chrome__javascript_tool(tabId, "window.close()")
```

Using playwright-cli:
```
playwright-cli -s=${RUN_ID}-reddit open "https://old.reddit.com/r/..."
sleep 4
playwright-cli -s=${RUN_ID}-reddit snapshot
playwright-cli -s=${RUN_ID}-reddit close
```

**Layer 3 — G2 and Capterra reviews (required, not optional):**

Competitor reviews on G2/Capterra contain high-quality user language with verified buyers. This is often more reliable than Reddit.

```
WebSearch: site:g2.com "[competitor name]" reviews
WebSearch: site:capterra.com "[competitor name]" reviews
WebSearch: "[competitor name]" review "switched from" OR "moved to" OR "pros and cons"
```

Read the top 2-3 review pages per major competitor via WebFetch.

**Layer 4 — Community forums and HN:**

```
WebSearch: site:news.ycombinator.com "[category]" OR "[competitor]"
WebSearch: "[category]" forum OR community discussion pain OR frustration
```

**What to search for across all layers:**
- How users describe the problem the product solves
- What language they use (not what the product calls it)
- What tools they mention and what they've tried
- What frustrations they express
- Whether the product's terminology matches user language
- **Switching signals** — explicitly search for "switched from", "moved to", "left [competitor]", "alternative to". Why do users leave their current tool? What triggers the switch? What do they wish they had?

**Citation rule: Every quote MUST have a direct URL to the source thread/review.** Do not include quotes sourced from aggregator articles or secondhand summaries. If you cannot link to the original thread, mark the quote as `[unverified — secondhand source]` and note where you found it.

**When done — close any browser tabs/sessions immediately.**

Save report to `$RESEARCH_DIR/reddit-pain-points.md`: user language, pain points, switching triggers, tools mentioned, sentiment, key quotes with direct thread/review URLs.

### Agent 3: Trend Validation (WebSearch primary, browser for Trends if available)

Past runs show Google Trends browser automation fails frequently (429 rate limits, JS rendering issues). Use a multi-source approach so trend data doesn't depend on a single tool.

**Google Trends — try browser first, fall back to proxies:**

Attempt 1 — Browser (may fail):
Using Chrome:
```
mcp__claude-in-chrome__tabs_create_mcp → get tabId
mcp__claude-in-chrome__navigate(tabId, "https://trends.google.com/trends/explore?q=TERM1,TERM2,TERM3&hl=en")
mcp__claude-in-chrome__read_page(tabId)
mcp__claude-in-chrome__javascript_tool(tabId, "window.close()")
```

Using playwright-cli:
```
playwright-cli -s=${RUN_ID}-trends open "https://trends.google.com/trends/explore?q=TERM1,TERM2,TERM3&hl=en"
sleep 6
playwright-cli -s=${RUN_ID}-trends snapshot
playwright-cli -s=${RUN_ID}-trends close
```

Attempt 2 — If browser fails (CAPTCHA, 429, empty), use proxy sources:
```
WebSearch: "google trends" "[category term]" growth OR rising OR declining
WebSearch: site:explodingtopics.com "[category term]"
WebSearch: "[category term]" search volume OR trend OR growth [current year]
```

These proxy sources frequently report Google Trends data in their articles. Cite the proxy source, not Google Trends directly.

**Funding signals (strong trend proxy):**
```
WebSearch: "[category]" funding OR raised OR series [current year]
WebSearch: site:crunchbase.com "[category]" funding rounds
WebSearch: "[category]" acquisitions [current year]
```

Funding velocity is often a better trend signal than search volume — VCs do their own market timing analysis.

**Thought leadership and HN:**
- WebSearch: `"[category term]" blog OR article [current year]`
- WebSearch: `site:news.ycombinator.com "[category term]"`
- WebFetch on the most relevant articles to read them

**Analyst coverage:**
```
WebSearch: "[category]" Gartner OR Forrester OR "magic quadrant" OR "market guide" [current year]
```

Analyst recognition signals market maturity — if Gartner has a Magic Quadrant for the category, it's established; if they just published a Market Guide, it's emerging.

**Structural shifts (critical — this is what separates good research from great research):**

Look for forces that are fundamentally reshaping this segment — not just "the market is growing" but "the rules of the game are changing." Examples: AI commoditizing code, no-code eating developer tools, regulation reshaping fintech, remote work killing office software.

```
WebSearch: "[category]" disrupted OR commoditized OR "paradigm shift" OR "future of" [current year]
WebSearch: "[category]" AI impact OR automation OR replaced by [current year]
WebSearch: "[category]" "no longer need" OR obsolete OR "changed everything" [current year]
WebSearch: "[adjacent technology]" replacing OR eliminating "[category]"
```

For each shift identified, capture:
- **What's changing:** the specific force or technology
- **Who's affected:** which players/segments win or lose
- **Timeline:** is this happening now, in 1-2 years, or 3-5 years?
- **Evidence:** concrete signals (product launches, funding shifts, user behavior changes, HN discourse)
- **Implication for the product:** does this shift help, hurt, or reshape the opportunity?

Look beyond the obvious. Check:
- VC thesis papers and partner blog posts about the category (a16z, Sequoia, First Round — they often identify shifts early)
- "Year in review" or "predictions" articles from category leaders
- HN threads where practitioners debate whether the category still makes sense
- Competitor pivots or shutdowns (a competitor pivoting away from the category IS a signal)

Save report to `$RESEARCH_DIR/trends-validation.md`: trend direction (with source — Google Trends direct, proxy, or funding signals), relative search volumes (if available), thought leadership landscape, discourse maturity, analyst coverage level, **structural shifts with evidence and timeline**.

### Agent 4: Market Sizing & GTM Intelligence (WebSearch + WebFetch)

**Step 1 — Market sizing (TAM/SAM/SOM):**
- WebSearch: `"[category]" market size [current year]`
- WebSearch: `"[category]" market report OR forecast`
- WebSearch: `"[category]" TAM OR "total addressable market"`
- Look for industry reports (Gartner, Forrester, Grand View Research, Statista, etc.), even preview snippets are valuable
- Search for the number of potential users/buyers in the target segment (e.g., "how many coaches in the US", "number of SaaS companies")
- Check competitor funding rounds on Crunchbase/TechCrunch — funded competitors signal a validated market; their valuations hint at perceived market size

**Step 2 — Go-to-market channel analysis:**
- WebSearch: `"[competitor name]" traffic OR marketing OR growth`
- Check SimilarWeb/SEMrush data if publicly accessible
- Look for competitor content strategies: do they blog, run a podcast, have a YouTube channel, do paid ads?
- WebSearch: `"[competitor name]" review OR testimonial` — where do users discover these tools?
- Check if competitors are on Product Hunt, AppSumo, marketplaces, or partnerships
- Look for affiliate programs, referral programs, integration directories

**Step 3 — Platform risk scan:**
- WebSearch: `"[big player]" [category feature]` for relevant big players (OpenAI, Google, Salesforce, HubSpot, etc. — choose based on category)
- Check if any major platform has announced or shipped features that overlap with this category
- Look for acquisitions in the space — who's buying whom?

Save report to `$RESEARCH_DIR/market-sizing-gtm.md`: TAM/SAM/SOM estimates with sources, GTM channels used by competitors, platform risk assessment.

## Phase 3 — Deep Dives (15-20 min)

Based on Phase 2 findings, launch **2-3 more parallel agents** using `model: "sonnet"` for deeper investigation into the most promising angles. Choose from:

- **Deep-dive into the #1 competitor** — WebFetch their product page, pricing, blog content, handbook (if public). Understand their GTM motion and what's working for them.
- **Deep-dive into user pain and switching** — more Reddit threads (via WebSearch `site:reddit.com` + WebFetch `old.reddit.com`), G2 reviews, specific workflows, horror stories, "I switched from X" threads
- **Deep-dive into adjacent market** — WebSearch + WebFetch for tools in the neighboring category
- **Deep-dive into emerging tools and platform moves** — WebSearch for Show HN, ProductHunt, recent startups, and announcements from big players entering the space

### Deep Dive Quality Criteria

Each deep-dive report must meet these minimum standards (past runs show wide quality variance when these aren't enforced):

1. **Minimum 3,000 words** — a deep dive that's shorter isn't deep
2. **At least 10 numbered citations with URLs** — no unsourced claims
3. **Pricing data required** if the deep dive covers a competitor (tiers, price points, model type)
4. **User sentiment required** if the deep dive covers user pain (direct quotes from Reddit/G2/HN with URLs)
5. **At least 2 decision points** — each deep dive must answer "so what does this mean for the product?"
6. **Honest uncertainty** — if data is sparse, say so explicitly rather than speculating

Follow the same tool selection rule: WebSearch/WebFetch first, browser only when needed. Each browser agent **must close its tab/session before returning results**. If using playwright-cli, each agent gets its own unique session name prefixed with `${RUN_ID}`.

Save each deep-dive report to `$RESEARCH_DIR/` with a descriptive filename (e.g., `deep-dive-miria-competitor.md`, `deep-dive-coaching-pain-points.md`).

## Phase 4 — Keyword Research (optional, if user can run Google Keyword Planner)

Generate a comma-separated list of 50-60 keywords for the user to paste into Google Keyword Planner. Organize by:

1. **Core product keywords** — what the product is
2. **Problem-aware keywords** — what users search when feeling the pain
3. **Competitor keywords** — competitor names + "alternative"
4. **Feature keywords** — specific capabilities
5. **Trend keywords** — emerging category terms

When the user pastes back the Keyword Planner results, analyze:
- Which terms have volume and are trending
- Which terms have high intent (high CPC = commercial intent)
- Which terms are dead ends (zero volume)
- Recommend a tiered keyword strategy

## Phase 5 — Synthesize into Document

Write the final synthesis document to `$RESEARCH_DIR/positioning-research.md`.

### Citation System

Use numbered references throughout the document, like a research paper. This makes every claim verifiable.

- **Inline citations:** Use numbered brackets — `[1]`, `[2]`, `[3]` — immediately after the claim they support.
- **Multiple sources for one claim:** `[1][2]` or `[1, 2]`.
- **Quotes:** Always followed by their reference number.
- **The References section** at the end lists every source with its full URL, title, author (if known), and date (if known).

Example: *"The AI coaching market is projected to reach $4.2B by 2028 [3], though current adoption remains concentrated in enterprise L&D departments [7][12]."*

Number references sequentially as they first appear in the document.

### Document Structure

```markdown
# [Product Name] — Market Research

**Date:** [today]
**Input:** [URL / Memo]

---

## 1. Executive Summary

The verdict — in 5-10 bullet points, a CEO who reads only this section should be able to make a go/no-go decision. Include:
- Market size and growth direction (one line)
- **Structural shifts:** the 1-2 forces reshaping this segment right now and whether they help or hurt
- Competitive density: how crowded is this, and who's most dangerous
- The core differentiator — and whether it's real or claimed
- The single biggest risk
- The recommended positioning (one sentence)
- The #1 thing to do next

## 2. Market Opportunity

### Market Size
- TAM / SAM / SOM estimates with sources [N]
- How these numbers were derived — be transparent about confidence level
- If hard data doesn't exist, say so and provide proxy estimates

### Market Timing
- Is this market emerging, growing, crowded, or consolidating?
- Google Trends data for key category terms [N]
- Funding activity in the space — are investors pouring in or pulling back? [N]
- **Decision point:** Is the window open, closing, or not yet open?

## 3. Competitive Landscape

### Competitor Overview
- Competitor table (name, tagline, category, target audience, pricing, URL, relationship to product)
- Visual competitive map showing positioning clusters and white space

### Pricing Landscape
- Price range across the category (lowest to highest)
- Pricing model comparison (per-seat, usage, flat, freemium)
- Where the pricing gaps are — is there an underserved price tier?
- What pricing signals about willingness-to-pay in this market

### Most Dangerous Competitors
- Deep dive on the 1-2 competitors that matter most
- What they do well, where they're weak, and what they're likely to do next
- **Decision point:** Can you win against them, and on what axis?

## 4. User Research

### How Users Describe the Problem
- Their language vs. the product's language — do they match?
- Pain points ranked by frequency and intensity [N]
- Key quotes with source [N]

### What Users Have Tried
- Tools and workarounds users mention
- What's working for them, what's not

### Switching Signals
- Why users leave their current tool — the specific triggers [N]
- What they look for in an alternative
- **Decision point:** What's the switching trigger you can own?

## 5. Go-to-Market Landscape

### How Competitors Acquire Users
- Channel breakdown: organic, paid, community, partnerships, marketplaces
- What's working — which competitors are growing and how [N]
- Content and thought leadership: who owns the conversation

### Available Channels
- Underexploited channels competitors aren't using well
- Partnership and integration opportunities
- **Decision point:** What's the most capital-efficient GTM path?

## 6. Value Proposition Analysis
- Ranking table: each value prop scored on market need, differentiation, evidence strength
- Which props are validated by user research vs. only claimed
- Which props are truly defensible vs. easily copied
- **Decision point:** Which 1-2 props should lead your messaging?

## 7. Positioning Strategy
- The core tension (if any — e.g., keyword momentum vs. audience fit)
- Recommended positioning and why
- One-liner
- Differentiation from nearest neighbors
- Messaging tone (augment vs. replace, etc.)

## 8. Structural Shifts

This section identifies forces that are fundamentally reshaping the segment — not incremental trends, but paradigm changes that alter who wins and how.

For each shift:
- **The shift:** What's changing and why (1-2 sentences)
- **Evidence:** Concrete signals — product launches, funding patterns, user behavior, practitioner discourse [N]
- **Winners and losers:** Which players/segments benefit, which are threatened
- **Timeline:** Happening now / 1-2 years / 3-5 years
- **Implication for the product:** Does this create opportunity, threaten the premise, or require repositioning?
- **Decision point:** What should the product do about this shift?

Examples of structural shifts to look for:
- Technology commoditization (e.g., AI making code/design/content near-free)
- Buyer behavior changes (e.g., self-serve replacing enterprise sales)
- Category convergence (e.g., two adjacent categories merging into one)
- Regulatory changes that reshape the playing field
- Platform shifts (e.g., mobile-first, API-first, agent-first)
- Talent/skill commoditization (e.g., no-code enabling non-technical founders)

Be specific and evidence-based. "AI will change everything" is not a structural shift analysis. "GPT-4o's March 2026 code agent scored 87% on SWE-bench, up from 33% a year ago, which is pushing [category] tools from 'nice to have' to 'existential' for their users" is.

## 9. Risks & Threats

### Platform Risk
- Which big players could enter this space or add this feature? [N]
- Recent acquisitions or announcements that signal intent [N]
- How defensible is the product if a major platform moves in?

### Market Risks
- Where the product's claims diverge from market reality
- Positioning vulnerabilities competitors could exploit
- Timing risks — too early, too late, or wrong market conditions

### Blind Spots
- What this research couldn't answer — data gaps, unknowns
- Areas that need validation before committing

## 10. Keyword & Search Strategy
- Tiered keyword table (awareness, intent, niche, competitor)
- Search strategy implications
- Which terms have real volume and commercial intent

## 11. Pricing Recommendation

Past research runs consistently identify pricing gaps but stop short of a concrete recommendation. This section must go further:

- **Recommended price point(s)** with rationale tied to competitive gaps identified in Section 3
- **Pricing model** (per-seat, usage-based, flat, freemium) with justification
- **Tier structure** if applicable (free, starter, pro, enterprise)
- **Evidence basis** — which data points support this price (competitor benchmarks, willingness-to-pay signals from user research, funding-implied revenue targets)
- **What's NOT validated** — explicitly flag that willingness-to-pay is inferred from competitor pricing and user sentiment, not from primary research (conjoint analysis, pricing surveys). Recommend specific validation steps (e.g., "test $X vs $Y on landing page", "run 10 customer development calls focused on pricing")
- **Decision point:** What price to launch at and what triggers a price change

## 12. Next Steps
- Prioritized action list — what to do first, second, third
- Quick wins vs. strategic bets
- What to validate before scaling

---

## References

[1] Author (if known). "Title." *Source/Publication.* Date (if known). URL
[2] ...
[3] ...
```

## Browser Protocol Reference

This section applies to agents that need browser interaction (Reddit, Google Trends, fallback).

### CRITICAL: Browser Tab/Session Lifecycle

Orphaned Chrome windows/tabs are the #1 operational problem with this skill. Every browser tab or session that opens MUST be closed — no exceptions, even if the agent errors out or times out.

### Chrome (claude-in-chrome) — Preferred

**Opening and navigating:**
```
mcp__claude-in-chrome__tabs_create_mcp → returns tabId
mcp__claude-in-chrome__navigate(tabId, url)
mcp__claude-in-chrome__read_page(tabId)  # get page content
```

**Clicking elements:**
```
mcp__claude-in-chrome__read_page(tabId)  # find coordinates of element
mcp__claude-in-chrome__computer(tabId, action: "click", x, y)
```

**Scrolling:**
```
mcp__claude-in-chrome__javascript_tool(tabId, "window.scrollBy(0, 800)")
mcp__claude-in-chrome__read_page(tabId)
```

**Closing when done:**
```
mcp__claude-in-chrome__javascript_tool(tabId, "window.close()")
```

### Playwright-cli — Fallback

Only use when chrome tools are unavailable.

**Rules:**
- **Never use `--headed`.** Always run headless.
- Each agent MUST use a session name prefixed with the run ID (`-s=${RUN_ID}-reddit`, `-s=${RUN_ID}-trends`, etc.)
- **Close the session as soon as you're done reading.**
- **Each agent closes its own session before returning results.** The last thing an agent does is `playwright-cli -s=SESSION_NAME close`.
- Always `sleep 4` after navigation before taking a snapshot.

### Obstacle Handling (both tools)

After every navigation, handle obstacles:

1. **Cookie / consent banners:** Click "Accept", "Accept all", "I agree", "Got it".
2. **CAPTCHA / "Verify you are human":** Attempt to solve. If unsolvable after 2 attempts, close and skip the URL.
3. **Login walls / paywalls:** Do NOT log in. Note the wall and move on.
4. **Age gates / region selectors:** Click through with any reasonable option.
5. **Newsletter popups / modal overlays:** Click "X", "Close", or "No thanks".
6. **Redirect loops / blank pages:** Wait and retry once. If still empty, close and skip.

### Browser Cleanup (mandatory — parent orchestrator)

After ALL agents for this run have completed, the parent orchestrator MUST run cleanup.

**If using playwright-cli:**
```bash
# Force-close all sessions for this run
for session_name in reddit trends discovery reddit-deep fallback; do
  playwright-cli -s="${RUN_ID}-${session_name}" close 2>/dev/null || true
done

for session in $(ls .playwright-cli/${RUN_ID}-* 2>/dev/null); do
  playwright-cli -s="$(basename "$session")" close 2>/dev/null || true
done

mkdir -p "$RESEARCH_DIR/browser-data" 2>/dev/null
mv .playwright-cli/${RUN_ID}-* "$RESEARCH_DIR/browser-data/" 2>/dev/null || true
rmdir .playwright-cli/ 2>/dev/null || true
```

**If using Chrome:** Tabs should already be closed by each agent. No additional cleanup needed since Chrome manages its own process.

## Writing Rules

### Audience: C-level Decision Makers

This document will be read by founders, CEOs, and executives who need to make strategic decisions. Write accordingly:

- **Be brutally honest.** If the product is poorly positioned, say so. If a claimed differentiator is weak, call it out. The reader needs the truth, not reassurance. Sugarcoating wastes their time and leads to bad decisions.
- **Surface hard truths early.** Don't bury bad news in section 7. If the market is crowded, the differentiation is thin, or the timing is wrong, lead with that.
- **Make every section decision-ready.** Each section should answer "so what?" and "what do I do about this?" — not just describe the landscape.
- **Be direct and concise.** No filler, no hedging, no "it could be argued that." State the finding, back it with evidence, give the implication.
- **Challenge assumptions.** If the product's positioning contradicts what the market evidence shows, say so explicitly. In URL mode, compare what the website claims against what users actually say and competitors actually do.

### Be Opinionated

Rank everything. Don't present options without a recommendation. The document should say "do this, not that" — not "here are some things to consider."

When the evidence is mixed, say so — but still take a position and explain the risk of being wrong.

### Sources and Attribution

Use the numbered citation system described in Phase 5. Every factual claim must have a reference number. In the References section at the end, format each source as:
- Reddit threads: `[N] "Thread title." r/SubredditName. Date. URL` — include upvote count in the inline text
- Competitor sites: `[N] "Page title." CompanyName. URL`
- Blog posts / articles: `[N] Author. "Title." Publication. Date. URL`
- Industry reports: `[N] "Report title." Publisher. Date. URL`
- Crunchbase / funding data: `[N] "Company — Crunchbase." URL`

### Evidence Density

- Quantify: "42 comments," "337 upvotes," "$299-999/mo"
- Quote directly when the user's words are more powerful than a summary
- Rank by frequency: pain points that appear in 5 threads matter more than those in 1

### Structure

- Lead every section with its conclusion — the "so what"
- Use tables for comparisons
- Use code blocks for visual maps (competitive landscape diagrams)
- Keep paragraphs under 3 sentences
- End each major section with a clear **Implication** or **Decision point** — what this means for the product strategy

## Self-Review Checklist

Before saving the final document:

- [ ] Executive summary delivers the full verdict — a CEO who reads only this section can make a go/no-go decision
- [ ] Hard truths and weaknesses are surfaced, not buried — the document doesn't read like a pitch deck
- [ ] Market sizing section exists with TAM/SAM/SOM estimates (even rough ones) and honest confidence levels
- [ ] Pricing landscape is analyzed, not just listed
- [ ] **Pricing recommendation section exists** with concrete price point, tier structure, evidence basis, and explicit flags for what's unvalidated
- [ ] Switching signals are captured — why users leave their current tool
- [ ] GTM channels are mapped with a recommendation on the most efficient path
- [ ] Platform risk is assessed — which big players could enter and kill this
- [ ] Every competitor entry has a link to their homepage
- [ ] Every factual claim has a numbered citation [N]
- [ ] **Every user quote has a direct URL** to the source thread/review — no secondhand quotes without explicit `[unverified]` marking
- [ ] References section at the end lists every source with full URL, title, author, and date
- [ ] **G2/Capterra reviews are included** in user research — not just Reddit
- [ ] Value props are ranked, not just listed — and weak ones are called out as weak
- [ ] The document takes a clear, defensible position on positioning
- [ ] Keyword strategy is tiered with rationale
- [ ] Risks are concrete, specific, and honest — not generic disclaimers
- [ ] Every major section ends with a **Decision point** or **Implication**
- [ ] **Deep dive reports each meet quality criteria** (3,000+ words, 10+ citations, pricing data if competitor-focused, user sentiment if pain-focused)
- [ ] Next steps are actionable and prioritized
- [ ] All browser tabs (Chrome) or playwright-cli sessions for this run are closed and artifacts cleaned up
- [ ] All intermediate reports (competitive-landscape.md, reddit-pain-points.md, trends-validation.md, market-sizing-gtm.md, deep-dives) are saved in `$RESEARCH_DIR/`
- [ ] Final synthesis is written to `$RESEARCH_DIR/positioning-research.md`
- [ ] Nothing was written to the project root — everything is inside `research/{name}/`
