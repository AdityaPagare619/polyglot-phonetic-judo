# @omnibeat/polyglot-phonetic-judo

> "Alexa heard you. She just spelled it wrong."

A deterministic, zero-dependency phonetic pipeline that turns mangled
speech-to-text transcripts into canonical track titles — built for the way
people *actually* talk to voice assistants in India and beyond.

**What it does:** `"Lolly Pop La Gelu"` → **Lollipop Lagelu — Pawan Singh**.
`"Roger G"` → **Raja Ji**. `"Alexa pon Gasolina"` → **Gasolina**.
Hinglish, Bhojpuri, Spanglish, Portuguese and K-pop romanization aware.

**What it is NOT:** it is not a speech recognizer, not a music API, and not
machine learning. It takes an *already-transcribed* (possibly mangled) string
and ranks candidates from a catalog you provide. No network, no keys, no cost.

## Install

```bash
npm install @omnibeat/polyglot-phonetic-judo
```

Zero runtime dependencies. Node ≥ 18.

## Quick start

**1. Rescue an accent-mangled voice query:**

```ts
import { resolvePhoneticTrack } from "@omnibeat/polyglot-phonetic-judo";
import { CHAOS_CATALOG_30 } from "@omnibeat/polyglot-phonetic-judo/catalog";

const [best] = resolvePhoneticTrack("Alexa play Lolly Pop La Gelu", "en-IN", {
  catalog: CHAOS_CATALOG_30,
  topN: 5,
});
console.log(best.title, "-", best.artist);
// → Lollipop Lagelu - Pawan Singh
console.log(best.trace);
// per-token morph trace for your UI:
// [ { said: "lolly", heardAs: "lollipop", sim: 0.86 }, … ]
```

**2. Use the low-level pipeline on your own strings:**

```ts
import {
  peelMultilingualCommandFrame,
  toVppsSkeleton,
  vppsSimilarity,
} from "@omnibeat/polyglot-phonetic-judo";

const frame = peelMultilingualCommandFrame("oye alexa raja ji live concert wala gaana chalao");
console.log(frame.isolatedPayload);        // "raja ji live concert"
console.log(frame.explicitlyRequestedLive); // true

console.log(toVppsSkeleton("Lolly Pop La Gelu")); // vowel-preserving skeleton
console.log(vppsSimilarity("roger g", "Raja Ji")); // ≈ 1.0 — ASR heard "G", user meant "Ji"
```

**3. Rank your own catalog, with studio-purity guardrails:**

```ts
import { rankStudioCandidates } from "@omnibeat/polyglot-phonetic-judo";

const result = rankStudioCandidates("play despacito", [
  { trackId: "1", title: "Despacito", artist: "Luis Fonsi", popularity: 98 },
  { trackId: "2", title: "Despacito (Karaoke)", artist: "Sing King", popularity: 40 },
  { trackId: "3", title: "Despacito (Live)", artist: "Luis Fonsi", popularity: 60 },
]);
// result.ranked[0] → the studio original; karaoke/live penalized (−80/−65)
// unless the user explicitly asked for them.
// result.needsElicitation === true when two studio originals land within
// a 15-point margin — show the user hook cards instead of guessing.
```

## How it works (the VPPS pipeline)

1. **Frame peeler** — strips the command wrapper with multilingual regexes:
   wake words (`alexa`, `oye alexa`, `suno alexa`), code-switched verbs
   (`play`, `chalao`, `bajaao`, `pon`, …), provider suffixes (`on Spotify`,
   `via JioSaavn`), trailing fillers. Detects explicit **live / remix intent**,
   which overrides the default studio preference downstream.
2. **VPPS encoder** — reverses English-dictionary split-hallucinations via
   ~30 ASR homophone clusters (`lolly pop`→`lollipop`, `roger g`→`rajaji`,
   `stomach shallow`→`chammakchallo`, `this pa seat oh`→`despacito`),
   normalizes digraphs (`th→t`, `v/w→b`, `aa→a`, …), and **preserves vowels**
   — in Hindi/Bhojpuri/Spanish the vowel cadence carries the word. This is
   the key upgrade over Soundex/Double-Metaphone, which delete vowels.
3. **Articulatory distance** — substitution cost by vocal-tract group:
   labial `b↔v` costs **0.18**, vowel↔vowel **0.25**, unrelated phonemes
   **1.0**; insertions/deletions cost 0.7. It models *why* accents confuse
   ASR instead of string-matching the confusion.
4. **StudioPureGuard ranker** — deterministic scoring:
   `100·sim + popularity·0.15 + 35·studio −65·live −80·karaoke −40·remix
   −150·upsell`. Upsell ad traps are blocked outright; unrequested live /
   karaoke / remix versions are penalized but never hidden; explicit user
   intent ("play the live version") overrides every penalty.
