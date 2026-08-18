# Softball Stats App — Project Guide

## Overview
Single-file HTML app at `/Users/mu_jing/repos/softball-stats/index.html` (~1.7MB).
Tracks a recreational softball league (Burlingame Dad's American League) across multiple seasons.
No build system, no dependencies — open in browser locally or serve from webserver.

**Production URL:** `https://pasghetticode.github.io/softball-stats/`  
GitHub Pages serves with gzip automatically (~500KB wire size).

## File Inventory
| File | Size | Purpose |
|------|------|---------|
| `index.html` | 3.8MB | Main app — all HTML/CSS/JS + fall-2026 data inline (+ inline seasons fallback tag) |
| `recaps.json` | 137KB | All game recaps (lazy-loaded on first game page visit) |
| `seasons.json` | 1.9MB | Historical seasons spring-2026 back through spring-2019 (lazy-loaded on season switch) |
| `avatars.json` | 1.0MB | Player SVG avatars (lazy-loaded in background on page load) |
| `player_photos.json` | <1KB | Real player photos keyed by player ID (e.g. `"seacrest-cayman": "https://..."`) — lazy-loaded; shown as thumbnails in scouting report |
| `scouting_overrides.json` | <1KB | AI-generated scouting reports keyed by teamId — structured JSON data rendered by `renderScoutingOverride()` in the app. Empty object `{}` = use rules-based fallback. Currently has: `hoover` (12 players), `franklin` (11 players). |
| `manifest.json` | <1KB | PWA manifest (name, theme color, install icons) |
| `sw.js` | 2KB | Service worker — pre-caches shell; network-first HTML, stale-while-revalidate JSON |
| `icons/icon-{180,192,512,maskable-512}.png` | ~10KB each | PWA install icons; regenerate via `python3 scripts/make_icons.py` |

**The core data files must be deployed together.** When updating weekly stats, you typically only need to update `index.html` and `recaps.json`. PWA files (`manifest.json`, `sw.js`, `icons/`) only change when modified explicitly.

## Architecture

### Data: `window.INLINE_SEASONS`
Only the **current season** (`fall-2026`) lives inline in the HTML (char ~64000):
```js
window.INLINE_SEASONS = { "fall-2026": {...} }
```
Historical seasons are in `seasons.json` and fetched lazily when the user switches seasons.

**Season object structure:**
```json
{
  "league": { "name": "...", "season": "...", "lastUpdated": "YYYY-MM-DD" },
  "teams": [ { "id": "franklin", "name": "Franklin", "wins": 1, "losses": 1, "ties": 0, "pct": 0.5 } ],
  "players": [ { "id": "lanam-todd", "name": "Lanam, Todd", "teamId": "franklin", "seasonStats": {...} } ],
  "games": [ { "id": "week1-game1", "week": 1, "date": "2026-03-11", "location": "...",
                "homeTeamId": "franklin", "awayTeamId": "carey",
                "homeScore": 20, "awayScore": 8,
                "homeStats": [...], "awayStats": [...] } ]
}
```

**Player stat fields** (per-game and season aggregate):
`PA`, `AB`, `H`, `R`, `E`, `D` (doubles), `T` (triples), `HR`, `BB`, `SF`, `BI` (RBI)

**Sparse per-game stats:** Zero-value fields are omitted from stored per-game stats. A normalization shim restores them to 0 at runtime. Do NOT store `0` values in `homeStats`/`awayStats` — only include non-zero fields.

**Season stats:** Do NOT store computed fields (`AVG`, `SLG`, `OPS`) — these are always recomputed from raw totals at render time. Only store: `G`, `PA`, `AB`, `H`, `R`, `E`, `D`, `T`, `HR`, `BB`, `SF`, `BI`.

**Player IDs:** `lastname-firstname` all lowercase, e.g. `lanam-todd` → "Todd Lanam"
- Exception: multi-part names like `jan-de-voogd-robert`
- Display name stored in `name` field as `"Last, First"` — always use this, not the ID
- Known case: `vanzandt-jonah` → name is `"VanZandt, Jonah"` (capital Z)

**Cross-league players:** A single player can play for both an AL and NL team in the same season (e.g. plays mckinley-2 on Tuesday nights and mckinley-1 on Thursday nights). This is intentional and supported — do NOT create separate player IDs for the same person. The player's `teamId` is set to whichever team they've played the most games for (primary team). Their `seasonStats` reflects primary-team games only. `getTeamPlayers()` finds all players who appeared in any game for a team, so cross-league players show up on both teams' rosters. `findPlayerAcrossSeasons()` computes per-team stats separately for the career stats table.

**Team IDs:** `franklin`, `st-catherines`, `lincoln-lions`, `hoover`, `washington-wildkittens`,
`burlingame-expansion`, `roosevelt-blue`, `carey`, `mckinley-2`, `bhs-panthers`, `bis`,
`mills-vikings` (new in Fall 2026), etc.

**A team's `league` can change between seasons** — ids are stable, `league` is per-season. Fall 2026
moved St. Catherines and Hoover to the NL and BIS and Roosevelt Gold to the AL. Never assume a
team's league from a prior season.

### Recaps: `recaps.json`
Flat JSON object keyed by game ID, fetched lazily on first game page visit:
```json
{ "week1-game1": "<p>...</p><p>...</p><p>...</p>", "fall25-game1": "..." }
```
- Fall 2026 keys: `fall26-game1` ... `fall26-gameN` (must NOT reuse `weekN-gameM`, which spring-2026 already owns)
- Spring 2026 keys: `week1-game1` ... `weekN-gameN`
- Fall 2025 keys: `fall25-game1` ... `fall25-game43`
- HTML with `&mdash;`, `&rsquo;`, `&ndash;` entities
- **3 paragraphs** per recap: intro/hero, winners' detail, losers' detail
- Games are **evening/night** games — never say "afternoon"
- Playoff games: label as `— Playoff First Round`, `— Semifinal`, or `— Championship` in the `<strong>` tag

### Lazy Loading
- `avatars.json` — fetched immediately in background on page load; falls back to DiceBear API if not yet loaded
- `seasons.json` — fetched when user switches to a historical season; also available via inline `<script type="text/plain" id="seasons-data">` tag for local `file://` testing
- `recaps.json` — fetched on first game page visit; also available via inline `<script type="text/plain" id="recaps-data">` tag for local `file://` testing

### Router
Hash-based SPA: `#/`, `#/schedule`, `#/players`, `#/team/:id`, `#/player/:id`, `#/game/:id`, `#/compare`

---

## League Structure

The app tracks **both leagues** (AL and NL) for all seasons. Each team has a `"league"` field (`"AL"` or `"NL"`). The AL/NL toggle in the header filters standings, schedule, and bracket by the selected league.

Franklin's league determines which league is labeled AL vs NL for a given season — but both leagues' full game and player data are always included in `seasons.json`.

### Week Numbering Rules

- **AL and NL week numbers are independent** — they do not need to match each other. Assign each game's `week` based solely on when that league's games were played.
- **Always assign week numbers by date**, not by schedule position. Look at the actual game date and compare it to the other games in that league to determine which week it belongs to.
- **Rivalry/interleague games** get their own week number — do NOT lump them into the preceding regular-season week just because they appear as the next batch. They're a distinct event on the schedule (usually a boxed section in the PDF).
- **Playoff games** (QF, SF, championship, consolation) always get `week` set AND the appropriate boolean flag (`quarterfinal`, `semifinal`, `championship`, `consolation`). Games missing `week` will not appear in the schedule view.

---

## How to Add New Week Results (Fall 2026)

### Source: bayareastats.com

League site: `https://www.bayareastats.com/burl/burl_dadsal_leg.htm`
- Check `lastUpdated` date on that page to confirm new stats are posted
- Get game **scores** and team **standings** from this page
- **Individual game pages (e.g. `burl_dadsal_t04_g03.htm`) are often 404** — don't rely on them
- Get **box scores** from each team's summary page instead (the "LAST WEEKS GAME" table)

**Team page URLs:** `https://www.bayareastats.com/burl/burl_dadsal_tNN.htm`

**The tNN → team mapping is NOT yet known for Fall 2026** — the roster of AL teams changed (see below), so the numbering will differ. Derive it by fetching the league page and reading the actual `<a href>` links next to each team name; do not reuse the Spring 2026 numbering.

Spring 2026 mapping, for reference only (stale as of Fall 2026):
`t01` Carey · `t02` McKinley 2 · `t03` Burlingame Expansion · `t04` St. Catherines ·
`t05` Franklin · `t06` Lincoln Lions · `t07` Hoover · `t08` Washington Wildkittens · `t09` Roosevelt Blue

**Box score columns from the site:** `PA, AB, H, R, O (outs—ignore), E, D, T, HR, BB, SF, BI`
Map directly to our stat fields; drop the O column.

**WebFetch prompt to use:** "Show the most recent game box score. List EVERY player row with their name and ALL stat columns exactly as shown in the table — include column headers."

**Finding game box scores — always traverse links, never guess URLs:**
Each team page lists all their games at the bottom as `GM-1`, `GM-2`, etc. with `<a href>` links. **Always read these links directly from the page** — do not construct or guess game page URLs (e.g. `_g03.htm`) based on naming patterns, because the site's file naming is inconsistent. Instead:
1. Fetch the team page
2. Extract the actual `<a href>` links for each `GM-N` entry
3. Follow those links to find the box score for the game you need

**If the final game has no separate link** (i.e. the last `GM-N` entry on the team page has no corresponding `<a href>`), the box score is embedded at the top of the team page itself under the "LAST WEEKS GAME" section — use the team page URL for that game.

**If a numbered game page returns 404**, it simply means that was the team's last game and the box score appears on the team page itself under "LAST WEEKS GAME". Never conclude the game doesn't exist — always fall back to the team page.

### Workflow

1. Fetch the league page to confirm the update date and read all scores/standings
2. Identify which team has the bye (compare standings teams to all 9 — the one missing a "Last Game" result)
3. Fetch all 8 playing teams' pages **in parallel** (use 8 simultaneous agent calls)
4. Run a Python script to update `index.html` (stats) and `recaps.json` (recaps)
5. Write recaps and insert them at the end of the fall-2026 entries in `recaps.json` (fall-2026 recaps go **first**, before the spring-2026 entries)

**Sanity checks before writing:**
- Each game's homeStats + awayStats hit totals should match homeScore/awayScore from the league page
- New players: anyone in a box score not in `player_map` needs to be added before accumulating stats
- Team records: wins/losses after the week should match the league page standings exactly

### Step 1: Update `window.INLINE_SEASONS['fall-2026']` in index.html

```python
import json

with open('index.html', 'r') as f:
    content = f.read()

idx = content.find('window.INLINE_SEASONS')
start = idx + content[idx:].find('{')
depth = 0
for i, ch in enumerate(content[start:]):
    if ch == '{': depth += 1
    elif ch == '}':
        depth -= 1
        if depth == 0:
            end = start + i + 1
            break

seasons = json.loads(content[start:end])
s = seasons['fall-2026']
player_map = {p['id']: p for p in s['players']}

# Per-game stat dicts — ONLY include non-zero fields:
# {"playerId": "...", "PA":N, "AB":N, "H":N, ...}  ← omit fields with value 0

# Update game stubs:
for game in s['games']:
    if game['id'] == 'fall26-gameN':
        game['homeScore'] = N
        game['awayScore'] = N
        game['homeStats'] = home_stats_list  # sparse — no zero fields
        game['awayStats'] = away_stats_list

# Add new players (G=0 seasonStats, will be incremented below):
s['players'].append({"id": "...", "name": "Last, First", "teamId": "...",
    "seasonStats": {"G":0,"PA":0,"AB":0,"H":0,"R":0,"E":0,"D":0,"T":0,"HR":0,"BB":0,"SF":0,"BI":0}})

# Accumulate stats (increment G by 1 per game played; sum all other fields):
for stat in all_week_stats:  # flat list of all homeStats + awayStats for this week
    p = player_map[stat['playerId']]
    ss = p['seasonStats']
    ss['G'] += 1
    for k in ['PA','AB','H','R','E','D','T','HR','BB','SF','BI']:
        ss[k] += stat.get(k, 0)

# Recompute AVG/SLG/OPS — do NOT store, just verify they'll compute correctly
# Update team records to match league page standings
# Set lastUpdated:
s['league']['lastUpdated'] = 'YYYY-MM-DD'

new_content = content[:start] + json.dumps(seasons, separators=(',', ':')) + content[end:]
with open('index.html', 'w') as f:
    f.write(new_content)
```

**For each completed game, update:**
1. `games[i].homeScore` and `games[i].awayScore`
2. `games[i].homeStats` and `games[i].awayStats` — sparse per-player stat objects (no zero fields)
3. `teams[i].wins` / `teams[i].losses` / `teams[i].ties` / `teams[i].pct`
4. `players[i].seasonStats` — aggregate stats across all games played (no AVG/SLG/OPS)
5. `league.lastUpdated` — set to today's date

**Win pct formula:** `(wins + 0.5 * ties) / (wins + losses + ties)`

**Team records include ALL games** — regular season, rivalry/interleague, playoff, AND consolation games. Recompute by iterating all games with scores and accumulating W/L/T for both home and away teams.

**Season stats fields to recompute:**
- `G`: games played
- `PA`, `AB`, `H`, `R`, `E`, `D`, `T`, `HR`, `BB`, `SF`, `BI`: running totals
- `AVG`: `H / AB` (3 decimal places, no leading zero: `.333`) — computed at render, not stored
- `SLG`: `(singles + 2D + 3T + 4HR) / AB` — computed at render, not stored
- `OPS`: `OBP + SLG` where `OBP = (H + BB) / PA` — computed at render, not stored

### Step 2: Add recaps to `recaps.json`

`recaps.json` is a flat JSON object. Fall 2026 is the newest season, so new `fall26-gameN` recaps go at the **top** of the file, before the first spring-2026 entry (`week1-game1`).

```python
import json

with open('recaps.json', 'r') as f:
    recaps = json.load(f)

new_recaps = {
    'fall26-game1': '<p><strong>...</strong> &mdash; ...</p>\n\n<p>...</p>\n\n<p>...</p>',
    # ...
}

# Newest season first; existing keys keep their relative order
recaps = {**new_recaps, **recaps}

with open('recaps.json', 'w') as f:
    json.dump(recaps, f, separators=(',', ':'))
```

**After adding recaps, also update the inline fallback tag in index.html:**
```python
import json

with open('recaps.json') as f:
    recaps_json = f.read()

with open('index.html', 'r') as f:
    content = f.read()

tag_start = content.find('<script type="text/plain" id="recaps-data">')
tag_end = content.find('</script>', tag_start) + 9
new_tag = f'<script type="text/plain" id="recaps-data">{recaps_json}</script>'
content = content[:tag_start] + new_tag + content[tag_end:]

with open('index.html', 'w') as f:
    f.write(content)
```

**Recap writing guidelines:**
- ESPN-style, 3 paragraphs, ~150-200 words total
- Open with the score and a compelling hook; name the standout performer
- Second paragraph: winning team's key contributors with stats
- Third paragraph: losing team's best efforts, closing line
- Evening/night games only — no "afternoon"
- Format: `"fall26-gameN": "<p><strong>Team A Score, Team B Score</strong> &mdash; ...</p>\n\n<p>...</p>\n\n<p>...</p>"`
- **Never invent team nicknames.** Only use nicknames that appear explicitly in the team data (the `name` field in the teams array). Do NOT guess or hallucinate nicknames (e.g. calling McKinley 2 "the Generals"). If you don't have the nickname, just use the team name.
- **Always verify recap stats against the actual game data** before writing. Cross-check hit totals, run totals, RBI counts, and individual player lines against `homeStats`/`awayStats` in the game object. Never write stats from memory.
- **RBI is plural when the count is 2 or more** — write "2 RBIs", "3 RBIs", etc. Only "1 RBI" is singular.
- **Known team nicknames:** Franklin = Falcons, Washington Wildkittens = Wildkittens, Lincoln Lions = Lions, St. Catherines = Tigers, Hoover = (no known nickname), Roosevelt Blue = (no known nickname), Burlingame Expansion = (no known nickname), Carey = (no known nickname), McKinley 2 = Bulldogs

**After each week:** Update the last recap key in this file (e.g. `fall26-game8`).

**Current last recap key:** none yet — Fall 2026 has no recaps. Last Spring 2026 key was `week11-game3`.

### Step 3: New players
If a player appears for the first time, add them to `seasons['fall-2026'].players`:
```json
{ "id": "lastname-firstname", "name": "Last, First", "teamId": "team-id",
  "seasonStats": { "G": 0, "PA": 0, "AB": 0, "H": 0, "R": 0, "E": 0, "D": 0, "T": 0, "HR": 0, "BB": 0, "SF": 0, "BI": 0 } }
```
G starts at 0 because the accumulation loop will increment it.

### Files to deploy after each update
- `index.html` — always (stats + inline fallback tags updated)
- `recaps.json` — always (new recaps added)
- `seasons.json` — only if historical data changed (rare)
- `avatars.json` — only if new player avatars were added (rare)
- `manifest.json`, `sw.js`, `icons/` — only if PWA files changed (rare). `sw.js` was bumped to `v2` for the Fall 2026 rollover, so **deploy it once** with that change.

**No service worker bump is needed for normal weekly updates** — the SW uses network-first for HTML and stale-while-revalidate for JSON, so users pick up new stats and recaps automatically. See PWA section below for when a `CACHE_VERSION` bump *is* required.

---

## Team Context (Fall 2026)

**Season set up 2026-08-18 from the two schedule PDFs:**
- AL: `https://www.bayareastats.com/burl/burl_dads_al_2026_fall.pdf`
- NL: `https://www.bayareastats.com/burl/burl_dads_nl_2026_fall.pdf`

No games played yet — all 18 teams 0-0-0, `players: []`, 72 regular-season game stubs.
Game IDs: **AL** `fall26-game1`..`fall26-game36`, **NL** `fall26-nl-game1`..`fall26-nl-game36`.

**League realignment from Spring 2026:**
- **AL → NL:** St. Catherines, Hoover
- **NL → AL:** BIS, Roosevelt Gold
- **Gone:** Taylor Bulldogs (no longer in either league)
- **New team:** **Mills Vikings** (NL) — brand-new `mills-vikings` id, no prior season history.
  It has **no `TEAM_LOGOS` entry and no `TEAM_COLORS` entry** in index.html, so it renders with a
  blank logo slot and default colors (same graceful state OLA is in for colors). Add both if the
  real logo/colors become available.

**AL teams (PDF listing order):** BIS · Roosevelt Gold · Lincoln Lions · Washington Wildkittens ·
Franklin · Roosevelt Blue · Carey · Burlingame Expansion · McKinley 2

**NL teams (PDF listing order):** McKinley 1 · Washington Wildcats · Mills Vikings ·
Millbrae Spring Valley · OLA · BHS · Lincoln Legends · St. Catherines · Hoover

Franklin is in the AL, so the AL/NL labeling resolves correctly and the header toggle shows both.

**Regular season — all games at Bayside Park Field #1 / #2, 7:00 and 8:05 PM.**
Each league is a single round-robin: 36 games, every team plays each opponent exactly once
(8 games each), one bye per week. **AL and NL week numbers are independent** and their dates differ.

| Week | AL date | AL bye | NL date | NL bye |
|---|---|---|---|---|
| 1 | Tue Aug 25 | McKinley 2 | Wed Aug 26 | Hoover |
| 2 | Wed Sep 2 | Washington Wildkittens | Tue Sep 1 | Millbrae Spring Valley |
| 3 | Tue Sep 8 | BIS | Wed Sep 9 | McKinley 1 |
| 4 | Wed Sep 16 | Franklin | Tue Sep 15 | OLA |
| 5 | Tue Sep 22 | Roosevelt Gold | Wed Sep 23 | Washington Wildcats |
| 6 | Wed Sep 30 | Roosevelt Blue | Tue Sep 29 | BHS |
| 7 | Tue Oct 6 | Lincoln Lions | Wed Oct 7 | Mills Vikings |
| 8 | Wed Oct 14 | Carey | Tue Oct 13 | Lincoln Legends |
| 9 | Tue Oct 20 | Burlingame Expansion | Wed Oct 21 | St. Catherines |

**Playoffs — NOT yet in the data** (matchups are seed-based, so stubs can't be created until the
regular season ends). Bracket is identical in both leagues: 3v6 / 4v5 / 1v8 / 2v7 in round one,
then Winner Gm4 v Gm1 and Winner Gm3 v Gm2, then Winner Gm6 v Gm5 for the title
(higher seed is home). Add as week 10/11 games with `quarterfinal` / `semifinal` / `championship`.

| Round | AL | NL |
|---|---|---|
| Quarterfinals (week 10) | Wed Oct 28 | Tue Oct 27 |
| Semifinals (week 11) | Tue Nov 3, 7:00 | Wed Nov 4, 7:00 |
| Championship (week 11) | Tue Nov 3, 8:05 F2 | Wed Nov 4, 8:05 F2 |

**9th place fun game:** Wed Nov 4, 8:05 PM, Field #1 — 9th place AL vs 9th place NL
(one game, listed on both PDFs; mark `consolation: true`).

**Scouting reports were reset** for the new season: `scouting_overrides.json` is now `{}` (and the
inline `scouting-overrides-data` tag matches). The Spring 2026 reports are archived at
`scouting_overrides_spring2026.json`. Note that `SCOUTING_OVERRIDES` is keyed by **teamId only, not
season** — a stale override would render on the new season's team page, which is why it was cleared.

### Season Rollover Checklist (what was done on 2026-08-18, repeat next season)
1. Move the outgoing season out of `window.INLINE_SEASONS` and into `seasons.json` as the **first** key.
2. Put the new season in `window.INLINE_SEASONS` as the only entry (both leagues' teams 0-0-0, `players: []`, game stubs with no scores). Check for teams new to the league — they need `TEAM_LOGOS` / `TEAM_COLORS` entries.
3. Pick a **non-colliding game-id prefix** (`fall26-gameN` / `fall26-nl-gameN`) — recaps and spray charts are keyed by game id across all seasons.
4. index.html edits: add `<option>` to `#season-picker`, add to `SEASON_LABELS`, append the new id to **every** `SEASON_ORDER*` array (there were 17 ascending ones plus 1 descending `SEASON_ORDER_VP`), set `let currentSeason`, the header `<h1>` `switchSeason(...)` onclick, and the `router()` URL-param default.
5. Re-sync the `seasons-data` inline tag (it must now include the newly archived season).
6. Clear `scouting_overrides.json` to `{}` (archive the old one) and sync its inline tag — overrides are keyed by teamId only, not by season.
7. Bump `CACHE_VERSION` in `sw.js` so clients don't serve a stale `scouting_overrides.json` / `seasons.json` from the SW cache.
8. Verify in a browser: standings, schedule, players, records, a team page, a game page, compare, plus switching to a historical season and back. Zero console errors expected.

### Spring 2026 Final (archived in `seasons.json`)
- **AL champion:** Washington Wildkittens (#6 seed) — beat St. Catherines 33-15 in the final
- **NL champion:** McKinley 1 — beat Washington Wildcats 11-10

---

## Fall 2025 Season Notes
- **Champion:** BHS Panthers (undefeated in regulation all season)
- **Runner-up:** Franklin Falcons (Cinderella run from last place in spring)
- **Semifinalists:** St. Catherines (lost to BHS 15-14), BIS (2nd place regular season, upset by Franklin 11-8)
- **Playoff format:** Week 10 = First Round, Week 11 games 41-42 = Semifinals, game 43 = Championship
- **game44** was a consolation game, removed from data
- Franklin's revenge game: beat Expansion 12-10 in playoffs after being drubbed 19-5 in Week 3

### The Jack Maguire Storyline (Fall 2025)
Jack Maguire played for Burlingame Expansion. He is **not a dad** — which raised eligibility questions throughout the season in a league called the "Dad's American League." He was by far the most dominant power hitter in the league.

**His home run arc:**
- Week 1 (game2): 2 HRs — first game, nobody else in the league had hit any
- Week 2 (game6): 2 more HRs (4 total) — eligibility murmurs growing
- Week 3 (game9): 3 HRs, 10 RBI — peak dominance; eligibility debate at full boil
- Week 4 (game14): 1 HR (8 total for season) — debate still unresolved
- Weeks 6–9: No more HRs — teams began pitching around him deliberately
- Playoffs (game38): Intentionally walked; Franklin refused to let him beat them (he'd gone 3 HRs/10 RBI vs. Franklin in Week 3)

**When writing recaps involving Maguire:** reference this arc where relevant — early games should note the eligibility question and his unprecedented power; later games should note teams pitching around him or walking him intentionally.

---

## Known Player Name Quirks
- `vanzandt-jonah` → display as "Jonah VanZandt" (capital Z in Van**Z**andt)
- `wong-uland` → "Uland Wong" (source pages have typo "Vland"; `uland` is correct)
- `oleary-chris-burl` → Burlingame Expansion's Chris O'Leary (suffixed to distinguish from Roosevelt's Joe O'Leary `oleary-joe`)
- `shapiro-benjamin` → Roosevelt Blue's Ben Shapiro (full name, not `shapiro-ben`)
- `singhal-rajeau` → source pages have typo "Rajecv"; `rajeau` is correct
- `rokovich-mike` → source pages sometimes show "Rakovich"; `rokovich` is correct
- `lagory-ed` → source pages sometimes show "LaGary"; `lagory` is correct
- `to-jackie` → source pages sometimes show "Tu"; `to` is correct
- `shah-pooja` → source pages sometimes show "Poaja"; `pooja` is correct
- `sosnick-jason` → source pages have typo "Sasnick"; `sosnick` is correct
- `sterrett-ryan` → source pages have typo "Stennett"; `sterrett` is correct
- `williams-benton` → source pages have typo "Barton"; `benton` is correct
- `moorthy-arjun` → source pages have typo "Argun"; `arjun` is correct
- `uharriet-philippe` → source pages have typo "Phillipe"; `philippe` is correct
- `delucchi-tony` → source pages use "Anthony"; `tony` is correct
- `burri-mark` → source pages have typo "Marv"; `mark` is correct
- `phillips-tamar` → source pages have typo "Tomar" or "Tamor"; `tamar` is correct
- Player names stored as `"Last, First"` in the `name` field; `displayName()` converts to "First Last"

---

## Scouting Reports

AI-generated scouting reports live in `scouting_overrides.json`, structured as data (not raw HTML) and rendered by `renderScoutingOverride()` in `index.html`. After any changes, sync the inline fallback tag with the Python snippet below.

**Lineup count rule:** Teams dress 10–13 players per game, not 9 like baseball. When building a scouting lineup, check the last 2–3 games and include every player who appeared in any of them. 10 is the bare minimum — most lineups will have 11–12. Cross-reference `homeStats`/`awayStats` arrays in the game data to find who played.

**After editing `scouting_overrides.json`, always sync the inline tag:**
```python
import json
overrides = json.dumps(json.load(open('scouting_overrides.json')), separators=(',',':'), ensure_ascii=True)
with open('index.html', 'r') as f: content = f.read()
tag_start = content.find('<script type="text/plain" id="scouting-overrides-data">')
tag_end = content.find('</script>', tag_start) + 9
content = content[:tag_start] + '<script type="text/plain" id="scouting-overrides-data">' + overrides + '</script>' + content[tag_end:]
with open('index.html', 'w') as f: f.write(content)
```

**Insight guidelines:**
- Base all insights strictly on available stats — don't assume batting tendencies (e.g. pull hitter) without data
- Extra-base hits (2B, 3B, HR) imply balls getting into gaps or over OF — use this to justify OF depth
- High ISO / HR = play OF deep; contact-only (no XBH) = OF can play shallower
- RBI/H ratio > 1.0 = dangerous with runners on; < 0.4 = table-setter role
- Triples suggest deep gap contact — OF must play very deep for those hitters
- Include both pitching advice AND OF positioning for each player

**Verbosity target:** Match Hoover's level of detail. Tips: **130–200 chars**. PitchTips: **70–110 chars**. Hoover is the gold standard — use it as a calibration reference when writing or reviewing any team's report.

**Lineup tip content rule:** The `tip` and `pitchTip` fields on each player card are for **tactical/qualitative insight only** — OF positioning, role in the lineup, what makes the player dangerous. Do NOT restate numbers that are already shown in the `chips` array (e.g. don't write ".833 average" or "14 RBI" if those are already chips). The reader sees the chips; the tip should add meaning, not repeat data.

**No handedness references:** Never use "pitch inside/outside/away", "jam him", "pull-side", "outer/inner half", "shift OF left/right", "up and in", "middle-in/out". We don't know which side batters hit from. Use vertical location cues instead ("keep it low", "don't let him extend") and neutral OF positioning ("play OF deep", "shade the alleys/gaps", "OF plays standard depth"). **Exception:** RF triples references are OK — right field is farther from home plate regardless of handedness, so it's field geometry not batter handedness.

**Batting order:** Use weighted relative position across the last 3 games (most recent = 3x weight, second = 2x, third = 1x). Normalize each player's position within their lineup (0.0–1.0 scale) before averaging — this preserves relative order regardless of lineup size (10–14 players). Players in the scouting report but absent from all 3 recent games go at the end.

**Lineup size:** Include every player who appeared in any of the last 2–3 games. Most lineups will have 11–12 players. 10 is the bare minimum. Use `homeStats`/`awayStats` arrays in game data to find who played.

**Use subagents in parallel** (one per team) when generating or updating scouting reports — queuing everything in memory causes resource issues. Write each team's output to a temp file (e.g. `/tmp/scout_{team}.json`), then merge at the end.

**Full scouting report schema — every team object has these top-level keys:**
```json
{
  "summaryStats": { "rpg": "...", "avg": "...", "slg": "...", "hr": N, "doubles": N, "record": "W-L" },
  "summaryNote": "...",
  "lineup": [ { "playerId": "...", "chips": [...], "tip": "...", "pitchTip": "..." } ],
  "sections": [
    { "title": "Power Threats",     "full": true,  "players": [...] },
    { "title": "Gap Hitters",       "full": false, "players": [...] },
    { "title": "Triples Threats",   "full": false, "players": [...] },
    { "title": "Hot Right Now",     "full": false, "players": [...] },
    { "title": "Watch List — Surprise Bats", "full": false, "players": [...] },
    { "title": "Run Producers (RBI/Hit > 0.7)", "full": false, "players": [...] },
    { "title": "Table Setters (score more than they drive in)", "full": false, "players": [...] },
    { "title": "Defensive Game Plan", "full": true, "gamePlan": [ { "heading": "...", "tip": "..." } ] }
  ]
}
```
Player entries in `sections[].players` use `{ "playerId": "...", "chips": [...], "tip": "..." }` (no `pitchTip`).
`gamePlan` entries use `{ "heading": "...", "tip": "..." }`.
**CRITICAL: When updating scouting reports, ALWAYS preserve the `sections` array.** If writing new reports from scratch, rebuild all 8 sections from stats. Never write a report with only `summaryStats`, `summaryNote`, and `lineup` — that silently drops the entire bottom half of the scouting card.

**Section size limit:** Each section's `players` (or `gamePlan`) array must have **at most 4 entries**. Trim to the top 4 by relevance before writing.

---

## Key Functions
- `switchSeason(seasonId)` — lazy-loads old seasons if needed, then switches; redirects to `#/schedule` if currently on a game page
- `_loadOldSeasons()` — fetches `seasons.json` (server) or reads `#seasons-data` inline tag (file://)
- `loadAvatars()` — background-fetches `avatars.json`; no-op on file://
- `normalizeSeason(s)` — fills missing zero-value per-game stat fields; called on INLINE_SEASONS at startup and on lazy-loaded seasons
- `getRecap(id, cb)` — fetches recap by game ID; uses `recaps.json` (server) or `#recaps-data` inline tag (file://)
- `displayName(name)` — converts "Last, First" → "First Last"
- `calcAVG/SLG/OBP/OPS` — stat calculators (always called at render time, never read from stored data)
- `viewBoxScore(gameId)` — renders game detail page
- `router()` — hash-based routing

---

## Important Implementation Notes

### Searching for function definitions
The file contains ~1MB of base64-encoded SVG avatar data. **Do NOT use `content.find('function viewBoxScore(')` or similar** — base64 strings can accidentally contain these character sequences. Always search for line-starting definitions using regex:
```python
import re
m = re.search(r'\nfunction viewBoxScore\(', content)
```

### JSON serialization
Always use `json.dumps(seasons, separators=(',', ':'))` (compact, no spaces) when writing back to index.html. Pretty-printing adds ~100KB.

### Inline fallback tags
Three inline tags exist in index.html for local `file://` testing — keep them in sync with their JSON files:
- `<script type="text/plain" id="seasons-data">` — all historical seasons JSON, i.e. everything in `seasons.json` (placed just before `</body>`). **This now includes spring-2026**, so it grew by ~1.6MB when fall-2026 became the current season.
- `<script type="text/plain" id="recaps-data">` — mirror of `recaps.json`
- `<script type="text/plain" id="scouting-overrides-data">` — mirror of `scouting_overrides.json`

### PWA / Service Worker
The app is installable as a PWA. Three pieces:
- `manifest.json` — referenced from `<head>` via `<link rel="manifest">`; defines name, icons, theme color
- `sw.js` — registered at end of script section (skipped on `file://`)
- `icons/icon-{180,192,512,maskable-512}.png` — generated by `python3 scripts/make_icons.py` (renders the ⚾ Apple Color Emoji)

**Caching strategy in `sw.js`:**
- Pre-caches shell on install: `./`, `./index.html`, `./manifest.json`, two icon PNGs
- HTML: **network-first** with cache fallback — weekly stat updates flow through automatically on next online visit
- JSON / icons / other assets: **stale-while-revalidate** — instant load from cache + background refresh

**When to bump `CACHE_VERSION` in `sw.js`** (top of the file, e.g. `'v1'` → `'v2'`):
- You change the caching logic or list of pre-cached files in `sw.js` itself
- You need to force-evict every client's cache (e.g., recovering from a bad deploy)

**When NOT to bump:**
- Weekly stat updates (`index.html`, `recaps.json`) — picked up automatically via the strategies above
- New seasons added to `seasons.json` — same
- Avatar changes — same

The version bump matters because the `activate` handler deletes any cache whose key isn't the current `CACHE_VERSION`, forcing a clean slate. Routine content updates don't need this since cached entries are keyed by URL and overwritten in place.
