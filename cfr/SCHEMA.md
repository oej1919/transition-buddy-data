# 38 CFR Part 4 dataset schema

Reference data backing `app/disability-reference.tsx` in transition-buddy
(tts-mobile). This is a **reference tool, not a rating calculator** — the
schema exists to hold the published structure and language of the VA
Schedule for Rating Disabilities, not to compute or predict outcomes.

## Naming convention note

`pay-tables/` and `va-rates/` use snake_case keys. This dataset uses
camelCase instead, deliberately — it maps near-1:1 onto the app's existing
`CfrCondition` TypeScript interface (`constants/cfr.ts`), so the fetch layer
in Part 2 can pass the parsed JSON through with minimal reshaping. Flag if
you'd rather it match the older files' snake_case for consistency instead.

## File layout

```
cfr/
  cfr-part4-v1.json   — versioned snapshot (this version)
  latest.json         — identical copy of the current version; this is
                        what the app actually fetches (same convention as
                        pay-tables/ and va-rates/)
  SCHEMA.md           — this file
```

A future amendment to the schedule ships as `cfr-part4-v2.json`, with
`latest.json` updated to match — the app never hardcodes a version number,
so this requires no app code change (see Part 2).

## Top-level shape

```ts
interface CfrPart4Dataset {
  datasetVersion: string        // "v1" — bump on any structural/content change
  cfrTitle: 38
  cfrPart: 4
  schemaSource: string          // canonical source URL, for provenance
  bodySystems: BodySystem[]
}
```

## BodySystem

```ts
interface BodySystem {
  id: string          // stable slug, e.g. "musculoskeletal" — used for routing/filtering, never renamed once shipped
  name: string         // display name, matches real CFR subpart structure
  cfrRange: string     // e.g. "§4.71a" or "§4.100–§4.104"
  conditions: Condition[]   // [] is valid — an empty shell category
}
```

