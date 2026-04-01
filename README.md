# Positioning Research

A Claude Code skill that conducts deep positioning research for any product or idea. Give it a URL or a short product description and it produces a comprehensive, C-level-ready positioning document with competitive analysis, user research, pricing recommendations, and go-to-market strategy.

## What It Does

This skill automates the kind of market research that typically takes a consultant weeks. It:

1. **Understands the product** — scrapes a website or works from your description
2. **Asks clarifying questions** — so it researches the right angles
3. **Runs parallel research agents** (on Sonnet for cost efficiency) — competitive landscape, user pain points (Reddit, G2, Capterra), trend validation & structural shifts, market sizing & GTM intelligence
4. **Performs deep dives** — 2-3 additional agents investigate the most promising angles
5. **Synthesizes everything** into a single positioning document with numbered citations

### Output Structure

Each research run creates a folder under `research/` with:

```
research/{product-name}-positioning/
├── competitive-landscape.md      # Competitors, pricing, positioning gaps
├── reddit-pain-points.md         # User language, pain points, switching triggers
├── trends-validation.md          # Market trends, funding signals, analyst coverage
├── market-sizing-gtm.md          # TAM/SAM/SOM, GTM channels, platform risk
├── deep-dive-*.md                # 2-3 focused investigations
├── positioning-research.md       # Final synthesis (the main deliverable)
└── browser-data/                 # Archived browser session artifacts
```

The final `positioning-research.md` is a 12-section document:

1. Executive summary (go/no-go decision ready)
2. Market opportunity & timing
3. Competitive landscape with pricing analysis
4. User research with direct quotes and sources
5. Go-to-market landscape
6. Value proposition ranking
7. Positioning strategy recommendation
8. **Structural shifts** — forces reshaping the segment (e.g., AI commoditizing code, category convergence) with evidence, timelines, and implications
9. Risks & threats (including platform risk)
10. Keyword & search strategy
11. Concrete pricing recommendation with validation methodology
12. Prioritized next steps

Every claim is backed by numbered citations with URLs. Every user quote links to the original thread or review.

## Prerequisites

### Claude Code (required)

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) is Anthropic's CLI tool for Claude. It's what runs the skill, manages the parallel research agents, and orchestrates the entire workflow. Install it and make sure you have an active Anthropic API key configured.

### Browser automation (optional, but recommended)

Most research (~95%) uses WebSearch and WebFetch — built-in Claude Code tools that search Google and fetch web pages without a browser. This includes Reddit (via `site:reddit.com` search + `old.reddit.com` fetching) and competitor reviews on G2/Capterra. Browser automation is only needed as a fallback for JS-heavy sites and Google Trends.

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

- **Sonnet for research, Opus for synthesis.** All parallel research agents (Phase 2 and Phase 3) run on `model: "sonnet"` — the cheaper, faster model. These agents do search-and-summarize work that doesn't need the most capable model. Only the final synthesis (Phase 5), where the skill takes a position and writes opinionated recommendations, uses Opus.
- **WebSearch/WebFetch over browser automation.** Browser tools are expensive in tokens (screenshots, DOM snapshots, multi-step interactions). The skill uses lightweight WebSearch and WebFetch for ~95% of research — including Reddit (`site:reddit.com` + `old.reddit.com`) and G2/Capterra reviews. Browser is only used as a fallback when static fetching fails.
- **Parallel execution.** Running agents in parallel doesn't save tokens, but it cuts wall-clock time in half — you get results in 30-40 minutes instead of 60+.

### Reducing costs

If cost is a concern:

- **Skip deep dives.** Phase 3 (deep dives) adds 2-3 more agents. You can interrupt after Phase 2 and ask Claude to synthesize with what it has.
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
4. It launches 2-3 deep-dive agents based on findings (~15-20 min)
5. It synthesizes everything into the final positioning document
6. (Optional) It generates keywords for Google Keyword Planner — paste results back for analysis

### After the research

The positioning document is a starting point, not the final word. Once the run completes, keep the conversation going — Claude still has all the research context loaded. This is where you get the most value:

- **Deep dive on a specific angle.** "Tell me more about how competitor X prices their enterprise tier" or "What exactly are users saying about onboarding friction?"
- **Cross-check references.** Ask Claude to re-read a specific source and verify a claim. Citations can drift during synthesis — spot-checking keeps the document honest.
- **Challenge the recommendations.** "Why did you rank this value prop higher than that one?" — forcing Claude to defend its reasoning often surfaces nuance the document missed.
- **Explore adjacent questions.** "What would the positioning look like if we targeted SMBs instead of enterprise?" or "How does this change if we go freemium?"
- **Request additional research.** "Can you look into the regulatory landscape for this market?" or "Find more Reddit threads about migration pain from competitor Y."

The intermediate reports (`competitive-landscape.md`, `reddit-pain-points.md`, etc.) are also useful to reference directly — ask Claude to pull specific data from them.

## Configuration

The project includes a `.claude/settings.local.json` that pre-approves the tools the skill needs (web search, web fetch, browser automation, file I/O). You can review and adjust these permissions as needed.

## Tips

- **Be specific in your memo.** The more context you give upfront, the better the research. Include target audience, known competitors, and what angle matters most to you.
- **Answer the clarifying questions thoughtfully.** These directly shape which subreddits are searched, which competitors are investigated, and which angles are prioritized.
- **Browser tools are a fallback.** The skill uses WebSearch and WebFetch for ~95% of research. Browser automation (Chrome or playwright-cli) is only used when a page blocks static fetching.
- **Research takes time.** A full run with deep dives takes 30-40 minutes. The parallel agents do the heavy lifting.
- **Review intermediate reports.** The individual agent reports in the research folder are useful on their own — you don't have to wait for the final synthesis.

## Example Output

Here's what a research folder looks like after a complete run:

```
research/ai-qa-testing-positioning/
├── competitive-landscape.md       (27 KB)
├── reddit-pain-points.md          (16 KB)
├── trends-validation.md           (18 KB)
├── market-sizing-gtm.md           (14 KB)
├── agentic-qa-deep-dive.md        (40 KB)
├── deep-dive-ai-code-quality.md   (15 KB)
├── deep-dive-top-competitors.md   (25 KB)
├── positioning-research.md        (83 KB)  ← main deliverable
└── browser-data/
```

The final positioning document is typically 30-80 KB — a thorough, opinionated analysis with concrete recommendations.

## License

MIT