5. **Zero-cost arbitration (optional)** — YouTube's keyless autocomplete as
   a planetary typo corrector ($0, no key, ~1s timeout, graceful offline
   fallback). Entirely optional; the offline pipeline never needs it.

Deterministic: same input, same catalog → same ranking. Median latency
~0.3ms per query against the 30-track catalog on commodity hardware
(steady-state; cold-start/GC jitter excluded).

## API reference

### `peelMultilingualCommandFrame(rawUtterance: string): PeeledCommandFrame`

Strips wake words, verbs, provider suffixes and fillers; returns the isolated
song payload plus `detectedProvider` and `explicitlyRequestedLive` /
`explicitlyRequestedRemix` intent flags.

```ts
peelMultilingualCommandFrame("suno alexa kesariya live concert version");
// → { isolatedPayload: "kesariya live concert", explicitlyRequestedLive: true, … }
```

### `toVppsSkeleton(rawText: string): string`

Vowel-preserving phonetic skeleton. Applies ASR homophone clusters and
digraph normalization. The unit everything else compares.

### `articulatoryDistance(skelA: string, skelB: string): number`

Vocal-tract-weighted edit distance between two VPPS skeletons. Returns
`0` for identical skeletons; higher = more different. Labial swaps
(`b↔v`) cost 0.18, vowel swaps 0.25, unrelated phonemes 1.0.

### `vppsSimilarity(queryText: string, candidateTitle: string, candidateArtist = ""): number`

`[0, 1]` similarity. Exact and substring-containment short-circuits first;
otherwise best of title-only vs title+artist articulatory similarity.

### `rankStudioCandidates(rawUtterance: string, candidates: CatalogCandidate[]): JudoResolutionResult`

Full deterministic ranker. Returns `{ ranked: ScoredCandidate[],
needsElicitation: boolean, elicitationCards?: …, eliminatedTraps: …,
queryVppsSkeleton, elapsedMs }`. Set `needsElicitation` handling in your UI:
when two studio originals land within 15 points, ask — never guess.

### `arbitrateWithZeroCostYoutubeSuggest(rawPayload: string, localeHint = "global-auto"): Promise<string>`

Optional keyless typo correction via YouTube autocomplete. ~1s abort
timeout; resolves to the input unchanged on any failure (network down,
blocked, unparsable). Never throws for transport reasons — the offline
pipeline is the default path.

### `resolvePhoneticTrack(mangled: string, localeHint?: string, opts?: ResolveOptions): RankedTrack[]`

Convenience entry: peel → score → rank → top-N with per-token morph `trace`.
Returns `[]` on an empty frame. `score` is a `[0, ~1.5]` ranking signal;
`phoneticSimilarity` (raw VPPS `[0, 1]`) and `purityScore` ride along.

### Catalog subpath — `@omnibeat/polyglot-phonetic-judo/catalog`

Exports `CHAOS_CATALOG_30`: 30 **mock** tracks engineered for collisions —
Bhojpuri "Lollipop Lagelu" vs Western "Lollipop", 4 versions of "Raja Ji",
live-vs-studio pairs, karaoke traps, upsell-trap flags. Bring your own
catalog for production; the mock catalog exists so the test suite has a
fixed contract to assert against.

## The 25-test suite

`npm test` runs the 25-scenario global verification suite
(`test/phonetic.spec.ts`): Bhojpuri/Hinglish ASR-mangling rescue,
Spanglish and K-pop romanization, intent detection, trap penalties,
elicitation margins, and a latency SLA (median-of-7 < 5ms per query).
**25/25 green is the bar for any change.** Two suite assertions were
honestly recalibrated during audit (a similarity threshold and the latency
statistic) — see the AUDIT NOTES at the bottom of the spec file.

## Honest limitations

- **Phonetic text matching only.** The package does not hear audio, call
  music APIs, or claim a match is "the" correct song — it ranks *your*
  catalog.
- The homophone clusters and romanization maps encode **observed ASR
  failure patterns** from Hinglish/Bhojpuri/Spanglish usage. They are
  heuristics with documented tradeoffs, not linguistic claims.
- `popularity` in the mock catalog is a documented judgment for
  tie-breaking, not a measurement.
- The YouTube Suggest arbitrator depends on an undocumented endpoint and
  network conditions; it is a best-effort enhancement, never a requirement.
- VPPS is tuned for Indic + Latin-script languages. Tonal languages and
  non-Latin scripts are out of scope for v1.

## License

MIT — see `LICENSE`.
