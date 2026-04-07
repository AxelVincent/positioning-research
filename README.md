# Positioning Research

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that conducts deep positioning research for any product or idea. Give it a URL or a short product description and it produces an interconnected research wiki with competitive analysis, user research, pricing recommendations, and go-to-market strategy. The output is an [Obsidian](https://obsidian.md)-compatible vault — each research page uses YAML frontmatter, `[[wikilinks]]`, and tags, so you can navigate findings through Obsidian's graph view, backlinks panel, and search.

The research is oriented around **invalidation** — finding reasons the thesis is wrong, then seeing what survives.

Most positioning research presupposes the product should exist, which biases every search query and every quote toward confirmation. Invalidation flips that default: it prioritizes churned user language, failed competitors, and skeptic arguments over marketing copy and press releases. The findings that survive this scrutiny are the ones worth betting on — and if nothing survives, that's a valuable answer too.

## What It Does

This skill automates the kind of market research that typically takes a consultant weeks. It:

1. **Understands the product** — scrapes a website or works from your description
2. **Asks clarifying questions** — so it researches the right angles
3. **Runs parallel research agents** (on Sonnet for cost efficiency) — competitive landscape (third-party evidence only), user pain points & switching signals (Reddit, G2, Discord, Glassdoor), trends & structural shifts, market sizing & invalidation
4. **Asks before going deeper** — reviews Phase 2 findings with you and recommends targeted deep dives (1-2 agents) rather than automatically launching them
5. **Synthesizes everything** into an Obsidian wiki — interconnected pages with wikilinks, tags, and a Map of Content index. Leads with "The Case Against" to earn trust by surfacing hard truths before building the positive case

### Output Structure

Each research run creates an Obsidian-compatible wiki under `research/`:

```
research/{product-name}-{research-topic}/
├── index.md                    # Map of Content — the entry point
├── verdict.md                  # The Case Against + go/no-go decision
├── market-reality.md           # User pain points, switching signals, quotes
├── competitive-landscape.md    # Overview table, pricing landscape, positioning map
├── competitors/
│   ├── {competitor-slug}.md    # One page per major competitor (top 3-5)
│   └── ...
├── market-sizing.md            # Bottom-up + top-down sizing, demand signals
├── trends-and-shifts.md        # Structural shifts, timing, funding signals
├── positioning-strategy.md     # Positioning + pricing recommendation
├── next-steps.md               # Validation experiments, action items
├── sources.md                  # Centralized numbered reference list
└── deep-dive-*.md              # Targeted investigations (opt-in)
```

Every page uses YAML frontmatter (date, type, product, tags) and `[[wikilinks]]` to cross-reference other pages. Open the research folder in [Obsidian](https://obsidian.md) to get graph view, backlinks, and tag-based filtering across your findings.

The wiki covers:

- **The Case Against** (`verdict.md`) — why this might not work: failed companies, skeptic arguments, platform risk, data gaps, and a go/no-go verdict
- **Market Reality** (`market-reality.md`) — what users actually say, what they've tried, switching triggers, employee sentiment on competitors
- **Competitive Landscape** (`competitive-landscape.md` + `competitors/*.md`) — overview from user evidence (not marketing copy), with a dedicated page per major competitor
- **Market Sizing** (`market-sizing.md`) — bottom-up math (primary), top-down as supplement with confidence flags, demand signals
- **Trends & Structural Shifts** (`trends-and-shifts.md`) — forces reshaping the segment with evidence, winners/losers, timelines, market timing
- **Positioning Strategy** (`positioning-strategy.md`) — recommended positioning grounded in user language, pricing recommendation with evidence basis
- **Next Steps** (`next-steps.md`) — prioritized actions, specific validation experiments, what to kill

Every claim is backed by numbered citations that resolve in `sources.md`, annotated with credibility tags (`user review`, `competitor claim`, `analyst estimate`).

## Prerequisites

### Claude Code (required)

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) is Anthropic's CLI tool for Claude. It's what runs the skill, manages the parallel research agents, and orchestrates the entire workflow. Install it and make sure you have an active Anthropic API key configured.

### Browser automation (optional, but recommended)

Most research (~95%) uses WebSearch and WebFetch — built-in Claude Code tools that search Google and fetch web pages without a browser. This includes Reddit (via `site:reddit.com` search + `old.reddit.com` fetching), G2 reviews, Discord server discovery, LinkedIn/Greenhouse job postings, and Glassdoor/Blind employee sentiment. Browser automation is only needed as a fallback for JS-heavy sites and Google Trends.

You need **one** of the following:

#### Option A: Chrome + claude-in-chrome (preferred)

The [claude-in-chrome extension](https://chromewebstore.google.com/detail/claude-in-chrome/) connects Claude Code to your running Chrome browser via MCP (Model Context Protocol). Claude can open tabs, navigate pages, read content, click elements, and close tabs — all in your actual browser session.

**Setup:**
1. Install the extension from the Chrome Web Store
2. Open Chrome and make sure the extension is active (you should see its icon in the toolbar)
3. That's it — Claude Code auto-detects it when the skill runs

**Why preferred:** It uses your existing Chrome session with your cookies, logged-in accounts, and extensions. Some sites that block bots will work fine because you're using a real browser with a real profile.

#### Option B: playwright-cli (fallback)

[playwright-cli](https://github.com/anthropics/claude-code/tree/main/packages/playwright-cli) is a headless browser tool built for Claude Code. It runs a Chromium browser in the background — no visible window — and lets Claude navigate, take snapshots, and interact with pages.

**Setup:**
```bash
npm install -g @anthropic-ai/playwright-cli
# Verify it's installed:
which playwright-cli
```

**When this is used:** Only if chrome tools aren't available (extension not installed, Chrome not running). The skill automatically detects which tool is available and picks the right one.

#### No browser at all?

The skill still works without any browser automation. You'll miss Google Trends data and won't be able to read JS-heavy sites that block WebFetch, but the core research (competitors, Reddit, G2/Capterra, market sizing) will run fine.

## Cost

A full research run is **token-intensive**. The skill launches 6-7 parallel agents, each performing multiple web searches and page fetches, then synthesizes everything into a long-form document. Expect a single run to cost between **$5 and $15** depending on the depth of the topic and how many deep dives are triggered.

### How the skill manages costs

The skill is designed to minimize cost without sacrificing quality:

- **Three-tier model strategy.** Haiku subagents handle web page fetching and extraction (turning 80K-token pages into 500-token summaries). Sonnet agents run all parallel research (Phase 2 and Phase 3). Only the final synthesis uses Opus.
- **Haiku summarization layer.** No research agent reads raw web pages directly. Every WebFetch goes through a Haiku subagent that extracts only what's needed — this is the biggest cost savings.
- **WebSearch/WebFetch over browser automation.** Browser tools are expensive in tokens (screenshots, DOM snapshots, multi-step interactions). The skill uses lightweight WebSearch and WebFetch for ~95% of research — including Reddit (`site:reddit.com` + `old.reddit.com`) and G2 reviews. Browser is only used as a fallback.
- **Opt-in deep dives.** Phase 3 is no longer automatic — the skill reviews Phase 2 findings with you and only launches additional agents if you agree.
- **Parallel execution.** Running agents in parallel doesn't save tokens, but it cuts wall-clock time in half.

### Reducing costs

If cost is a concern:

- **Skip deep dives.** Phase 3 (deep dives) is now opt-in — just say "proceed to synthesis" when asked.
- **Narrow the scope.** A focused directive like "focus only on competitive pricing" will result in fewer searches and shorter reports.
- **Use Sonnet for everything.** You can switch Claude Code to use Sonnet (`/model sonnet`) before invoking the skill. The synthesis won't be as sharp, but it'll cost ~3-5x less.

## Installation

1. Clone this repository:

```bash
git clone https://github.com/YOUR_USERNAME/positioning-research.git
cd positioning-research
```

2. The skill lives in `.claude/skills/positioning-research/SKILL.md`. Claude Code automatically picks it up when you run it from the project directory.

3. Create the research output directory (if it doesn't exist):

```bash
mkdir -p research
```

## Usage

Start Claude Code from the project directory, then invoke the skill:

### Analyze an existing product by URL

```
/positioning-research https://linear.app
```

### Research a product idea from a description

```
/positioning-research A tool that helps sales teams automate follow-up emails based on CRM activity
```

### Add focus directives

```
/positioning-research https://notion.so focus on the project management angle for engineering teams
```

### What happens next

1. Claude scrapes the URL or reads your memo
2. It asks you 2-5 clarifying questions — answer these, they shape the research
3. It launches 4 parallel research agents (takes ~15-20 min)
4. It reviews findings with you and recommends targeted deep dives — you decide whether to go deeper or proceed
5. It synthesizes everything into an interconnected wiki, leading with "The Case Against"

### After the research

The wiki is a starting point, not the final word. Once the run completes, keep the conversation going — Claude still has all the research context loaded. This is where you get the most value:

- **Deep dive on a specific angle.** "Tell me more about how competitor X prices their enterprise tier" or "What exactly are users saying about onboarding friction?"
- **Cross-check references.** Ask Claude to re-read a specific source and verify a claim. Citations can drift during synthesis — spot-checking keeps the document honest.
- **Challenge the recommendations.** "Why did you rank this value prop higher than that one?" — forcing Claude to defend its reasoning often surfaces nuance the document missed.
- **Explore adjacent questions.** "What would the positioning look like if we targeted SMBs instead of enterprise?" or "How does this change if we go freemium?"
- **Request additional research.** "Can you look into the regulatory landscape for this market?" or "Find more Reddit threads about migration pain from competitor Y."

Open the research folder in Obsidian to browse findings visually — the graph view shows how competitors, market signals, and positioning recommendations connect, and the backlinks panel surfaces relationships you might miss reading linearly.

## Configuration

The skill requires several tool permissions (web search, web fetch, browser automation, file I/O) that Claude Code will prompt you to approve during the run. To skip these prompts and let the skill run fully autonomously, rename the example settings file:

```bash
cp .claude/settings.local.json.example .claude/settings.local.json
```

This pre-approves all the tools the skill needs. Review the file before applying — it grants broad permissions including shell access, web fetching, and browser automation.

## Tips

- **Be specific in your memo.** The more context you give upfront, the better the research. Include target audience, known competitors, and what angle matters most to you.
- **Answer the clarifying questions thoughtfully.** These directly shape which subreddits are searched, which competitors are investigated, and which angles are prioritized.
- **Don't be alarmed by "The Case Against."** The document leads with reasons the thesis might be wrong — this is by design. It earns trust by surfacing hard truths before building the positive case. The findings that survive scrutiny are the ones worth betting on.
- **Browser tools are a fallback.** The skill uses WebSearch and WebFetch for ~95% of research. Browser automation (Chrome or playwright-cli) is only used when a page blocks static fetching.
- **Research takes time.** A full run takes 30-40 minutes. The parallel agents do the heavy lifting.
- **Open the wiki in Obsidian.** Point Obsidian at the research folder to get graph view, backlinks, and tag filtering. The wiki is designed for non-linear exploration — follow the links that matter to you.

## Example Output

Here's what a research wiki looks like after a complete run:

```
research/ai-qa-testing-positioning/
├── index.md                       (2 KB)   ← start here
├── verdict.md                     (8 KB)
├── market-reality.md              (12 KB)
├── competitive-landscape.md       (10 KB)
├── competitors/
│   ├── cypress.md                 (6 KB)
│   ├── playwright.md              (6 KB)
│   └── selenium.md                (5 KB)
├── market-sizing.md               (8 KB)
├── trends-and-shifts.md           (7 KB)
├── positioning-strategy.md        (10 KB)
├── next-steps.md                  (4 KB)
├── deep-dive-top-competitor.md    (12 KB)
└── sources.md                     (6 KB)
```

The wiki totals 80-100 KB across all pages — a thorough, opinionated analysis that leads with invalidation and backs every claim with cited evidence. Open it in Obsidian for the best reading experience.

## License

MIT
