# Changelog

All notable changes to the AniScraper (formerly Nyaa Stremio Addon) are documented here.

---

## [2.6.3] - 2026-08-25 — Two Ways to Number an Episode: Cross-Source Metadata and Season Mapping

### Fixed
- **Half of a show's releases were unreachable because of one apostrophe** — reported against *Beyond Time's Gaze* (`tvdb:472119` / 光阴之外): searching the show returned only releases spelled with the apostrophe, and searching without it returned only releases spelled without. The query path preserved punctuation end to end while every gate around it discarded it — `normalizeTitle()` folds diacritics and nothing else, and providers send the string verbatim — so the addon only ever searched whichever spelling its metadata source happened to carry. Confirmed on Nyaa: *Shridhuu* publishes this show as `Beyond Time's Gaze`, *ToonsHub* as `Beyond Times Gaze`, and no query existed that could reach the second. Alternate spellings are now searched after the canonical ones, so they cost a request only when the canonical form came back short.
- **A release numbered per-season was dropped against a catalog numbered flat, and vice versa** — TVDB and TMDB both file *Beyond Time's Gaze* as a single season (46 and 52 episodes respectively) while release groups name episodes per-season, so a request for episode 35 was rejecting `[ToonsHub] Beyond Times Gaze S02E09` on `torrentSeason !== sourceSeason`. Season 1 ran 1-26, so S2E9 **is** episode 35 — the two schemes are each internally consistent and neither is wrong. A release's season/episode is now converted to an absolute number and compared with the request's. Live for episode 35: 1 release before, 4 after.
- **A per-season Kitsu id searched the wrong episode entirely and returned nothing** — Kitsu files a long-running show as one entry *per season*, so `kitsu:45077` is *Swallowed Star* season 2 and its "episode 1" means absolute episode 27. Read as absolute 1, the search matched only two 143-episode multi-season packs, which the seeder threshold then dropped — 0 streams. Kitsu publishes the season chain itself (each entry links its prequel/sequel and carries its own `episodeCount`), so the boundaries are now reconstructed per show with no hand-written configuration: *Swallowed Star* resolves to S1 1-26, S2 27-52, S3 53-85, S4 86-∞. Verified live — `kitsu:45077:1` → absolute 27 → 3 releases; `kitsu:48061:20` → absolute 105 → 2 releases.
- **A donghua reached through a Kitsu id was classified as not-donghua, silently disabling its absolute-episode search** — the Chinese-origin signal is read from TVDB/TMDB `originalLanguage`, and a `kitsu:` request resolves Kitsu alone, which carries no such field. The Level-0 absolute-episode query is gated on that flag, so *Swallowed Star* never searched its absolute episode at all. The gate now also accepts a resolved season layout, which is direct evidence that the show has a meaningful absolute number and that the request's own number is not it.
- **Kitsu dropped out of a request entirely, with no error, for any show without abbreviated titles** — `resolveKitsuId` spread `attrs.abbreviatedTitles` unguarded, and Kitsu returns `null` for that field on many entries. `[...null]` throws, the function's own `try` swallowed it, and the show silently lost its romaji and Japanese titles.
- **The shared episode cache could not match two spellings of the same show** — lookups compared title aliases as exact strings, so two users whose catalog addons spell a show differently never shared a cached document and each paid for the identical search.

### Changed
- **Metadata sources are resolved together instead of first-wins** — each source was guarded behind the previous one having failed (`if (tmdbId && !kitsuInfo && !tvdbInfo)`), so a `tvdb:` request never called TMDB at all. That cost the richest alias list available (TMDB's `alternative_titles`), a second opinion on season shape, and better artwork. The identifier needed was already in hand: TVDB's response carries the TMDB id in `remoteIds`, TMDB's carries the TVDB id in `external_ids`. Sources now resolve in parallel and merge field by field with recorded provenance. For the reported show this took the alias set from 4 to 9.
- **Per-show corrections can be declared in `src/data/show-overrides.json`** — for the cases no amount of cross-referencing can settle. Keyed by external id and never by title (a title-keyed config would reproduce the aliasing problem it exists to fix), each entry lists every id namespace the show is catalogued under and matches on any of them, so one entry answers for `tvdb:`, `tmdb:`, `tt` and `kitsu:` requests alike. Supports extra aliases, denied aliases, per-season ids, and an explicit season layout whose final season may be left open for a show still airing. Every field is optional and applied as a patch, so an entry the sources later render redundant becomes a no-op rather than a stale wrong answer.
- **An alias too generic to identify a show can now be denied per-show** — TMDB lists `"Beyond Time"` as an alternative title for *Beyond Time's Gaze*, the same string that served a *Frieren* episode in 2.6.2. Widening the alias union re-imported it, so the override file can remove it at the source rather than the matcher cleaning up after a wasted query.

---

## [2.6.2] - 2026-08-18 — Wrong-Show Contamination: Every Match Must Name the Series

### Fixed
- **A request could return another show's episode entirely, correctly labelled with the episode you asked for** — reported against *Beyond Time's Gaze* (`tvdb:472119`, the 2025 donghua 光阴之外), whose S1E23 was served an episode of *Frieren: Beyond Journey's End*. The metadata was never wrong; the matcher was. Nothing on the series path checked which *show* a release belonged to: `matchesRequest` accepts the request's title list and never reads it, deciding purely on season/episode numbers, and the arc gate above it only rules on releases that already overlap the request (zero overlap is "different show or other language", deliberately not its call). A release therefore only had to carry the right numbers. Two independent ways that happened, both reproduced live:
  - *Alias noise.* Kitsu carries the short alternative title `"Beyond Time"` for this show. Indexers full-text a query against the entire release name, group tag included, so `"Beyond Time"` matches `"[Anime **Time**] Frieren: **Beyond** Journey's End"` — on the group name plus one title word. Its Season 01 pack advertises no episode range, so it was kept on its season alone, and then cleared the file-list guard rail too, because Frieren's season 1 also has an episode 23.
  - *Number-shaped queries.* A *Dandadan* S2E9 request (absolute episode 21) had cached *Wind Breaker* S02E09, *The Dangers in My Heart* S02E09, *The Fable* - 21, *Cool Doji Danshi* - 21, and the *Undead Unluck* and *Dungeon Meshi* `01 ~ 21` packs, whose ranges happen to cover 21. Found across the cache: an *86* request matching releases whose CRC32 hash merely begins `86` (`[86B40025]`, `[86879217]`), *Nana* matching *Nanatsu no Taizai*, *Shiki* matching *Shikimori-san*, *Ben 10* matching *Benriya Saitou-san*, *Uzumaki* matching *Brave 10* through the group tag `[Diogo4D-Uzumaki10]`, and *Batman: Caped Crusader* matching *Shingeki no Kyojin* through `[PinkBatman]` and the uploader `(IAmBatman)`.

  Every keep now has to name the requested series, checked once ahead of all episode logic — which show and which episode are separate questions, and a proven episode range answers only the second. Cached search documents holding wrong-show results were purged after full backups, so the next request for each re-searches with the fix in place.
- **A noisy search query could stop the search before the precise ones ran** — the title-only phase decided it had done enough by counting the batch torrents an indexer returned, not the ones that survived matching. So the single `"Beyond Time"` query, whose only "batch" was the Frieren pack, ended the alias sweep at the first query and the show's remaining, more precise aliases were never searched. Both stopping rules now count torrents actually kept.

### Changed
- **Alternative titles too generic to identify a show are no longer searched** — the fallback alias set is whatever extra names the metadata sources happened to carry, and some are a single common word (Kitsu lists `"MAO"`), which an indexer matches against any release with those letters anywhere in it. Single-word latin aliases are dropped from the fallback query set only; a show's actual canonical, English and romaji titles are always searched however short they are, and native-script titles are exempt — they are the most precise queries there are on donghua and fansub indexers.

### Internal
- New `isUnnamedRelease()` / `releaseNamesSeries()` in `search/matching.js`, applied in `tryMatch` before `classifyTorrent`. A release names the series if a native-script alias appears in it outright; if a multi-word alias appears as a contiguous token phrase (`matchesMovieAlias`); if a distinctive one-word alias appears as a whole token; if the display title appears as a full-word phrase; or if the display title clears the existing similarity threshold. `matchesRequest`'s unused `filteredTitles` parameter is left alone — the gate now sits in front of it, so wiring it up would be redundant in a primitive with much wider reach.
- The gate **fails open** wherever the request holds no evidence that could have matched the release, because guessing there falls on legitimate releases as readily as on contamination. This is script-aware: a Japanese or Chinese alias can only ever appear in a native-titled release, so it is no evidence at all about a latin-titled one. *Special A* is the worked example — it survives tokenization as the bare packaging word *special*, and treating its Japanese titles (plus the romanization `"Supesharu ê"`, which no release group uses) as evidence let the gate confidently reject all five of the show's own releases on Nyaa.
- Thresholds were set against the live cache rather than by intuition, and three first attempts were wrong. A distinctive one-word alias needs four characters, not five: this arm compares whole tokens, so the short forms that collide do so *inside* longer words (`sth` within `Aesthetica`, `mao` within `Maou-sama`), which exact token equality already rejects — five silently dropped every plain `"[Group] Aria - 01"`. The display-title phrase arm needs three letters, not five: `"R-15"` and `"009-1"` reduce to words that are all under the token length floor, so no other arm can see them, and five dropped all ten genuine *R-15* releases along with the real *009-1*.
- `runTitleOnlySearch()` counts `matched` deltas instead of raw batch counts for both its early-exit rules.
- Alias pruning added to `buildMergedQueries()`, applied to `altTitleOnlyQueries` only.
- 31 new unit tests covering the Frieren and Dandadan regressions, one-word titles (*Aria*, *Kite*, *Grand Blue* vs *Grand Blues!*), short and numeric titles (*R-15*, *009-1*), titles whose identity lives in short words (*Special A*), native-script evidence, group-tag impostors, and the fail-open paths. Unit suite is 201 tests; full suite 209 passing.
- Validation methodology: the gate was replayed over the cached corpus with each entry's alias set resolved live rather than read from the (24h-TTL, mostly expired) metadata cache. Final pass over 100 distinct content ids kept 990 results and dropped 6, all six verified wrong-show. An earlier pass under-sampled Kitsu ids badly — the naive `:\d+(:\d+)?$` key strip removes both trailing segments of `kitsu:48834:6`, collapsing every Kitsu entry onto the bare key `kitsu` — which is what hid the *R-15* regression; content keys now go through `parseContentId`.

---

## [2.6.1] - 2026-08-15 — Absolute-Episode Mapping Fix for Season-Split Donghua IDs

### Fixed
- **Season-split donghua/anime requests (e.g. `S8E10`) could resolve no absolute episode at all, silently disabling absolute-numbered matching** — `computeAbsoluteEpisode` needed a single metadata source to report contiguous, nonzero episode counts for every season before the one requested. TVDB's contribution was structurally broken: `resolveTvdbId` read per-season counts off `season.episodes`, a field TVDB never populates — it always returns episodes as one flat, series-level list, never nested under each season — so every TVDB-derived season count was silently `0`, for every show, not just one. Reproduced live on *A Record of a Mortal's Journey to Immortality* (`tt12879782`): TMDB reports the whole 206-episode run as a single season (nothing to sum for a season 2-8 request), and TVDB's season 1-8 entries all showed `episodeCount: 0`, so `S8E8`/`S8E10` resolved to `absoluteEpisode: null`. `resolveTvdbId` now derives real per-season counts by grouping TVDB's flat episode list itself, and additionally exposes a direct per-episode `episodeAbsoluteMap` built from TVDB's own `absoluteNumber` field, which `computeAbsoluteEpisode` now prefers over summing counts — more reliable, since it already accounts for specials/bonus episodes interleaved into the regular run (this show's S8E14 carries `absoluteNumber` 190, four higher than a clean sum through season 7 would give).
- **TsukiHime (and other `advancedQuery` providers) never searched for a donghua's absolute episode number, even once it resolved correctly** — `buildConsolidatedLevels`, the query builder these providers use instead of the season/episode progressive search, had no equivalent of the absolute-numbered "Level 0" query the progressive path already had. It now injects the same level. Confirmed live: the same `S8E8` request went from 0 streams to 1 on TsukiHime after this fix.
- **Per-episode metadata (titles, overviews, and the season/episode lookup map) came back empty for any show reached via a plain IMDB ID that cross-references into TVDB** — `fetchEpisodesForSource`'s TVDB/TMDB branches used the ID parsed directly off the incoming content string, which is `null` when the request arrives as `tt...` rather than `tvdb:...`/`tmdb:...`, even though the already-resolved metadata object carried the real cross-referenced ID. This silently returned an empty episode list (and therefore an empty season/episode map) for exactly the request shape Stremio sends most often. Both branches now prefer the resolved ID already present on the metadata object.

