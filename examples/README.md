# Examples

Two complete worked examples and three additional page specs.

```
ramp/     the full evidence chain behind the Ramp page
attio/    the same, for Attio
reference-specs/   page specs only (CircleCI, Neysa, SymphonyAI)
```

## Re-render one without spending an API call

Each company folder carries every artifact `render.ts` needs. Copy them into `data/` (the tool's working directory, gitignored) and render:

```bash
mkdir -p data
cp examples/ramp/ramp.*.json examples/ramp/ramp.logo.svg data/
npm run render -- --company ramp
# → output/ramp.html
```

Swap `ramp` for `attio` for the other one. Add `--shot` to also produce a full-page screenshot — that's the artifact the agent inspects during visual QA. (The booking section embeds a live calendar from `app.cal.com`, so it renders on a deployed page but appears blank in a local screenshot, which has no network. That's expected.)

## What each file is

| File | Written by | What it holds |
|---|---|---|
| `<slug>.brand.json` | `fetch:brand` | Brandfetch's answer — colors, fonts, firmographics, logo pointer |
| `<slug>.logo.svg` | `fetch:brand` | The logo, inlined into the page as a data URI at render time |
| `<slug>.step0.json` | the agent | Per-platform ad counts and the **resolved advertiser identity**, with a note on what was and wasn't confirmed |
| `<slug>.profile.json` | `score` | The tier and the signal breakdown behind it |
| `<slug>.page.json` | **the agent** | The page spec — the one creative artifact. An ordered list of typed sections |
| `<slug>.html` | `render` | The finished standalone page |

`step0.json` is the most interesting file to read: its `note` field is the agent's own record of the research, including the channels it *rejected* and why. Both examples excluded Meta because the advertiser identity couldn't be confirmed among namesakes.

## Two caveats

**The raw ad-library dumps aren't here.** `<slug>.ads.json` and `<slug>.google.ads.json` run to several MB of SearchApi response data — vendor data, not ours to redistribute. The counts and themes distilled from them live in `step0.json`.

**The numbers inside the pages are illustrative.** Recommendation cards and any pipeline figures are labeled as such on the page itself. The *ad research* is real; the projected outcomes are examples of shape, not claims.
