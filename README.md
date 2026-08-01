# Claude Landing Pages

**A landing-page generator that does the research first — orchestrated end-to-end inside [Claude Code](https://claude.com/claude-code).**

Give it a company name. It researches that company's *real* paid-advertising strategy across Google, LinkedIn and Meta, decides whether they're worth pursuing, then designs and writes a landing page grounded in what they actually advertise — and renders it to a standalone HTML file.

No template gets filled in. No separate model API gets called. **Claude is the CMS.**

<p align="center">
  <img src="docs/images/ramp-page.png" alt="A generated landing page for Ramp, in Ramp's own acid-yellow brand color" width="680">
  <br>
  <em>Generated for Ramp from 3,986 of their live ad creatives. <a href="docs/CASE-STUDY.md">Read the full walkthrough →</a></em>
</p>

---

## What it actually does

You are selling something. You want a landing page aimed at one specific prospect — not a generic one with their logo pasted in.

You type `/demand-gen ramp.com`. Roughly ten minutes later you have `output/ramp.html`: a complete, standalone landing page, in Ramp's own brand color, whose argument is built from the 3,986 ads Ramp is actually running right now.

**In between, it:**

1. **Looks up their brand** — logo, colors, fonts — then screenshots their real homepage to check the color is right, because brand APIs get it wrong often.
2. **Pulls every ad they're running** on Google, LinkedIn and Meta from the public ad libraries, and reads the copy.
3. **Decides whether to bother.** A company running almost no ads gets skipped — there's nothing to personalize against.
4. **Writes the page.** Not filling a template: choosing which sections this particular story needs, in what order, and writing every line from what their ads actually say.
5. **Looks at the result** and re-renders if the logo is invisible or the button text is unreadable.

**What you need:** Node 18+, a SearchApi key, a Brandfetch key, and Claude Code. Both keys have free tiers.

**What it costs per page:** a few hundred API calls and about ten minutes, most of it unattended.

**What it's not:** a hosted product, a SaaS, or something with a UI. It's a folder you open in Claude Code and talk to.

---

## The idea

The interesting part isn't the scripts. It's the division of labor.

| | Does |
|---|---|
| **Scripts** (`src/`) | The boring, structured extraction — pull ad creatives, brand assets, firmographics; compute a tier score. Deterministic, testable, cheap. |
| **Claude** | Every judgment call — sourcing, identity resolution, brand verification, ICP qualification, **narrative design**, copywriting, and visual QA. |

Most "AI content" tools invert this: the model fills a fixed template and a human supplies the judgment. Here the model supplies the judgment and a *composer* supplies the consistency. `render.ts` is a **section composer, not a template** — Claude authors an ordered list of typed sections chosen for *this* company's story, so every page has a different information architecture. Same design system, different argument.

## The pipeline

```
company name
   │
   ▼
① BRAND        fetch:brand ......... logo, colors, fonts, firmographics  (Brandfetch)
   │           + Claude verifies color/logo against the REAL homepage screenshot
   ▼
② ADS          fetch:google ....... every Google creative      (SearchApi)
   │           fetch:linkedin ..... every LinkedIn creative + copy
   │           fetch:meta ......... Meta ad library (if identity confirmed)
   │           merge:vision ....... Claude reads image-archived ad copy in Chrome
   ▼
③ QUALIFY      score .............. Heavy / Moderate / Light tier
   │           + Claude's ICP-fit judgment (B2B? real GTM motion?)
   ▼
④ COMPOSE      Claude authors <company>.page.json
   │           — an ordered list of typed sections, chosen for THIS story
   ▼
⑤ RENDER       render --shot ...... page.json → output/<company>.html + preview.png
   │           + Claude looks at the screenshot and re-renders if it's off
   ▼
one finished landing page
```

Each script writes a JSON artifact the next step reads. The page spec Claude writes is the one *creative* artifact; everything else is evidence feeding it.

Run the whole thing with one command:

```
/demand-gen ramp.com
```

## What makes the page personal

Two sections do the real work, and both are hard to fake:

**`homework`** — *"we studied how you run paid before writing a word."* A per-channel recap of the target's actual creative counts, running dates, and themes, read from their live ad copy. This is the personalization proof: it isn't flattery, it's receipts.

**`statement`** — the mirror. *You do X for your customers; who does X for you?* Phrased in the company's own vocabulary. For Ramp:

> Ramp gives 70,000 finance teams real-time control of every dollar they spend — but its *own*
> growth engine runs the way finance ran *before* Ramp: spend out across channels, signups
> in-product, revenue in the CRM, nothing joined daily.

The tone rule that keeps this from being obnoxious is encoded in [the command](.claude/commands/demand-gen.md): frame it as *"the engine is dialed in — here's the gap,"* never *"you can't measure."* Criticism of a stranger's marketing reads as tone-deaf; recognition of it reads as homework.

## Worked example: Ramp

| Step | Result |
|---|---|
| **Brand** | Brandfetch said primary `#1F1F1F`. A screenshot of the real ramp.com said otherwise: the signature is **acid-yellow `#E4F222`** buttons on white. Verified color wins. |
| **Ads** | **2,597** Google creatives (1,805 video, always-on since Jan 2022) · **1,389** LinkedIn creatives with deep vertical ABM (Healthcare, Consulting, Franchisors…) · Meta **excluded** — identity inconclusive amid namesakes |
| **Qualify** | **Heavy, 76.5 / 100.** Strong ICP fit → proceed |
| **Compose** | 11 sections, light-first, two dark emphasis bands — `hero → homework → statement → capabilities → agentLoop → recommendations → comparison → defensibility → integrations → pilot → booking` |
| **Render + QA** | Screenshot confirmed: white logo auto-recolored to ink, yellow CTA readable with dark text, serif italic accent landing on the right phrase |

A second example (Attio: 842 creatives across two channels, Heavy 69.2) is included for comparison — same engine, materially different page.

📖 **[Read the full case study →](docs/CASE-STUDY.md)**

## Hard-won lessons

These cost real debugging time and are now encoded as rules in the workflow:

- **Identity resolution is the #1 trap.** Company names collide catastrophically. Searching "Plain" surfaces ~50 advertisers (PlainID, Plains State Bank, White Plains Hospital). "Ramp" surfaces Kraken and Brawl Stars. Resolve per engine — Google by domain → `advertiser_id`, Meta by confirmed `page_id`, LinkedIn by exact advertiser name — and never trust a raw keyword.
- **Trust nothing a brand API labels.** Brandfetch has mislabeled color *roles*, color *ordering*, and logo *themes* repeatedly. It called Avoma green (it's coral) and Primer blue (it's charcoal). Screenshot the real homepage and verify.
- **Keyword buckets misfire across domains.** Ramp's scorer matched the literal word "ramp" 760 times. Lead with themes read from the actual ad copy, not from a classifier.
- **Compose the arc from the business.** A measurement vendor earns an opportunity-led arc; a workflow company earns a bottleneck arc. Defaulting every company to hero→features→CTA wastes the one thing the agent is good at.
- **If it's not confirmed, don't include it.** No fabricated trust logos, no invented product metrics. Illustrative numbers are labeled illustrative. An unconfirmed channel is omitted, not guessed.

## Run it yourself

Requires Node 18+, a [SearchApi.io](https://www.searchapi.io) key, and a [Brandfetch](https://brandfetch.com/developers) key.

```bash
git clone <this-repo> && cd <this-repo>
npm install
cp .env.example .env      # then paste in your own keys
```

Open the folder in Claude Code and run `/demand-gen <company>`. It ships configured for a placeholder sender, **Northstar Analytics**. To point it at your own company, edit two things:

1. **`config/sender.json`** — name, agent name, accent color, logo path, booking link. This is the only place brand identity lives; nothing in `src/` is hardcoded.
2. **`context.md`** — the positioning the agent writes *from*: what you sell, who buys it, what their real pain is.

Then drop your own mark in `assets/` and point `sender.json` at it.

Individual scripts, if you want to drive them directly:

| Script | Output |
|---|---|
| `npm run fetch:brand -- --company <slug> --domain <d>` | `data/<slug>.brand.json` |
| `npm run fetch:linkedin -- --company <slug> --advertiser "<Name>" [--resolve]` | `data/<slug>.ads.json` |
| `npm run fetch:meta -- --company <slug> [--query "<Name>" --resolve \| --page-id <id>]` | `data/<slug>.meta.ads.json` |
| `npm run fetch:google -- --company <slug> --domain <d>` | `data/<slug>.google.ads.json` |
| `npm run merge:vision -- --company <slug> --file data/<slug>.google.vision.json` | updates the Google file |
| `npm run score -- --company <slug>` | `data/<slug>.profile.json` |
| `npm run render -- --company <slug> [--shot]` | `output/<slug>.html` |

## What's in here

```
.claude/commands/demand-gen.md   the workflow — the actual "program"
config/sender.json               who the pages are FROM — the only brand identity in the repo
src/                             fetchers, scorer, and the section composer
src/fallback/                    Playwright scrapers + a raw-payload probe (rarely needed)
context.md                       GTM analytics primer the agent writes from
examples/ramp/                   every artifact behind the Ramp page — runnable
examples/attio/                  same, for Attio
examples/reference-specs/        three more page specs (CircleCI, Neysa, SymphonyAI)
docs/CASE-STUDY.md               the full walkthrough
docs/DESIGN-SYSTEM.md            the fixed visual language
```

Each `examples/<company>/` folder holds the complete evidence chain — brand, scoring, page spec, and the rendered page — so you can re-render a real example without spending an API call. See [examples/README.md](examples/README.md).

**Deliberately not included:** API keys, internal positioning and competitor battlecards, and the raw ad-library dumps (that's vendor data, not ours to redistribute). The `.gitignore` enforces all three.

## Credits & license

Built with [Claude Code](https://claude.com/claude-code). MIT licensed — see [LICENSE](LICENSE).

The sender in every example is **Northstar Analytics**, a placeholder. Company names, logos, and ad copy referenced in `examples/` and `docs/` belong to their respective owners and appear for illustration only; the ad data is from public ad-library records.