### Internal
- `computeAbsoluteEpisode()`, `buildConsolidatedLevels()`, `fetchEpisodesForSource()`, and `resolveTvdbId()` all touched; new offline + live integration coverage in `test/integration/mortal-journey-absolute.test.js`.

---

## [2.6.0] - 2026-08-13 — Movie Search Overhaul: Right Film, Right Size, No Phantom Episodes

### Fixed
- **Movies that plainly exist on a source returned no streams at all** — reproduced end-to-end with the Shrouding the Heavens film (`tt41780250`), which has two dedicated releases on Nyaa and yet came back empty on every provider. Four independent faults were stacked on the movie path, each sufficient on its own to produce zero results:
  - *Only one search query was ever issued to an advancedQuery provider (TsukiHime).* A movie's consolidated query terms are built from `displayTitle` plus `titleOnlyQueries`, but that field was hardcoded empty for movies — so all eleven resolved title variants went unsearched and the single query actually sent was the canonical Kitsu title, a romanization (`"Zhe Tian Movie: Bei Guan Zhan Wang Teng"`) that no release group uses. Its aliases (`"Shrouding the Heavens"`, `"The Imperial Path"`) match immediately and are now searched.
  - *The `" movie"` search tag was appended unconditionally,* producing `"Zhe Tian Movie movie"` and `"Shrounding the Heavens Movie movie"` — a doubled token no release name carries, verified to return zero results against Nyaa, AnimeTosho and TsukiHime alike. A title that already ends in *movie*/*movies* is now left alone.
  - *Nyaa's category never widened for this show.* Nyaa is searched in the English-translated category first and only broadened to all-anime when the first attempt returns fewer than three results — but on a long-running franchise the series' own weekly episodes fill that quota while the film itself sits outside the category. Searching `"Shrouding the Heavens"` returned nine episode releases (so no widening) and not one of them was the film; the broader category returns both of its releases. Movie requests now always widen, since a film is a single target and a result count says nothing about whether the right one was found.
  - *The matcher scored candidates against the canonical title alone,* ignoring the twelve-alias list it was already being handed. Worse, tag-stripping reduces a release name built entirely of bracketed segments — which is how GM-Team files every donghua — to the empty string, scoring a flat 0.000. The two real releases scored 0.000 and 0.125 against a 0.3 threshold and were discarded. Movie matching now also tests the request's known aliases, and title comparison falls back to the bracket-delimited text when stripping leaves nothing behind.
- **A movie request could return the entire TV series** — a film and its parent series share a title, so title evidence alone cannot separate them, and matching the series name returned all 174 episodes as "the movie". Releases pinned to a specific episode (`"… - 170"`, `"…[174]…"`) or to a plain episode range (`"E001-E074"`, `"Ep. 75-101"`) are now rejected for movie requests. Packs that advertise the film alongside the episodes are kept — on TsukiHime and AnimeTosho such a pack is the only place some films are indexed at all.
- **Requesting one film of a franchise returned all of them** — asking for Kimetsu no Yaiba's *Mugen Train* (2020) surfaced *Infinity Castle*, *Mugenjou-hen* and *Akaza Sairai* releases, which then outranked the requested film on seeders and pushed it out of the results entirely (one of eight results was the right film; now all eight are, and a provider that returned none now returns three). The arc gate that already prevents this for series — rejecting a release that contains a requested title *plus* significant words no alias accounts for — was never reached on the movie path, and now runs there too. It also correctly rejects the TV re-cut of the same story (`"Mugen Train Arc"`, `"Mugen Ressha-hen (Broadcast Ver.) - TV + SP"`), which is not the theatrical film a movie request asked for. Three further leaks had to be closed for it to hold:
  - A metadata alternative title of literally `"The Movie"` reduces to the single word *movie* and matched every movie-labelled release in existence — it pulled `"[QTS] Saint Seiya THE MOVIE Blu-ray BOX Movie 3"` into a *Your Name* request. Aliases made up entirely of packaging words (*movie*, *film*, *gekijouban*, *ova*, *special*) are now treated as identifying nothing.
  - Title similarity scores the fraction of a title's words found in a candidate, so a one-word title makes every release containing it a perfect match: `"Kimi no Na wa."` reduces to just *kimi* and scored 1.00 against `"[Trix] Ride Your Wave … Kimi to, Nami ni Noretara"`. That route now needs at least two significant words; shorter titles fall through to the stricter phrase test.
  - One release survived every check because the title parser reads a `|`-delimited multi-language name from the wrong end — `"… Infinity Castle Part 1 … | Клинок… | En/Ru [субтитры][HEVC 10bit]"` parses to the title `"En"`, leaving the arc gate nothing to compare. When the parse comes back too thin to judge, the segments are now re-parsed and the richest one used.
- **A film bundled inside a season pack or film collection reported the whole pack's size and started playback on the wrong file** — a 2.97 GiB movie was advertised as 189.25 GiB and began at file 0, which is episode 1. The pack's file list is now read and the film's own file located, so its size, filename and index are the film's; the containing pack is still credited on its own line so the download source stays clear. The two pack shapes hide the film differently and both are handled: in a season pack every file is named for the series, so the movie label is the only distinguishing mark; in a film collection nothing says "movie" (they are all films), so the requested title picks it out — *Suzume* correctly resolves to its own 3.59 GiB file inside an eight-film Makoto Shinkai collection rather than the largest file in it. When the film's file cannot be singled out the stream is dropped rather than served, because a movie stream that plays episode 1 is worse than no stream: this refuses a twelve-film Blu-ray box where none of the marked files matches the request, and refuses a film box's single movie-labelled file when that film isn't the one asked for.
- **A season pack whose title advertises nothing could be served as a movie** — `"Zhe Tian - Shrouding the Heavens"` looks innocuous, names no episodes and no film, and is a 135 GiB pack holding 106 episodes with no film in it. A movie candidate whose title claims nothing either way now has its contents inspected, and is dropped when the files turn out to be a run of episodes. Releases that name themselves a film are taken at their word and skip the inspection entirely, and an inconclusive inspection keeps the stream — so a film released with extras, a second encode or a full Blu-ray structure (several video files, but no episode run) is unaffected.
- **Every movie stream displayed `"📜 Season 1  Episode ?"`** — the description template rendered the season/episode line unconditionally and filled it from its placeholders. Movies now omit the line.

### Changed
- **Movie searches now also query bare titles, not only `"<title> movie"`** — plenty of films are never tagged *movie* in the release name at all (GM-Team files donghua films as `"遮天剧场版 …"`, and series packs bundle a film as `"Episodes 1-169 + The Imperial Path Movie"`), so a tag-only query set could not see them. On providers that take one query at a time the tagged variants are tried first and the bare ones only if those come back short. On a provider that takes a single combined query per level the order is reversed — leading with the tag both misses films that never carry the word and pulls multi-film "Movie Collection" packs forward, measured live as dropping *Your Name* and *Suzume* from twenty results to nine.

### Internal
- `buildConsolidatedLevels()` now returns a movie-specific two-level plan (bare titles, then tagged) instead of falling through the episode/season levels with nothing in them; movie title variants and the `" movie"` tag rule are extracted as `movieTitleVariants()` and `withMovieTag()`.
- New `locateMovieFileInPack()` resolves a film's file within a pack and reports `located` / `single` / `unresolved` so callers can distinguish "nothing to disambiguate" from "cannot tell", plus `isEpisodic()` and `looksLikeFilmTitle()` for the inspection decisions above.
- `matchesMovieAlias()` requires alias words to appear as a contiguous phrase rather than as a bag of words — `"The Imperial Path"` had both its significant words present in a SPY x FAMILY episode titled `"Behind the Scandal The Path to an Imperial Scholar"`, reversed and eight words apart. Its two-word minimum is relaxed to one word only when choosing between files inside a torrent that already matched, where several real film titles carry just one significant word.
- 39 new unit tests across query building, movie matching, franchise disambiguation, in-pack film location and stream descriptions — all offline, with pack file lists injected rather than fetched. Suite is 179 tests.

---

## [2.5.2] - 2026-08-09 — Nuvio Metadata Compatibility, English TVDB Descriptions

### Added
- **Meta responses now carry Nuvio/Cinemeta-compatible key aliases** — some third-party Stremio clients (e.g. Nuvio) read a richer, differently-named set of keys than the minimal Stremio SDK spec (`genres` alongside `genre`, `imdb_id`/`moviedb_id`/`tvdb_id`, `language`, `country`, `runtime`, `slug`, `behaviorHints`, a string-formatted `imdbRating`). These are now added as parallel keys — never replacing the existing Stremio-native fields — and only when a metadata source actually provides the value, so nothing is padded with empty placeholders.

### Fixed
- **TVDB descriptions for donghua and other non-English-origin shows were returned in the original language instead of English** — the addon already fetched an English translation of the title for these shows, but was discarding that same translation's English overview and falling back to TVDB's default (original-language) description. Descriptions now consistently use the English overview when available, for both TVDB series and movies.
- **TVDB episode titles/overviews for those same shows stayed in the original language even after the fix above** — episodes were read from the series' `/extended?meta=episodes` response, which (confirmed live against TVDB) ignores a `lang` param entirely and never returns translated episode names, no matter what else is fixed around it. TVDB series metadata no longer sources episodes from that embedded list at all; it now always goes through the dedicated `/episodes/official/eng` endpoint, which *does* return real English names/overviews in bulk (no per-episode calls needed).
- **Episodes from TVDB, TMDB, or Kitsu could show up empty in some third-party clients even though Stremio displayed them fine** — video/episode objects built from these three sources only carried each API's native field names (`number`, `name`, `aired`, `image`) and were missing the fields the Stremio video schema itself expects (`id`, `episode`, `released`). Stremio's own client tolerates the gap, but stricter clients like Nuvio don't fall back and rendered no episodes at all. Every episode-producing path (including the paginated fallback fetchers) now emits both key styles side by side.

---

## [2.5.1] - 2026-08-03 — Batch Episode Accuracy Fix, Strict Source Selection

### Fixed
- **Some batch packs could play a different episode than the one you requested** — the shortcut that maps a requested episode to a file's *position* (using a metadata provider's own episode count) only checked that the picked file's season matched; it never checked that the file's own episode number matched too. On packs where the uploader's file order drifts from the metadata provider's count (extra specials, movies, or spin-off episodes mixed into a "complete" pack), this could serve the wrong episode's file while still labeling it with the episode you asked for. The addon now also checks the picked file's own episode number before trusting it, so what actually plays matches what you requested — not just the label.
- **A rare mismatch could pick the wrong torrent for Season 1 requests** — A safety check meant for shows with multiple seasons (matching a raw's absolute episode number when the season-relative number doesn't line up) was also running for Season 1, where it isn't needed and could occasionally lock onto the wrong episode's torrent if a metadata provider's own numbering was slightly off. It's now only used where it's actually needed (Season 2 and up).

### Changed
- **Torrent source selection is now strict** — Choosing a source (AnimeTosho / AniRena / TsukiHime) no longer silently switches to a different one behind the scenes if your chosen source has nothing for that episode. Only your selected source is used, so a stream's provider label always matches what you picked.

---

## [2.5.0] - 2026-08-01 — Bracket-Wrapped Title Fix (Oshi no Ko and similar)

### Fixed
- **A show whose own canonical title is fully bracket-wrapped (e.g. Kitsu titles Oshi no Ko literally as `"[Oshi no Ko]"`) served completely unrelated shows' torrents instead** — `stripTags()`, meant to strip release-tags like `[SubGroup]` or `(1080p)` off a torrent title, was also applied to the show's own metadata title when building S##E## priority-search patterns. When the entire title happened to be one bracketed span, stripping deleted it outright instead of just the brackets, turning a query like `"Oshi no Ko S01E01"` into a title-less `" S01E01"`. Sent to a provider, that title-less query matches *any* show's own "S01E01" release — confirmed live against TsukiHime, and found already cached in production (17 documents, ~1000 wrong-show torrent entries, all under this one series). `stripTags` now unwraps a title that is itself one full bracket/paren span instead of deleting it, while still stripping ordinary tags that don't cover the whole string. The stale poisoned cache entries for this series were purged (after a full backup) so the next request re-searches with the fix in place.
- **Same collapse-to-empty bug in `titleSimilarity`'s title normalizer** — a separate, duplicate bracket-stripping regex had the identical flaw: a movie or donghua title that's itself one bracket-wrapped span always scored a similarity of 0, silently matching nothing rather than the wrong thing (movie matching and the donghua weak-candidate fallback both gate on this score). Now reuses the fixed `stripTags` instead of re-implementing bracket stripping a second time.

### Internal
- **Consolidated-query (TsukiHime/advancedQuery) episode level split into exact vs. loose sub-levels** — the precise `"Title S01E01"` pattern and the bare `"Title 1"` fallback suffix (for releases with no S/E tag at all) were previously OR'd into one query. A low-selectivity bare-number term can swamp a recency-sorted provider's result window and bury every precise match — verified live, an OR'd query returned zero exact S01E01 hits in the top 100 results. The exact pattern is now queried first, with the loose suffix only used as a fallback when it doesn't find enough. The level-building logic was extracted into a pure `buildConsolidatedLevels()` function so this ordering is unit-testable without a network mock.

---

## [2.4.0] - 2026-07-26 — Season-Relative/Absolute Aliasing Fix, Return Cached Streams Toggle

### Fixed
- **Wrong season entirely served for multi-season shows when the absolute episode couldn't be resolved** — When metadata didn't yet map a requested season/episode to its true absolute episode number (a still-airing season, or a metadata gap), the matcher fell back to treating the season-relative episode number as if it were absolute. For any season above 1 this let an unrelated earlier-season batch satisfy the request purely by numeric coincidence — e.g. requesting S8E8 of a long-running donghua returned a completely different show's own "episode 8" from an unrelated "Season 1 Remastered — Episodes 1-21" pack, because 8 falls inside that pack's own range and it carries no season tag to disagree with. Season-relative numbers are now only trusted as absolute for season 1 (where they're identical by definition); for S2+ with no resolved absolute episode, the batch is dropped instead of guessed. Applied consistently across the keep/drop matcher, the weak-donghua `.torrent` verifier, and the pre-cache guard rail.
- **Multi-season complete packs could mislabel a bonus/spin-off episode as the requested season** — A pack whose title spans a season *range* (e.g. "Season 1-4 + Kanketsu-hen + OVA + Junior High") had that range collapsed to a single inferred season number, which was then silently inherited by every season-tagless file inside the pack — including unrelated bonus content. Concretely, requesting Attack on Titan S1E1 against a pack like this could surface the "Junior High" spin-off's own episode 1 instead of the real season 1 premiere, since both looked equally valid once the spin-off wrongly inherited "Season 1". Per-file season resolution now also reads the file's containing folder name (many packs organize by per-season/per-arc subfolders), and no longer inherits a season from a pack title that names more than one season.
- **Batch resolver's positional fallback used the wrong episode number** — When a batch's files carry no season markers at all, the resolver's last-resort "guess by position" fallback used the raw season-relative episode instead of the already-resolved effective episode used everywhere else in the same function, which could pick the wrong file for an absolute-numbered pack.

### Added
- **"Return Cached Streams" setting** (Configure → Performance Tuning) — on by default, matching current behavior. Turn it off to always run a full fresh search instead of serving from the episode cache. A fresh search that comes back empty now also deletes the stale cache entry for that episode, so this doubles as a manual, per-episode fix when an addon is serving a wrong/stale cached stream — no need to wait for the 30-day cache TTL.

### Internal
- **`scripts/clean-episode-cache.js` scope extended** — previously only detected wrong-episode *open-ended* (no range in the title) packs. Now also catches the season-relative-aliasing bug on *ranged* batches (the exact class of bug described above), and drops any cached batch whose title declares an explicit season disagreeing with the cached request. File-list verification now races three independent sources — itorrents.org, nyaa.si's per-torrent file-list page, and limetorrents.lol's search + detail pages — using whichever responds first with real data, so a single source being down or throttled no longer leaves a pack unverified.

---

## [2.3.0] - 2026-07-21 — Correct Qualities, Verified-Before-Cache Batches, Truncated .torrent Recovery

### Fixed
- **Missing 4K / second-tag qualities on recent episodes** — Releases that glue a quality tag right after the group with no separator (e.g. `[BonoboSubs][4k]Renegade Immortal - 仙逆 Xian Ni - 150`) had their episode number lost entirely, so that quality was silently dropped — most visibly the 4K versions of a show's newest episodes. The parser now relocates the extra leading bracket, so the title, episode, and resolution all parse and every quality is returned.
- **Wrong-episode season/complete packs no longer cached or served** — Open-ended packs with no episode range in the title (e.g. `"… S01 Complete"`, `"[Batch]"`) were kept optimistically and the episode was only checked at play time. They are now verified against the pack's actual file list **before** anything is cached: a pack is kept only if it truly contains the requested episode, and an unconfirmable pack is dropped rather than guessed. This stops flat-catalogued donghua (e.g. Renegade Immortal S1E150, where fansub "S01" packs are only the first cour) from surfacing a pack that can't contain the episode.
- **Large `.torrent` file lists recovered from truncation** — Public `.torrent` caches cut large torrents off partway through their `pieces` field, which made a strict decode fail (`Missing delimiter`) and the whole pack fall through to a guess. Because the file list sorts before `pieces`, a truncation-tolerant decoder now recovers it, so big batch packs get verified instead of skipped.
- **Volume/edition batches no longer mistaken for a different arc** — The arc filter treated packaging words like `"(Books 1-5)"` and Initial D's `"Stages 1-6"` as if they named a different arc, dropping legitimate batches of the requested series. Structural words (book/volume/part/season/cour/complete/batch/stage) are now ignored by the arc check, while genuine arc/sequel names (Alicization, Progressive, "Fifth Stage") are still rejected.

### Internal
- **Episode-correctness guard rail moved ahead of the cache** — the keep/drop pipeline now confirms batch episodes (`filterConfirmedBatches`) before `storeEpisodeCache`, so only correct-episode torrents are persisted; the verified file list is reused at stream-build to avoid a second fetch, and the batch resolver never positional-guesses an open-ended pack.
- **`parse-torrent-title` fork bumped to v1.4.2** — glued leading-bracket normalization; the scraper still carries no title-parsing regex.
- **New maintenance script `scripts/clean-episode-cache.js`** — retroactively purges wrong-episode open-ended packs cached before the guard rail existed. Verifies each pack against its real file list (metadata-independent), removes only what it can prove wrong, keeps everything unverifiable, and is dry-run by default.

---

## [2.2.0] - 2026-07-19 — Correct Episode From Batches, Better Donghua Raw Coverage

### Fixed
- **Batch packs no longer serve the wrong episode** — When a batch/season pack's file list couldn't be read (a remote `.torrent` fetch failing or timing out), the addon fell back to guessing a file position from the season-relative episode number. For absolute-numbered or multi-season packs this could serve a completely unrelated episode. The addon now only serves that positional guess when it can actually be bounded by the pack's episode range; otherwise it skips the pack instead of returning the wrong video.
- **`.torrent` file-list fetch reliability** — The underlying downloader used to read a batch's file list could fail to parse valid `.torrent` files pulled from public caches (a spurious "Missing delimiter" error), even when a normal download of the same file worked fine. Fetching now fetches the raw bytes directly and decodes them, recovering file lists that previously always fell through to a guess.
- **Episode ranges with spaced dashes now detected** — Titles like `"Episodes 77 - 124"` (as opposed to `"Episodes 77-124"`) weren't recognized as a range, so packs that didn't actually contain the requested episode weren't being rejected.
- **Donghua raws indexed only under their native title are now found** — Search now also queries the show's native (Chinese/Japanese-script) title, not just the English/romaji variants, surfacing fansub packs — including GM-Team-style raws — that were previously invisible.
- **Absolute-episode mapping no longer guesses across a data gap** — If a metadata source was missing episode counts for one of the seasons before the requested one, the addon used to sum what it had anyway and silently compute a wrong absolute episode, which made every raw for that request fail to match. It now requires complete season-by-season coverage before computing an absolute episode, and falls back safely otherwise.
- **Provider fallback for empty results** — If the configured source returns nothing for a specific episode, the addon now also tries TsukiHime (a public, keyless index with strong donghua coverage) before giving up.

### Internal
- **Test suite consolidated and made runnable** — `npm test` previously only ran one of nine test files; the rest were either disconnected from the test runner or console-log scripts with no assertions. Tests are now organized as `test/unit` (offline, always run) and `test/integration` (real API/DB workflows that skip cleanly when a key, network, or database isn't available, instead of failing or hanging). Added `npm run test:unit` / `npm run test:integration` to run either independently.

---

## [2.1.0] - 2026-07-16 — Accurate Arc, Season & Batch Matching

### Fixed
- **Wrong arcs and spin-offs no longer show up** — Searching a series (e.g. "Sword Art Online") no longer returns unrelated arcs, sequels, or spin-offs (Alicization, Gun Gale Online, Progressive, Ordinal Scale). A title now matches only itself and the alternate names your metadata knows about, so results stay on the show you actually asked for.
- **Season & complete packs now play the right episode** — Full-season and complete packs that don't spell out an episode range in their title (e.g. "Sword Art Online S1 [Complete]", "Kimetsu no Yaiba (batch)") were being dropped entirely. They're now matched and expanded to the requested episode.
- **TsukiHime batches no longer vanish** — TsukiHime doesn't report seeder counts, and those torrents were being incorrectly removed by the seeder-threshold filter. They're now kept (seed count simply shows as unknown).
- **Second-season titles detected correctly** — Releases that mark the season with a Roman numeral (e.g. "Sword Art Online II", "Mob Psycho 100 II") are now recognized as Season 2 instead of defaulting to Season 1.

### Improved
- **TsukiHime search accuracy** — TsukiHime now uses the same prioritized, most-specific-first search as the other providers (exact episode first, broadening only if nothing is found), pulling up to 100 candidates per query and filtering them locally the same way, so it benefits from the same matching accuracy and caching.

### Changed
- **"Max Batch Expansions" setting removed** — How many batch/season packs get expanded is now governed by your **Max Streams** setting instead of a separate cap (previously fixed at 3). You'll now get up to your chosen stream count from batch packs rather than an arbitrary limit.

### Internal
- All torrent-title parsing was consolidated into the `parse-torrent-title` fork (v1.4.0) — arc/subtitle separation, Roman-numeral seasons, batch/range detection (including bare and 4-digit ranges), absolute-episode range hints, and resolution/codec normalization. The scraper itself no longer carries any title-parsing regex.
- Provider layer restructured: a provider **registry** (adding a source is one file + one line), a pluggable **search-phase engine**, a dedicated **matching** module (arc gate + classification), a shared **weak-batch** registry, and a single home for remote `.torrent` file-list fetching.
- TsukiHime is no longer a hard-coded special case — it flows through the generic engine via a declared `advancedQuery` capability.

---

## [2.0.0] - 2026-07-07 — AniScraper Rebrand, TsukiHime Provider, Nyaa.si Paused

### Added
- **TsukiHime provider integration** — New torrent source (api.tsukihime.org), searchable independently of AnimeTosho and AniRena, with its own rate-limited request worker (2 concurrent requests) that strictly follows the rate-limit headers returned by the TsukiHime API.
- **Two-call search flow for TsukiHime** — Every title variant (English, Romaji, aliases) plus the target episode number are combined into a single unquoted OR query, finding the specific episode in one call instead of a multi-phase waterfall. If that finds nothing, a broader title-only OR query (classified locally from up to 50 candidates) runs as a fallback for obscure titles or unusual episode numbering.

### Changed
- **Renamed to AniScraper** — What started as a Nyaa.si-only scraper has grown, by user feature request, into a multi-source anime torrent aggregator (AnimeTosho, AniRena, and now TsukiHime). The addon is now **AniScraper** (formerly Nyaa Anime), with a new Stremio addon ID (`community.aniscraper.anime.addon`) and package name (`ani-scraper-stremio-addon`). Existing users should update/reinstall through the AniScraper configure page.
- **Nyaa.si temporarily disabled** — Nyaa.si has been paused as a selectable torrent source while we work through some issues on that provider. Existing saved configurations using Nyaa.si no longer scrape or reroute; update to AniScraper and pick AnimeTosho, AniRena, or TsukiHime from `/configure`. This is expected to be temporary.
- **AnimeTosho domain updated** — `animetosho.org` has been deprecated and its registration has resigned; the addon now talks to `animetosho.xyz` instead.

### Internal
- Every torrent provider now strictly enforces the rate limits its own API reports via response headers (remaining-quota and reset/retry-after), independent of the other providers' limiters — no provider's worker queue can ever throttle or be throttled by another provider's traffic.
- **AniRena rate limiting is now scoped per API key** — Previously all users shared one process-wide limiter, incorrectly pooling independent users' independent 60-req/60s quotas together. Each API key now gets its own lazily-created limiter with idle eviction.
- **MongoDB database renamed** — `nyaa-scraper-cached-requests` → `aniscraper-db`. Run `npm run migrate-db` before deploying to carry over existing caches; `MONGODB_DB_NAME` can point back at the old name to roll back without a code change.

---

## [1.9.3] - 2026-05-25 — Stream & Cache Bug Fixes

### Fixed
- **Season packs playing the wrong episode** — When streaming from a batch torrent (e.g. a full-season pack), the addon could serve the wrong video file — for example, clicking Episode 15 would play Episode 1. This is now resolved; the correct file is always selected.
- **Cached stream URL pointing to wrong episode** — After watching one episode, the cached download link could bleed into other episodes from the same pack. Each episode now stores its own link independently.
- **Resolve errors crashing some streams** — A crash (`isBatch is not defined`) could occur when the addon tried to cache a freshly resolved stream. This affected all debrid services and is now fixed.
- **Real-Debrid "hoster unavailable" not retrying** — When Real-Debrid returned a temporary hoster error (code 19), the addon was not retrying as intended. It will now correctly retry with a fresh link.

### Internal
- Resolved URL cache redesigned: each cached URL now lives directly on its torrent entry rather than in a separate root-level map, making cache scoping clearer and preventing cross-episode contamination.
- Two database migrations run automatically on startup to clean up old cache data and ensure a fresh start under the new schema.

---

## [1.9.2] - 2026-05-24 — Late-Season Donghua Batch Fix - 2026-05-19 — AniRena Integration and more Debrid Support

### Added
- **AniRena provider integration** — Implemented server-side token exchange and POST-based search so the provider acquires, caches, and renews bearer tokens (handles x-new-token) to keep searches authenticated and reliable.
- **Added AniRena API tests** — Added tests validating AniRena token acquisition, the POST search flow returns magnet links, and the token renewal behavior (covers token endpoint and search request/response cycle).
- **Debrid-Link integration** — Full support for Debrid-Link's seedbox service with automatic torrent management, file selection, and smart caching of download links.
- **Premiumize integration** — Full support for Premiumize with both instant downloads (directdl) and queued transfers, automatically choosing the fastest method.

### Fixed
- **Debrid-Link now correctly handles single-file torrents** — Previously would fail to find files because it was looking in the wrong place. Now correctly selects the video file.
- **Debrid-Link uses the correct API endpoints** — Fixed parameter names and request formats to match Debrid-Link's v2 API exactly.
- **Better error messages** — When a service is unavailable or has a problem, you'll now see a more specific error message (rate limit, auth failure, service down, etc.) instead of a generic failure.

### Internal
- Both services now properly detect when servers are overloaded, quotas are exceeded, or authentication has failed, and respond accordingly.
- Enhanced logging for debugging API issues.
- Removed dependency on Debrid-Link's disabled `/seedbox/cached` endpoint by using the torrent list instead.

### Changed

- **Resolution filter semantics** — The resolution controls in Optional Settings now act as an *exclusion* list: checked resolutions will be hidden from results. The `resolutions=` URL parameter now contains the list of *excluded* resolutions and is only included when non-empty. If you use a previously saved configuration URL, please open the configure page and re-save your settings so they are updated to the new format.

- **Removed language-detection UI & filtering** — The optional "Title Languages" setting and `franc`-based automatic language detection have been removed. The addon now considers all title variants when searching (no language-based filtering), improving recall on Nyaa results.

---

## [1.9.0] - 2026-05-18 — Flat-Catalog Donghua Matching & Test Suite

### Improved
- **Flat-catalog donghua (e.g. Apotheosis, Swallowed Star) now correctly returns streams for all seasons** — Catalogs that expose all episodes as Season 1 but whose fansub releases are internally split by season (S2, S3, …) were previously dropping matches due to a season mismatch. The provider now reads absolute-episode range hints embedded in batch titles (e.g. `(053-062)`) and accepts any torrent whose range covers the requested episode, regardless of the fansub's internal season numbering.
- **Bare full-season packs (e.g. "Apotheosis S2") matched correctly via pre-seeded hints** — When a bare season pack appears in search results alongside sub-packs that carry explicit absolute-episode ranges, the addon now pre-seeds a hint map before classifying any result. This means the bare pack is matched even when it appears *before* the sub-packs in Nyaa's result order.
- **Absolute range hints require 3+ digit numbers** — Two-digit per-season ranges (e.g. `(01-52)`) are intentionally ignored to avoid ambiguity with per-season episode counts; only 3-digit+ numbers (e.g. `(053-062)`, `(001-052)`) are treated as absolute hints.
- **Phase 2 query uses source episode for absolute-episode donghua** — When `absoluteEpisodeMode` is active (catalog reports S1 but the actual season is higher), the Nyaa search query now appends the source-season episode number (e.g. `10`) rather than the absolute number (e.g. `62`), matching what fansubs actually write in their filenames.
- **`donghuaNeedsBatches` phase runs regardless of Phase 2 results** — The title-only batch search phase now always executes for donghua, so full-season packs are always considered even when individual episode torrents were already found.
- **Title-only results for donghua are fully processed** — All results from the title-only search phase (not just ones already flagged as batches) are now classified and absorbed for donghua, catching bare season packs and large full-season archives.

### Fixed
- **`matchesRequest` no longer accepts non-batch torrents with a null episode** — A non-batch release without a parsed episode number (e.g. a bare "Apotheosis S2" pack before the PTT fix) was being accepted for any episode request when the season matched. It now returns `false`, preventing 170 GB full-season packs from being served for a single-episode request.
- **PTT now parses bare `S\d` season suffixes** — Titles like "Apotheosis S2" or "Battle Through the Heavens S4" (no episode component) were not being recognised as season-tagged releases. A new handler correctly parses these as `season=N, episode=null, isBatch=true`.

### Internal
- **`extractAbsoluteRangeHint(title)` exported from `providers/index.js`** — Pure utility function that extracts a `{ start, end }` pair from a 3-digit parenthetical range in a torrent title. Usable independently for testing and future providers.
- **Regression test suite migrated to Node.js built-in test runner** — `test/main.js` now uses `node:test` + `node:assert/strict` (zero extra dependencies). Added 26 new regression tests covering `extractAbsoluteRangeHint`, `matchesRequest` null-episode guard, PTT bare-season handler, and the full pre-seed + classify simulation for flat-catalog donghua. Run with `npm test`.

---

## [1.8.7] - 2026-05-13 — Smarter RealDebrid Cache Handling

### Fixed
- **Cached CDN links now stay valid even after deleting a torrent from RealDebrid** — Previously, if you deleted a torrent from your RD library manually, the addon would ignore the still-valid cached CDN URL and add the magnet all over again. The liveness check now runs first: if the cached URL is reachable, it is returned directly without touching the RD library at all.
- **Single-episode torrents no longer retrigger a new RealDebrid magnet on every click** — For individual episode releases (where `fileIndex` is the absolute episode number, not a positional file ID), the addon was checking for an RD file with `id = fileIndex + 1` (e.g. id=140 for episode 140), which never exists — single-file torrents always have `id = 1`. This caused the addon to conclude the file was not selected and re-add the magnet on every stream click. It now correctly detects single-file torrents and only requires that any file is selected.

---

## [1.8.6] - 2026-05-11 — Faster Searches and More Reliable Stream Matching

### Improved
- **Up to 6× faster stream search** — Uncached episodes now return in ~6 seconds instead of up to 40 seconds. The addon reads metadata from cache before firing any Nyaa queries, so the correct episode number and title are known upfront and only 1–2 targeted searches are needed.
- **Multi-season shows now find the right episode on the first try** — For shows with multiple seasons (such as long-running donghua), the addon previously had to guess the absolute episode number, get it wrong, and retry. It now looks up the exact mapping (e.g. S4E55 → episode 140) from the metadata cache before searching, so no retry is needed.
- **Search stops as soon as there's enough, but never wastes a completed search** — If two queries run in parallel and the first one fills your stream limit, the second one's results are now still absorbed into the cache. That way future viewers get a fuller pool to choose from without triggering an extra search.
- **More torrents saved to cache per episode** — All filtered matching torrents are cached regardless of your personal stream limit, so viewers with different resolution or language settings can get results from cache without triggering a new search.
- **TVDB shows (including Chinese donghua) now match the right English title on Nyaa** — Shows whose primary title on TVDB is non-Latin (e.g. Chinese) now look up their English season name directly from TVDB and use it as the primary search term.
- **Episode lists are now always fetched for season/episode mapping** — A bug was causing the episode list to never load for many shows, which meant the season/episode → absolute episode map was always empty. This is now fixed.
- **Title aliases from multiple concurrent requests now merge instead of overwrite** — If two requests arrive at the same time with different title variants (e.g. English vs Romaji), both sets of aliases are saved to the same cache document so future lookups succeed regardless of which title is used.

### Fixed
- **Stale cached data (from a previous bug) is now replaced instead of reused** — If the cache contained wrong-episode torrents from an older buggy run, the addon would return them and produce no playable streams. It now detects this case and replaces the stale data with a fresh correct search.

---

## [1.8.5] - 2026-05-11 — Stremio v4 P2P Compatibility Fix

### Fixed
- **P2P streams now work correctly on Stremio v4, v5, and web** — P2P stream objects were setting `url` to a magnet URI, which Stremio v4 tries to HTTP-fetch and fails on. The `url` field is now only used for debrid/HTTP streams. P2P streams use `infoHash` + `sources` exclusively.
- **Batch P2P streams now navigate to the correct episode file** — `fileIdx` was being set to the episode's position in the full torrent file list (including subtitles, NFOs, etc.), which could be a large number unrelated to playback order. It now uses the position among video-only files, which is what Stremio's P2P client actually indexes over.
- **Single-episode P2P streams use `fileIdx: 0`** — Single-episode torrents always have one video file; hardcoding index 0 avoids any incorrect index derived from the title.
- **Debrid streams no longer include `infoHash`, `sources`, or `fileIdx`** — These fields are P2P-only. Debrid streams resolve entirely via `url` and do not need them.

---

## [1.8.4] - 2026-05-11 — Extended Cache Retention to 30 Days

### Improved
- **Episode cache now retained for 30 days instead of 3 days** — Popular episodes stay cached much longer, reducing redundant searches to Nyaa and improving response times for shows you watch regularly.
- **Existing cached data automatically updated** — Migration 009 refreshes all cached episodes on first run, extending their expiration to 30 days from migration time.

### Internal
- **Extended default `EPISODE_CACHE_TTL`** — Increased from 3 days (259,200 seconds) to 30 days (2,592,000 seconds), with environment variable override support via `EPISODE_CACHE_TTL_SECONDS`.
- **TTL index validation** — Migration ensures the MongoDB TTL index is correctly configured with `expireAfterSeconds: 0` for proper expiration behavior.

---

## [1.8.3] - 2026-05-09 — Smarter Search Pipeline & Stability Fixes

### Improved
- **Anime titles pattern are too messy** — Shows sourced from TVDB (rather than Kitsu) were routing all their title variants through the wrong search path, causing them to fire every alternate title query even after already finding results. Search now takes the correct path for each type of show, stopping as soon as it has what it needs.
- **Search stops earlier when you already have enough streams** — Previously, the addon would continue running searches even after finding more than enough results to fill your stream limit. All search phases now respect your configured stream count and stop immediately once it's met.
- **Alt title searches only run when they're actually needed** — The addon now runs primary title searches first (e.g. English + Romaji). If those find batch torrents (full-season packs), alternative title variants (abbreviated names, raw Japanese titles, etc.) are skipped entirely. This cuts out several unnecessary Nyaa requests per episode on well-known shows.
- **Batch torrent search stops after first success** — When scanning alt titles for batch packs, the search now stops as soon as any batch is found, rather than continuing through all remaining variants.
- **Slightly more human-like request timing** — Parallel Nyaa requests now include a small random delay (up to 250ms) so they don't all land at exactly the same millisecond interval.
- **More streams returned when your limit is above 15** — A server-side cap was incorrectly limiting stream processing to 15 regardless of your configured maximum. If you set your stream limit to 20, you now get up to 20 streams processed.

### Fixed
- **RealDebrid "unknown resource" error when clicking a stream after leaving mid-load** — If you clicked a stream and navigated away before it finished adding to RealDebrid, the torrent would be left in a broken half-added state. The next time you clicked the same stream, the addon would find this stale entry, fail to fetch its info (404), and crash the request instead of recovering. It now detects this situation and cleanly re-adds the torrent, so playback works on the next attempt.

---

## [1.8.2] - 2026-05-07 — Fixed Wrong Episode, Improved Cache Reuse & Stale Link Detection

### Fixed
- **Batch torrents now use cached download links correctly** — When streaming multi-episode torrents (batches), the addon was bypassing the download link cache even when already cached, forcing unnecessary re-downloads from RealDebrid. The cache is now reused properly for each episode's file within a batch torrent, avoiding redundant API calls and download delays.
- **Accurate RealDebrid availability labels on batch episodes** — The [RD+] indicator now correctly shows whether each individual episode file is cached, rather than marking all batch episodes cached if any file was. Filename-based matching ensures the right episode file is identified even when multiple torrents share the same hash.
- **Batch stream URLs now mapped correctly per episode** — Fixed an edge case where requesting episode 2 from a batch torrent could return episode 1's download link, causing playback of the wrong episode. The cache key now includes the specific file index, making it unique per episode within a batch.
- **Expired or deleted stream links no longer get served from cache** — The addon now checks whether a previously saved stream link is still reachable before using it. If the debrid service has expired or removed the link (which happens after a few days), the addon falls back to generating a fresh one rather than handing Stremio a broken URL. This applies to RealDebrid, AllDebrid, and TorBox.

### Internal
- **Download link cache now embedded in torrent entries** — Resolved URLs are now stored directly on each torrent's data array entry (with fileIndex as the key), replacing the old root-level map. This prevents cache contamination and ensures each episode gets its own link, especially important for batch torrents spanning multiple episodes.
- **Filename-based torrent disambiguation** — When multiple torrents share the same hash in RealDebrid, the addon now uses filename matching to select the correct one, falling back to the first match if filenames don't differ.
- **Removed batch-specific cache bypass** — No longer unconditionally skips the download link cache for batch file streams; the cache is now safe to use for all cached torrents since URLs are keyed by both hash and file index.

---

## [1.8.1] - 2026-05-06 — Cache Storage Optimization

### Improved
- **Dramatically smaller cache database footprint** — Episode cache now stores only essential fields (`infoHash`, `title`, `source`) instead of all metadata. This reduces per-torrent cache size by ~80% (from several hundred bytes down to ~100-150 bytes per entry). For popular episodes with hundreds of cached torrents, this means significant MongoDB storage savings across your entire cache.
- **Same speed, less storage** — No performance impact. Magnet link generation and stream building work exactly the same way. Faster database queries as a bonus since documents are smaller and fit more in memory.

### Internal
- **Selective Field Persistence** — Modified `storeEpisodeCache()` to destructure and exclude non-essential fields before writing to MongoDB. Only `infoHash` (needed for stream building), `title` (for logging/debugging), and `source` (to track origin) are persisted.
- **Stream Building Unchanged** — `getMagnetLink(infoHash)` in builder.js already generates full magnet URIs with tracker lists on-the-fly; no enrichment logic needed to move or refactor. This pattern already proved efficient through previous phases.
- **Database Schema** — Cache documents still contain metadata at the episode level (seriesId, season, episode, titleAliases, canonicalTitle) for matching and reference. Only per-torrent fields are slimmed down.

---

## [1.8.0] - 2026-05-05 — Faster Streams & Smarter Caching

### Added
- **Way faster on repeat episode requests** — If you already watched an episode recently, the addon now remembers and returns streams instantly without searching Nyaa again. This applies across sessions and even across different users on the same server.
- **Results cached per episode for 3 days** — Streams are now saved specifically for each episode of each show. Previously the cache could sometimes serve the wrong results for a different show's episode at the same season/episode number — that's now fixed.
- **Searches stop as soon as results are found** — When the first successful search comes back with streams, the addon immediately skips all remaining searches, cutting out unnecessary requests.

### Improved
- Removed an old redundant database collection (`searches`) that was no longer needed.
- Extended the cache lifetime from 4 hours to 3 days, so popular episodes stay cached much longer.

---

## [1.7.1] - 2026-05-01 — Patch: Stream Quality & Completeness Fixes

### Improved
- **Streams now sorted by your resolution preference** — Results are returned in the order you configured (e.g. 4K first, then 1080p, then 720p), regardless of whether a torrent is cached or not.

### Fixed
- **Missing streams when some are already cached** — When one torrent was cached on your debrid service and another wasn't, only the cached stream was returned. Now all available streams up to your limit are shown.

---

## [1.7.0] - 2026-05-01 — Feature: Performance Tuning, UI Overhaul & Search Fixes

### Added
- **Performance Tuning Controls in Optional Settings** — New configurable options to optimize stream search speed based on your preferences:
  - **Max Streams to Return** — Stop searching once you have enough good streams (default: 8). Lower values = faster results, higher = more comprehensive search.
  - **Batch Expansion Limit** — Control how many batch torrents to expand via debrid service API calls (default: 3). Reduces API usage on slow connections.
  - **Seeder Threshold** — Only expand batch torrents with minimum seeders (default: 100). Filters out low-quality sources automatically.
  - **Concurrent Searches** — Run multiple search queries in parallel (default: 3). Higher = faster discovery, lower = lighter server load.
  - **Parallel Metadata** — Fetch metadata and torrents simultaneously instead of waiting for metadata first (default: on). Saves 5-10 seconds per search.
- **Info buttons on advanced settings** — Every advanced setting now has a small info button that opens a popover explaining what it does and when to change it — no more guessing what "Batch Seeder Threshold" means.

### Improved
- **Faster Search Results** — Intelligent early-exit logic now returns results as soon as high-quality streams are found, instead of waiting to exhaust all search patterns.
- **Redesigned configure page** — The setup page has a completely new look: deeper dark background, ambient purple lighting, new heading font, and a monospace detail font for a cleaner, more polished feel.
- **Cleaner Configuration UI** — Performance tuning settings moved to a collapsed "Advanced" accordion section, reducing UI clutter and making basic configuration simpler.
- **Smarter Batch Filtering** — Low-seeder torrents in batch expansions are automatically skipped, reducing unnecessary debrid API calls and improving speed on rate-limited services.
- **Batch Expansion setting hides when not applicable** — The "Max Batch Expansions" option now only appears when Real-Debrid is selected, since batch expansion is an RD-only feature.

### Fixed
- **Batch torrents no longer crash when expanding** — A bug caused batch torrent expansion to fail entirely for all three fallback cases (no debrid key, no file list, no video files). These now correctly return a fallback stream descriptor instead of throwing an error.
- **Wrong season shown in stream descriptions** — Stream descriptions were always displaying "Season 1" for torrents that don't include an explicit season marker in their title (e.g. absolute-numbered releases). The season label now correctly falls back to the actual requested season.
- **Donghua episode releases not matched** — Releases using pipe-surrounded episode numbers like `| 024 |` (common in Chinese/Donghua fansub groups) were not being parsed as episodes, causing them to be silently dropped. These are now correctly detected and matched.
- **Non-batch torrents wrongly treated as batch packs** — Single-episode torrents from fansub groups that omit a season marker were being classified as season pack "batches" due to a defaulted season value. This caused them to bypass all episode matching and get dropped. Only torrents with an explicit season marker (e.g. `S01`) are now treated as season packs.

---

## [1.6.2] - 2026-04-28 — Patch: Debrid Service Bug Fixes

### Fixed
- **AllDebrid & TorBox: Wrong episode selected** — Multi-episode torrents now pick the correct episode instead of always grabbing the largest or wrong file.
- **AllDebrid & TorBox: Faster repeat plays** — Resolved video links are now cached, so replaying an episode skips the full debrid pipeline and starts faster.
- **AllDebrid: Blocked account error** — Properly handles `AUTH_BLOCKED` errors with a clear blocked-access response instead of crashing silently.
- **TorBox: Queued torrents stuck** — Torrents in a queued state no longer cause an error; they now correctly show as downloading.
- **TorBox: Zipped/archived torrents** — Correctly detects when TorBox only has a zipped version of a torrent and shows the "archive unavailable" message.
- **General: Network hiccups** — Temporary connection errors (timeout, reset) on AllDebrid and TorBox now return a "downloading" state instead of failing the stream entirely.

---

## [1.6.1] - 2026-04-26 — Patch: TorBox & Source Tag Fixes

### Fixed
- **TorBox Availability Check** — Fixed TorBox cached torrent detection using the wrong request format. Availability checks now work correctly, properly showing which torrents TorBox can deliver instantly.
- **Provider Tag Missing on Cached Results** — Torrents loaded from cache were losing their source tag (e.g. "nyaa"), causing them to display incorrectly in stream results. This is now preserved correctly.

---

## [1.6.0] - 2026-04-26 — Smarter Anime Search

### Added
- **Absolute Episode Matching** — When searching for a seasonal episode (e.g. Season 4 Episode 97), the addon now also searches by the absolute episode number (e.g. Episode 182) using cached series metadata. This finds fansub releases that use continuous episode numbering instead of season/episode notation.
- **Version Badge** — The configure page now shows the current addon version next to the Stremio badge. Click it to view what changed in this release.

### Improved
- **Search Result Caching** — Priority search patterns (like "Show S04" and "Show Season 4") are now cached for 5 minutes, reducing repeated requests to Nyaa for the same episode.
- **Nyaa Source Simplified** — Removed unreliable mirror fallback logic. Requests now go directly to nyaa.si for faster, more predictable results.

---

## [1.5.0] - 2026-04-26 — Multi-Debrid Support

### Added
- **TorBox & AllDebrid Support** — You can now use TorBox or AllDebrid as your debrid service in addition to Real-Debrid. Switch between services using the new dropdown on the configure page — no more manual URL editing.
- **API Key Validation** — The Install button now validates all your API keys (debrid, TVDB, TMDB) before showing install options. If a key is wrong or expired, you'll see a clear error message pointing to exactly which key failed and why.

### Improved
- **Cleaner Configuration URLs** — All service tokens now use consistent named parameters (e.g. `rd=TOKEN`, `torbox=TOKEN`, `ad=TOKEN`) instead of a legacy positional format. URLs are shorter and easier to read and share.

---

## [1.4.2] - 2026-04-26 — Performance Optimization

### Improved
- **Faster Stream Loading (Cache Miss)** — Optimized parallel processing of torrent fetches, metadata lookups, and Real-Debrid availability checks. Stream lists now appear in 8-12 seconds instead of 30-40+ seconds when cache isn't available.
- **Reduced Timeout Waits** — Stricter 10-second timeout on Real-Debrid operations prevents hanging indefinitely on slow connections. Falls back gracefully if RD is unreachable.

### Internal
- **Parallelized Batch Operations** — Multi-file torrent expansion now processes multiple torrents simultaneously instead of sequentially, dramatically reducing wait time.
- **Batched RD API Calls** — Real-Debrid info requests now batch (5 concurrent) instead of queuing individually, cutting availability checks from 20+ seconds to 3-5 seconds.
- **Background Metadata Fetching** — Series metadata fetches in parallel with stream building, improving responsiveness on first load.
- **Better Error Resilience** — One slow/failing torrent no longer blocks results from other torrents; degradation is graceful.

---

## [1.4.1] - 2026-04-26 — Patch: Resolution Filter Persistence Fix

### Fixed
- **Resolution Filter Not Prefilled** — Resolution checkboxes now correctly restore their unchecked state when loading a saved configuration. Previously all checkboxes defaulted to checked regardless of saved preferences.

---

## [1.4.0] - 2026-04-26 — P2P Streaming & Resolution Filters

### Added
- **P2P Direct Magnet Links** — Optional P2P streaming mode lets you watch without Real-Debrid. Direct peer-to-peer connections provide instant playback without needing to wait for RD caching. Also works with trackers for faster peer discovery. Toggle in Optional Settings (only shows when no RD key is configured).
- **Video Resolution Filter** — Choose which resolutions to include when searching (4K, 1440p, 1080p, 720p, 480p). Rejected resolutions are excluded from results entirely, so you only see streams that match your quality preferences. Uncheck unwanted resolutions in Optional Settings.
- **Responsive Optional Settings Modal** — Settings modal now uses 3-column layout on larger screens (while keeping 2 columns on mobile) for a cleaner, more spacious interface.

### Improved
- **Faster P2P Peer Discovery** — Magnet links are now enriched with comprehensive DHT trackers when P2P mode is active. Reduces initial peer connection time from 30+ seconds to 5-10 seconds, making P2P streams practical for immediate playback.
- **Configuration Management** — Added URL parameters `p2p=true` and `resolutions=4K,1080p,720p` for persisting user preferences in saved configurations.

### Fixed
- **P2P Mutual Exclusivity** — P2P toggle automatically hides when an RD key is present (they don't work together). If RD key is cleared, P2P option reappears. RD always takes priority if both settings somehow exist in a URL.

### Internal
- **Tracker Enrichment** — Integrated existing `getMagnetLink()` from httpHelper to provide optimal tracker set (fetches from GitHub + anime-specific trackers) instead of a static list.
- **Resolution Filtering** — Applied at stream-building stage, filtering after episode/season matching but before stream output for efficiency.

---

## [1.3.2] - 2026-04-24 — Internal Refactor

### Changed
- **Unified Logging System** — Implemented centralized logger with 3-level control (`LOG_LEVEL=1|2|3`), replacing scattered `console.*` calls with consistent `[DEBUG]`, `[INFO]`, `[WARN]`, `[ERROR]` prefixes
- **Code Quality Improvements** — Removed 10 dead code files and unused exports; reduced `src/` from 37 → 27 files through intelligent module merging
- **Reduced Comment Verbosity** — Trimmed verbose JSDoc and inline comments to short, actionable summaries per method/block
- **In-Memory Cache Abstraction** — Extracted `createMemCache()` factory for consistent cache behavior across all metadata sources (Kitsu, IMDB, TMDB, TVDB)

### Fixed
- **TVDB ID Resolution Bug** — Fixed `tvdbIdFromImdb()` using hardcoded URL; now correctly uses dynamic `imdbId` parameter

### Internal
- **Module Consolidation** — Merged 5 tiny utilities into `helpers.js`, 2 limiters into `limiters.js`, and 2 DB files into single `db/index.js`
- **Module Refactoring** — Absorbed `cache/metadataService.js` into `cache/metadata.js` for improved cohesion
- ✅ All verifications passing: app startup clean, logging levels functional, `console.*` completely removed, full test coverage

---

## [1.3.1] - 2026-04-22 — Patch Release

### Added
- **Smart Torrent Title Recognition** — The addon now uses intelligent matching to extract season, episode, and title information from torrent filenames. It recognizes Japanese, Chinese, and Korean anime titles with high accuracy, handles various naming conventions (different formatting, symbols, and abbreviations), and automatically corrects common parsing mistakes. This means it finds the right episodes even when torrent names are messy or use unusual formatting.
- **Accurate File Sizes for Batch Downloads** — When you select an episode from a multi-file pack, the addon now shows the estimated file size for just that episode instead of the whole pack size. Makes it easier to know what you're getting before downloading.
- **Improved Stream Information** — Stream listings now show clearer episode numbers, titles, file sizes, and quality information. Better icons and formatting make it easier to spot what you want at a glance.
- **Faster Streaming** — Direct playback links are now cached, so if you replay the same episode, it loads instantly without contacting Real-Debrid again.

### Fixed
- **Better Error Messages** — Real-Debrid errors are now properly reported, making it easier to troubleshoot issues if something goes wrong.
- **Metadata Without Real-Debrid** — Episode lists and metadata now work correctly even if you haven't connected a Real-Debrid account.
- **Stable Nyaa Access** — Nyaa mirror selection is more reliable and handles temporary outages better.

---

## [1.3.0] - 2026-04-14 — Patch Release

### Fixed
- **Real-Debrid Auto-Downloads** — Pack torrents (multiple episodes) no longer automatically download all files when browsing for streams. Only the episode you select will be downloaded, saving bandwidth and avoiding cluttered RD libraries.
- **Single Episode Selection** — When you click a specific episode from a multi-file torrent pack, Real-Debrid now only downloads that one episode instead of all files in the pack.
- **Faster RD Queries** — Eliminated duplicate requests to Real-Debrid that were slowing down stream playback. Each file is checked once, delivering results faster.
- **Nyaa Mirror Reliability** — Fixed mirror selection to handle temporary service issues better. Mirrors that are temporarily overloaded are now retried instead of being marked permanently broken.
- **Episode Number Detection** — Better at parsing episode numbers from filenames, including Asian formats (Japanese, Chinese, Korean characters). Works with more torrent naming styles.
- **Cleaner Stream Names** — Stream display names are now clean and readable, showing episode info instead of raw technical details.

---

## [1.2.1] - 2026-04-12

### Fixed
- **Batch Torrent Handling** — Multi-file torrents now return only individual episode streams, removing the redundant pack-level stream.
- **Stream Labels** — Improved stream labels to include the filename of the selected episode for better clarity.

---

## [1.2.0] - 2026-04-07 — Feature Update

### Added
- **Language Filtering & Detection** — Automatic language detection using `franc` library to identify content language (Japanese, English, etc.)
- **Cross-Reference Metadata** — Unified resolver supporting Kitsu, TVDB, TMDB, and IMDB with automatic cross-linking when primary source fails
- **Anime-Only Content Filtering** — Genre-based anime detection across all metadata sources:
  - Kitsu: Always anime-only database
  - TVDB: Checks for "Anime" genre tag
  - TMDB: Checks for "Anime" or "Animation" genre tags
  - IMDB/Cinemeta: Checks for "Animation" genre tag
- **Enhanced Movie Query Generation** — Improved search patterns for movie titles including alternate title combinations with separators (dash, colon)
- **TMDB Fallback for IMDB** — When Cinemeta fails to resolve an IMDB ID, automatically attempts TMDB cross-reference lookup for richer metadata
- **Genre Extraction** — All metadata sources now extract and normalize genre information for filtering and detection

### Improved
- Movie search queries now focus on individual title variants plus meaningful combinations, avoiding redundant permutations
- Metadata resolution now prioritizes anime-specific sources (Kitsu) and validates content type before processing
- Request pipeline includes early-exit for non-anime content at both stream and meta endpoints

---

## [1.1.0] - 2026-04-05

### Added
- **MongoDB Caching Layer** — Persistent request cache using MongoDB Atlas with TTL-based auto-cleanup:
  - Search cache (4-hour expiration) for torrent queries
  - Availability cache (5-day expiration) for Real-Debrid status
  - Automatic migration system for schema changes
- **In-Flight Request Deduplication** — Concurrent identical requests reuse same promise instead of duplicating API calls
- **Early-Stop Query Pattern** — Sequential title variant execution stops immediately on first successful result (≥1 match), reducing API calls by ~93%
- **Rate Limiting with Bottleneck** — Per-site request throttling:
  - Nyaa: Serial 1-per-second requests
  - AnimeTosho: 2 concurrent with 500ms gap
- **Worker Queue System** — Global stream handler queue and per-request query queue for controlled parallelism

### Improved
- Replaced redundant per-query individual caching with unified group cache keyed by MD5 hash of sorted queries
- Removed duplicate database entries: from 14+ documents per show to 1 group cache document
- Enhanced logging with prefixed categories (`[Stream]`, `[Cache]`, `[Nyaa]`, etc.) for better diagnostics

### Changed
- Removed `getOrFetchSearch()` wrapper function — simplified to direct rate limiter calls
- Merged all caching logic into `getOrFetchSearchGroup()` for single source of truth

---

## [1.0.0] - 2026-04-04

### Added
- **Core Torrent Streaming** — Nyaa.si and AnimeTosho torrent source integration with Stremio addon framework
- **Real-Debrid Integration** — Instant availability checking and torrent-to-direct-link resolution
- **Multi-Source Metadata** — Kitsu primary anime database with fallback to TVDB and TMDB
- **Episode Caching** — Server-side episode list caching with Real-Debrid availability indicators (⚡)
- **Configurable API Keys** — User-supplied TVDB and TMDB API keys via addon configuration page
- **Title Matching & Normalization**:
  - Roman numeral ↔ Arabic number conversions (e.g., "Season II" ↔ "Season 2")
  - Title part splitting on colon/dash separators
  - Bracket/parentheses tag stripping
  - Fuzzy title similarity matching for robust torrent filtering
- **Stremio Catalog** — Browse anime titles from Kitsu with poster artwork and metadata
- **Series Meta Object** — Full series information with genre, year, status, rating per Stremio spec

### Notes
- Initial stable release with foundational streaming pipeline
- Supports anime series and movies via kitsu, tvdb, tmdb and IMDB routing protocols

---

## Version Format

Versions follow semantic versioning: `MAJOR.MINOR.PATCH`
- **MAJOR**: Breaking changes or new content sources
- **MINOR**: New features or significant improvements
- **PATCH**: Bug fixes and minor enhancements
