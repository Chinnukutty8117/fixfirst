# Fixfirst: from complaints to action

Built for **PromptWars x ASTRA: From Complaints to Action** (AI for Civic Impact).

Fixfirst turns scattered citizen reports into a prioritized, explainable and trackable work queue. A citizen reports a problem in under a minute. The platform categorizes it, summarizes it, links duplicates, scores its priority with a plain-language explanation, and shows hotspots on a map. An operator then assigns it and moves it to resolution, and the citizen can follow every status change.

## Run it

It is a single static file with no build step and no dependencies.

```
open fixfirst.html        # or double-click it, or serve the folder with any static server
```

- Hosted preview: https://claude.ai/artifact/AjAuZnSjxEhBjjFwMS4zT1
- Google Fonts are loaded if online and fall back to system fonts if not.
- "Use my current location" needs a normal browser tab (some embedded previews block geolocation) and, for a pin to appear, a position inside the demo area (see Limitations).
- All data lives in memory. Refreshing the page resets to the seed data.

## What it covers

| Stage | What the prototype does |
|---|---|
| **Capture** | Text, optional photo, category (or "let the AI choose"), and location by map tap, browser geolocation, or a mocked landmark lookup. Works on a phone-sized screen. |
| **Understand** | Category, one-line summary, duplicate check, and a 0-100 priority with itemised reasons, shown right after submission. |
| **Reveal** | Operations map with priority-coloured dots, cluster rings and a ranked queue. An action brief names the top hotspot and the top item to fix. |
| **Act** | Review, assign to a team (suggested automatically), start work, resolve, each with an optional note. Citizens look up an ID in the Track tab and see the timeline. Merged duplicates follow their parent's progress. |

## Architecture

One HTML file, three layers:

1. **Data**: an in-memory array `R` of report objects (`id`, `text`, `x`, `y`, `cat`, `status`, `log[]`, `dups[]`, `dupOf`, `score`, `factors[]`). 31 synthetic seed reports are ingested through the same pipeline as live ones.
2. **Pipeline** (all plain JavaScript functions):
   - `classify()` → category and confidence
   - `ingest()` → summary and duplicate linking
   - `rescore()` → priority, factors, suggested team (re-run after every change, because priority depends on neighbours and duplicates)
   - `clusters()` → hotspots
3. **UI**: four views (Report, Track, Operations, Limits & honesty) rendered with template strings, plus a hand-drawn SVG map.

## The AI approach

Everything is rule-based and deterministic, so every output can be explained. There is **no language model or external API call**.

**Category.** Each category has a keyword list. The category with the most keyword matches wins, and confidence is high with 2 or more matches, medium with 1, and low with none. If the citizen picks a category, theirs is used and the AI's suggestion is shown beside it.

**Summary.** The first sentence of the report, trimmed to about 85 characters.

**Duplicate detection.** A new report is compared with existing open, non-duplicate reports of the same category. It is linked when it is within 9 map units (roughly 500-600 m) **and** either the word overlap (Jaccard similarity on stop-word-filtered tokens) is at least 18% or the distance is 3 units or less. The best-scoring match becomes the parent. The result card shows the distance and overlap that triggered the link.

**Explainable priority** (capped at 100). The score is the sum of the items below, and the UI lists each one.

| Factor | Points |
|---|---|
| Category baseline | waterlogging 14, road 10, streetlight/power 10, water supply 10, waste 8, other 5 |
| Urgency phrases (total capped at 60) | live wire/sparking 34, open manhole 28, injury or collapse risk 22, blocked emergency access 18, structural danger 16, flooding or sewage 14, children nearby 10, near a hospital 10, smoke or fire 10, busy public spot 5 |
| Nearby open reports | +3 each within 9 units, max 15 |
| Merged duplicates | +4 each, max 12 |
| Age | +1 per day open, max 8 |

Levels: Critical 70 and above, High 50-69, Medium 30-49, Low below 30. An urgency subtotal of 28 or more suggests the 24h emergency team. The brief also explains why the top item outranks the second by naming the largest factor the second item lacks.

**Hotspots.** Open, non-duplicate reports within 9 units of each other are chained into groups. A group of at least 3 reports spanning at least 2 categories is a cluster, drawn as a ring on the map. Clusters are ranked by the sum of their report scores.

**Action brief.** A template filled from the data: area, counts, categories, urgent IDs and cluster total. One hand-written rule adds a hypothesis when drain and road problems co-occur: water may be undermining the road surface, so clear the drain before patching.

## Synthetic data

31 invented reports across Hampankatta, Kadri, Bendoor and Jeppu (area names borrowed for flavour; the locations and incidents are made up). They include deliberate duplicates (for example the two Kadri drain reports and the two Hampankatta pothole reports), urgent safety issues (live wire, open manhole, fallen tree blocking an ambulance route), four geographic clusters, and a few reports already assigned, in progress or resolved. No real personal data is used.

## Demo script (about 5 minutes)

1. **Report:** submit "Drain blocked near Kadri bus stop, water on road, children walking through it", pin it near Kadri (or type "Kadri" as the address), then read the category, summary, duplicate link, and priority factors on the result card.
2. **Operations:** open the action brief and point out the Kadri hotspot ring on the map and the "why it outranks" line.
3. **Act:** click the top item, assign it to the suggested team, then start work with a note.
4. **Track:** open the report ID in the Track tab and show the citizen's timeline. Then track the duplicate you just created and show that it mirrors its parent's progress.

## Limitations and honesty note

**Real**
- Every report you submit is processed live by the pipeline above.
- Priority, duplicates, clusters and the brief recompute as data changes.
- The assign and status workflow updates the operator queue and the citizen view.
- Browser geolocation uses the device's actual position.

**Mocked**
- All seed reports are synthetic.
- The map is a schematic SVG covering about 0.05° latitude by 0.06° longitude around Mangaluru (12.85-12.90 N, 74.82-74.88 E). It is not a real basemap and the roads are drawn by hand. Geolocation outside that box is reported but not pinned.
- The address box recognises only four made-up area names.
- No SMS, email or push notifications and no integration with any real authority or ticketing system. "Citizen updates" means the tracking page.
- No accounts, authentication or persistence.

**Where the AI can be wrong**
- Keyword matching misses unusual wording, other languages, sarcasm and reports that mix several problems. It can also misread negation ("light is not out").
- Duplicate detection can miss paraphrases with little word overlap, and can merge two separate problems that are close together and described similarly.
- The priority weights are hand-chosen and not validated against real outcomes. They encode assumptions (for example that children nearby adds 10) that officials should review before any real use.
- Cluster size depends on a fixed 9-unit threshold, so very large or very sparse areas behave differently.
- Photos are stored but **not analysed**.
- The brief is a template and the road/drain hypothesis is a single rule, not learned insight.
- Operators should treat the ranking as decision support, not a verdict.

**Next steps**
- Embedding-based duplicate matching and an LLM for summaries and multilingual reports, with a human review step.
- Photo analysis, for example to confirm a pothole or a blocked drain.
- Real map tiles and geocoding, so location works anywhere.
- Persistence and accounts, plus citizen notifications.
- Tune and test the priority weights against historical complaint and resolution data, and measure fairness across neighbourhoods so that low-reporting areas are not under-served.

## Repository layout (suggested)

```
fixfirst.html     the whole app
README.md         this file
```