15 body systems matching 38 CFR Part 4 Subpart B exactly (see "Body system
coverage" below for the full list and rationale).

## Condition

```ts
interface Condition {
  id: string                  // stable slug, e.g. "tbi" — never renamed once shipped
  commonName: string          // plain-language name, how a veteran would search ("PTSD")
  vaName: string               // formal CFR/VA name ("Mental Disorders — General Rating Formula")
  bodySystem: string           // matches parent BodySystem.id — intentionally redundant.
                                // Mirrors the existing constants/cfr.ts convention where each
                                // condition already carries its own bodySystem string, so
                                // existing filter/search logic in disability-reference.tsx needs
                                // minimal change after flattening bodySystems[].conditions.
  diagnosticCodes: string[]   // one or more DC codes/ranges, e.g. ["8045"] or ["5235-5243"]
  cfrSection: string           // e.g. "§4.130"
  ratingMethod: "standard" | "faceted"  // see "Rating methods" below
  pactAct: boolean              // true if this is a PACT Act presumptive condition
                                // (Gulf War/burn-pit/Agent Orange exposure) — eligibility framing
                                // for these changes faster than the base schedule; see BACKLOG.md
  status: "shell" | "unverified" | "verified"
  lastVerified: string | null  // ISO date the criteria text was last checked against eCFR;
                                // null when status is "shell" or "unverified"
  summary: string               // "" until written — plain-language summary of what's rated
  ratingTiers: RatingTier[]    // used when ratingMethod is "standard"; [] when "faceted" (always — see below)
  facets: Facet[]                // used when ratingMethod is "faceted"; [] when "standard" (always)
  ecfrUrl: string               // "" until confirmed; see "eCFR URL note" below
  todo?: string                 // present only on unverified/shell entries — what's left to do
}
```

### Rating methods

Most conditions use a single 0–100% ladder (`ratingMethod: "standard"`,
data lives in `ratingTiers`). A handful — TBI (DC 8045) is the one in this
version — don't: they score several independent facets of function, each
on its own scale, and the *highest* facet score determines the overall
rating. For those, `ratingMethod: "faceted"` and the data lives in
`facets` instead. **The two fields are mutually exclusive by convention**:
whichever one `ratingMethod` doesn't point to stays `[]`, always — the UI
reads `ratingMethod` first to decide which field to render, never both.

### Facet / FacetLevel

```ts
interface Facet {
  id: string            // e.g. "memory_concentration"
  name: string           // e.g. "Memory, Attention, Concentration, Executive Functions"
  levels: FacetLevel[]  // [] until populated
}

interface FacetLevel {
  level: 0 | 1 | 2 | 3 | "total"   // "total" = facet-specific max-severity flag (not every facet allows it)
  percent: number                    // the overall rating % this level maps to (typically 0/10/40/70/100)
  criteria: string
}
```

TBI's `facets` is `[]` this version, same as its `ratingTiers` — its
`todo` explains why (10 facets, each independently scored, highest facet
wins) rather than forcing real content into a shape that didn't fit it
yet.

### `status` meanings (drives what the app is safe to render)

- **`"shell"`** — category placeholder only. No DC code, no CFR section,
  no criteria. (6 body systems are 100% shells this version — see below.)
- **`"unverified"`** — identifying metadata (DC code, CFR section, names)
  is filled in and can be cited, but `ratingTiers` is `[]` and `summary`
  is `""` because the actual tier criteria text hasn't been sourced and
  checked against eCFR yet. **Never fabricate this text to fill the gap.**
- **`"verified"`** — fully populated, `lastVerified` is real.

### RatingTier

```ts
interface RatingTier {
  percent: number    // 0, 10, 20, ... 100 — not every condition has every tier
  criteria: string     // the published diagnostic criteria for that tier
}
```

Why `{ percent, criteria }` instead of the old flat `ratingTiers: string[]`
(each string like `"30% — Occasional decrease..."`): separating the number
from the text lets the UI sort/badge tiers without string-parsing, and lets
future tooling answer "how many tiers does this condition have populated
out of how many it should have" — directly useful both for the eventual
per-condition rating-criteria viewer and for tracking population progress
across the whole dataset.

### eCFR URL note

Existing migrated entries have deep links with an internal eCFR
`subject-group-ECFR...` hash segment — that hash isn't guessable, only
obtainable by visiting the actual eCFR page. For the 3 new unverified
entries, `ecfrUrl` uses the simpler section-only permalink form instead
(`ecfr.gov/current/title-38/chapter-I/part-4/section-{X}`) — this is a
real, working URL format (it's exactly what `lib/ecfr.ts`'s `getEcfrUrl()`
helper already generates), just without the deep-link hash. Not a
fabrication, just a less specific permalink until someone visits eCFR to
get the precise one.

## Body system coverage (this version)

15 systems, matching 38 CFR Part 4 Subpart B:

| id | name | conditions this version |
|---|---|---|
| musculoskeletal | Musculoskeletal System | 2 (migrated) |
| eye | Eye (Organs of Special Sense) | 0 — shell |
| ear | Ear & Other Sense Organs | 2 (migrated) |
| respiratory | Respiratory System | 1 (migrated) |
| cardiovascular | Cardiovascular System | 1 (new, unverified) |
| digestive | Digestive System | 1 (migrated) |
| genitourinary | Genitourinary System | 0 — shell |
| gynecological | Gynecological Conditions & Disorders of the Breast | 0 — shell |
| hemic_lymphatic | Hemic & Lymphatic Systems | 0 — shell |
| skin | Skin | 1 (migrated) |
| endocrine | Endocrine System | 1 (new, unverified) |
| neurological | Neurological Conditions & Convulsive Disorders | 3 (2 migrated + TBI new/unverified) |
| mental_disorders | Mental Disorders | 1 (migrated) |
| dental_oral | Dental & Oral Conditions | 0 — shell |
| infectious_immune_nutritional | Infectious Diseases, Immune Disorders & Nutritional Deficiencies | 0 — shell |

9 conditions migrated as-is from `constants/cfr.ts` (status `verified`,
`lastVerified: "2024-08-01"` — preserving their original vintage, not
claiming they were re-checked today). 3 new conditions added as
`unverified` shells (Hypertension, TBI, Diabetes mellitus) — high-value
because they're extremely common veteran claims, but no criteria text
exists for them yet pending real eCFR sourcing (see BACKLOG.md task,
Part 4). 6 body systems are pure category shells with zero conditions.

## Annual/event-driven re-verification

Unlike pay tables and VA rates (annual COLA cycle, predictable December 1
effective date), 38 CFR Part 4 amendments are irregular — there's no fixed
re-verification date. `lib/ecfr.ts`'s existing `checkCfrCurrency()` already
handles *detecting* drift at runtime by comparing against eCFR's live
amendment date; this dataset's `lastVerified` per condition is the
complementary static record of when each entry's text was last
hand-checked. See BACKLOG.md for the full population task.
