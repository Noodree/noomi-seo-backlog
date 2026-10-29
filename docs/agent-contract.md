# NOOMI SEO backlog — data contract (schema 2, 2026-09-16 evening)

**Any agent that touches `backlog-data.json` reads this file first.** SEO-persona infrastructure on the CRM pattern: Audrée uses the app (`backlog.html`), agents use these rules. Rebuilt the same day after Audrée's review: **the board holds content ideas, not keywords.** Keywords and competitor pages are raw data that feed ideas.

## File map
```
noomi/seo/backlog/
├── backlog.html          # the app: Ideas · Keywords · Competitors · Formula
├── backlog-data.json     # THE data. One JSON, canonical, Drive-versioned.
├── agent-contract.md     # this file
├── README.md             # for Audrée
├── run-log.md            # one entry per weekly run (newest on top)
├── import/seed.py        # rebuilds keywords[] + competitorPages[] from raw pulls (agents never run it)
├── import/ideas-seed.json# the 2026-09-16 brainstorm output (merged once)
└── pulls/                # raw DataForSEO responses, full mode, one file per call (YYYY-MM-DD-<what>.json)
```

## Three layers
1. **Keywords** = raw demand data, Canada only (Google Ads CA + Bing CA, en and fr). Status `raw` | `targeted` (attached to an idea) | `ignored`. Not a to-do list.
2. **Competitor pages** = what people who buy in our category actually read. Two pulls per competitor (Labs `relevant_pages`): **Canada** (location 2124, field `etvCa`) and **their home market, the US** (location 2840, field `etvUs`), because Canadian buyers read the same US pages and the US numbers show which topics work at scale. Tracked: takeroon.com, mindlabpro.com (ca.mindlabpro.com has no Canadian rankings; use www), magicmind.com, mojo.shop (MOJO gummies; mojo.com is a sports app), takethesis.com (no blog: product pages only), lemmelive.com (Lemme), staywyldorganics.com (mushrooms, mostly out of lane); neurohacker.com (Qualia) has no Canadian rankings. Field `ourFit` 1 to 5. **Rule: any in-lane competitor page with `etvCa` ≥ 3 or `etvUs` ≥ 50 gets an idea** (`competitorRef` set, `ourFit` filled), then the idea's Canadian keyword volume decides its priority. Raw data, feeds ideas.
3. **Ideas** = the board. One idea = one article: working title, angle, funnel, the keyword group it targets (primary keyword goes in the H1; the rest in H2s and FAQ; EN and FR keywords attach to the same idea), outline, FAQ questions, estimated traffic, only-NOOMI asset, image spec, status. Status is a tag: `idea` | `brief` | `drafting` | `review` | `scheduled` | `published` | `cancelled`. Cancel is one click; agents never delete an idea, they cancel with a note. **Ideas have no language** (`lang: "en+fr"`): every article ships in EN, the FR twin is produced from it (Audrée 2026-09-17). `research: "requested"` marks an idea Audrée typed in the app that still needs volume, KD, outline and FAQ; the next run (or "research the ideas" in a session) fills it and sets `research: "done"`.

## Schema
```json
{
  "meta": {"schemaVersion": 2, "updatedAt": "ISO",
    "weights": {"demand": 0.35, "winnability": 0.25, "fit": 0.25, "geo": 0.15},
    "funnelTargets": {"tof": 40, "mof": 40, "bof": 20},
    "funnelDefinitions": {"tof": "...", "mof": "...", "bof": "..."},
    "ideaStatuses": [...], "keywordStatuses": ["raw","targeted","ignored"],
    "clusters": ["brain-fog","focus-energy","nootropics","brain-supplements","ingredients","memory","perimenopause-cognition","brand-compare","fr"],
    "tags": [...]},
  "keywords": [{"id": "kw-<slug>-<lang>", "keyword": "", "lang": "en|fr",
    "volume": {"googleCa": 8100, "bingCa": 330}, "competition": 7, "cpc": 0.53, "kd": 39, "trend": "+0%",
    "intent": "I|CI|T", "cluster": "", "funnel": "tof|mof|bof (hint only)",
    "fit": 1-5, "geo": 1-5, "winnability": null|1-5, "priorityOverride": null,
    "status": "raw|targeted|ignored", "ideaId": null, "claimRisk": "none|watch|blocked",
    "source": "gkp:2026-08-27;dataforseo:google:2026-09-16", "notes": [], "createdAt": "", "updatedAt": ""}],
  "ideas": [{"id": "idea-YYYY-MM-<slug>", "workingTitle": "", "angle": "", "lang": "en|fr", "funnel": "tof|mof|bof", "cluster": "",
    "status": "idea", "fit": 1-5, "winnability": null|1-5,
    "primaryKeywordId": "kw-…", "keywordIds": ["kw-…"], "estTrafficOverride": null,
    "outline": ["H2 …"], "faq": ["question …"], "competitorRef": {"domain": "", "url": "", "etvCa": 0} | null,
    "onlyNoomiAsset": "", "draftPath": null, "url": null, "publishAt": null, "shopifyArticleId": null,
    "image": {"register": "R1-R4", "prompt": "", "model": "", "seed": null, "gradeVersion": "", "file": ""},
    "gsc": {"clicks": 0, "impressions": 0, "position": 0, "period": "YYYY-MM-DD..YYYY-MM-DD"} | null, "ga4": {"sessions": 0, "users": 0, "period": ""} | null,
    "tags": [], "notes": [], "createdAt": "", "updatedAt": ""}],
  "competitorPages": [{"id": "cp-<domain>-<slug>", "domain": "", "url": "", "title": "", "etvCa": 60, "rankingKeywords": 38, "top3": 2, "ourFit": null, "ideaId": null, "note": "", "pulledAt": "", "source": ""}],
  "backlinkLeads": []
}
```
All fields except `id`, `status`, `updatedAt` (and `keyword` / `workingTitle`) are optional; tolerate missing fields, preserve unknown ones. Never change an existing id. `notes[]` = `{date, author, text}`; never edit or delete a note whose author is `"audree"`.

## Scoring (the app and agents use the same rules)
- **Winnability** comes from DataForSEO keyword difficulty (`kd`, Labs `bulk_keyword_difficulty`, location 2124): KD ≤20 → 5, ≤35 → 4, ≤50 → 3, ≤70 → 2, else 1. **Never from ad competition** (Audrée 2026-09-16: "rhodiola rosea gummies at 1/5, why would that be hard?"; ad competition measures advertisers, not organic SERPs). Unknown KD shows "?" and counts as 3; the weekly run fills KD for any keyword attached to an idea. A hand-set `winnability` (1 to 5) overrides.
- **Priority** (keyword or idea) = 100 × (w_d × demand + w_w × winnability/5 + w_f × fit/5 + w_g × geo/5), demand = log10(Google CA + Bing CA + 1) / log10(10001). An idea sums its keywords' volume.
- **Estimated traffic** (idea) = summed volume × CTR at the position winnability predicts: 5 → 11%, 4 → 6%, 3 → 3%, 2 → 1.2%, 1 → 0.4%. `estTrafficOverride` wins when set.
- **Funnel** is a property of the idea (Audrée's definitions in `meta.funnelDefinitions` and blog-style-guide §3). "Best supplements for X" is MOF. A keyword's `funnel` is a grouping hint.
- **Fit (ICP fit)**, score the question not the topic: **5** the exact question that reader asks on a work day (afternoon focus, deadlines, stress, sleep debt, caffeine, mental stamina, "does this ingredient work") · **4** same reader, adjacent moment (memory at work, energy without caffeine, ingredient dosages, honest comparisons) · **3** general-public health question the reader might also ask (brain fog basics, supplements in general) · **2** a different life situation (students, seniors, athletes) or a curiosity piece · **1** medical, clinical or off-lane; we do not write it even if it ranks. **GEO** = chance an AI answer cites it: definitional/question-shaped 5, comparisons 4, transactional 1 to 2.
- **Trend** (YoY change from the Keyword Planner export) is not a weight: +6 on P at ≥100%, +3 at ≥50%, −3 at ≤−50%.
- **Estimated Canadian visits of a competitor page** (`etvCa`) = DataForSEO Labs ETV: for each keyword the page ranks for in Canada, Canadian volume × click rate of its position, summed. An estimate, not analytics.

## Rules for agents
1. **Canada only.** Every volume from location_code 2124, languages `en` and `fr`, Google Ads and Bing. Record the source string. Never write US or global numbers.
2. **Lane.** Supplements, nootropics, ingredients, mental performance, focus, energy, brain fog and its everyday causes (sleep, stress, caffeine, screens, afternoon). Medical conditions, drugs, diets and gut disorders are out; `import/seed.py` holds the exclusion list, extend it there.
3. **One idea per intent.** Before creating an idea, search `ideas[]` (including cancelled ones) and the live posts; attach keywords to the existing idea instead of duplicating. Group all phrasings the same article satisfies ("ashwagandha brain fog", "ashwagandha and brain fog", "ashwagandha brain fog reddit" = one idea).
4. **Claims gate at idea time.** `claimRisk: blocked` keywords never become ideas; `watch` ideas carry a note that the product appears only with approved claims and no treatment verb.
5. **Funnel mix.** Target 40/40/20 across ideas in `brief`…`scheduled`. Pick the next briefs from the stage furthest under target.
5a. **Format rotation.** Every idea carries a `format` (`meta.formats`: explainer, how-to, listicle, comparison, reddit-roundup, pressure-profile, tracker-review, ingredient-hub, faq). Consecutive briefs never share a format; over any 8 published articles no format appears more than 3 times (Audrée 2026-09-17: diversity of angles, not one repeated shape). BOF and MOF keep a share of "top 10 / best X" listicles because AI answers cite ranked lists.
5c. **Reddit.** Reddit is a research input and a format, not a channel: (1) monthly, harvest question phrasings from r/Nootropics, r/productivity, r/Biohackers for FAQ blocks (the follow-up questions people ask after learning about a topic: cost, does it work for me, which one for my case, how long, risks; style guide §5.7) and long-tail variants ("X reddit" queries attach to the same idea); (2) the `reddit-roundup` format ("What Reddit says about X, and what the studies say") answers "keyword + reddit" searches with quoted threads (≤15 words per quote, linked) plus evidence; (3) nobody posts NOOMI content on Reddit; founders-as-individuals only, per the radar's 90/10 rule.
5d. **Use-case pages.** `meta.useCasePages` maps clusters to the two use-case pages (corporate athletes; perimenopause brain fog). Every brief names the one it links to; the run's draft step includes that link in the body. First-hand quotes from Audrée's network are an asset she offers (Sept 17): note them in `onlyNoomiAsset`, collect them through her, never invent.
5e. **Niche ICP as angle, not keyword.** "Tech workers stress/burnout", "supplements for tech workers", "AI entrepreneur" phrases have ~10 Canadian searches a month each (pull 2026-09-17). The ICP is applied as the angle and the first-hand asset on keywords that do have volume (burnout symptoms 3,600, canadian entrepreneurs 1,300, cognitive load 1,300, context switching 320, ai fatigue 210).
5g. **No pricing content (Audrée 2026-09-17).** No article compares prices, cost per day, or dose counts across brands, and no 'why 4 gummies per dose' style piece. NOOMI's premium is handled on the PDP and in ads, not in the blog. Comparisons that remain are ingredient-vs-ingredient or format-vs-format (shots vs gummies) on evidence and dosing science only.
5f. **Competitor brands in our content (Audrée 2026-09-17).** No article reviews a named competitor's product, and no article passes a verdict on a competitor's quality or claims. Allowed: category comparisons (shots vs gummies, gummies vs capsules) and comparison tables that list every brand's own published label facts and list prices in CAD, NOOMI included with the same scrutiny. Not allowed: "X review", "is X worth it", quoting scraped customer reviews about a competitor, or characterizing a competitor's marketing. Any piece that names a competitor gets a legal read before publish. `reddit-roundup` pieces quote Reddit about topics and ingredients, never about a named brand.
5b. **Coverage.** Every keyword with P ≥ 70 is either attached to an idea or `ignored` with a note saying why (the Formula tab lists the uncovered ones). A high-priority keyword with no idea is a bug, not a backlog.
6. **Status moves need evidence.** `drafting` = `draftPath` exists; `review` = calendar block exists; `scheduled` = Shopify article exists with `isPublished: false` and `publishAt`; `published` = URL live, set by Audrée only.
7. **Raw pulls go to `pulls/`** in full mode (`noAiMode: true`); the data file holds only normalized rows. Keep `pulls/` under 20 MB (delete raw files older than 90 days once ingested).
8. **Read-modify-write** the whole file with a JSON parser, once per run, after reloading. Never regex-edit. Write `updatedAt` on every row you change.
10. **Google data is read-only** and comes only through `seo/tools/google_read.py` (OAuth token in a local secrets folder, never in the repo). Raw pulls go to `seo/data/gsc/`. Never call the Google APIs any other way, never write to Search Console or GA4.
9. Tags only from `meta.tags`; propose new ones in the run summary.

**Competitors view (app, 2026-09-17):** the table shows articles only by default (URL contains /blog/, /blogs/, /journal/, /articles/, /learn/, /news/, /guides/, /resources/); product, collection and site pages stay in `competitorPages` as raw data behind a "Show all pages" toggle. The competitor-page-to-idea rule applies to articles only; never propose an idea from a competitor's product, collection, FAQ or store page.

**Research logs (2026-09-25):** `seo/research/clusters/<cluster>.md`, one per cluster, newest entry on top; read before any research pass, write after. SERP checks: two per article, DataForSEO organic live advanced, location 2124.
