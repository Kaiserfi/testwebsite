# testwebsite (laidbackcrew.org)

GitHub Pages site for the LaidBackCrew / RedRune friends. Branch `main`, custom domain in `CNAME`. Pushing to `main` publishes within a few minutes. Three repos work together:

| Repo | Branch | What |
|---|---|---|
| Kaiserfi/testwebsite (this one) | `main` | the website, the downloads, the flipping Worker's source (`worker/`) |
| Kaiserfi/redrune-server | `main` | the RedRune 727 server (has its own CLAUDE.md) |
| Kaiserfi/redrune-client | `master` | game client, PC launcher, Android app (its release scripts write into `downloads/` here) |

The owner is not a programmer, works a lot from a phone, and uses Windows/PowerShell on PC. Give simple steps, one command per code block. Commit and push straight to `main` (no pull requests), and say what the owner has to do (usually nothing: Pages updates itself).

`_config.yml` excludes `CLAUDE.md` and `worker/` so they are not served on the public site. Nothing secret may be committed here at all: the repo is public.

## Pages

- `index.html`: OSRS skills leaderboard (calls the OSRS highscores API from the browser).
- `redrune.html`: RedRune highscores, reads `redrune-highscores.json` (written by the game server; list of players with total level/XP and per-skill data).
- `flipping.html`: Flipping leaderboard (Grand Exchange profit from the Flip Ledger RuneLite plugin), reads the Worker below.
- `icons/`: images.
- `downloads/`: files published by the redrune-client release scripts, never edited by hand:
  `RedRune.exe`, `RedRune.jar` (PC client), `RedRune.apk` (Android), `redrune-desktop.json` (`{"revision": N, "notes": "..."}`, read by the PC client's update check), `redrune-android.json` (`versionCode`, `versionName`, `notes`, `apk` URL, read by the app's update check) and `downloads/cache/` (the game cache as zip parts + `cache.json`, which the Android app downloads; regenerate from the server's `data/cache` after cache changes).
  The standing rule (see redrune-client CLAUDE.md) is that the Android app is published right after every PC release.

## Flipping leaderboard (`flipping.html` + Cloudflare Worker `flip-leaderboard`)

How it works:
- The Flip Ledger plugin (each friend's RuneLite) POSTs its player's current 1 hour / 12 hour / 24 hour results to the Worker (`POST /submit`, needs the crew key in `X-Group-Key` and the player's own token in `X-Player-Token`; a name is claimed by whoever submits it first).
- The Worker `https://flip-leaderboard.kaiserfi918.workers.dev` keeps them in a D1 database (binding `DB`, secret `GROUP_KEY`). Windows reset on Eastern time (the `h24` window is one Eastern day).
- `GET /leaderboard` (public, `Cache-Control: max-age=30`) returns `{now, windows: {h1, h12, h24, d3, d7, all}}`. Each window: `start`, `end` (null for d3/d7/all: they never reset), `current` (up to 50 players: `name, profit, flips, updated, best, bestItem, worst, worstItem`), `previousStart`, `previous`.
- **3 days, 7 days and all time (added 1 Nov 2026)** are not stored: the Worker adds up the `h24` rows (one per player per Eastern day). 3 days = today and the 2 days before, 7 days = today and the 6 before, all time = every day kept. Profit and flips are summed, best flip / worst loss are the biggest of any day, names that differ only by case are merged. The 1h/12h rows are deleted after 3 days; **`h24` rows are kept forever** (before 1 Nov 2026 they were deleted after 3 days, so the 7 day and all time boards fill in from that date).
- The page shows the extra tabs only when the Worker returns those windows (so page and Worker can be deployed in either order), shows no reset countdown for them, and the tier badges (Bronze 1+, Iron 5M, Steel 7.5M, Black 10M, Mithril 15M, Adamant 20M, Rune 30M, Dragon 50M) are based on **all-time profit** (owner's decision, 1 Nov 2026; they used to use the 24h board), falling back to the 24h board if the Worker has no `all` window. The all-time total only counts days the Worker has kept (see above), and the thresholds were first chosen for a single day, so they may need raising later.

The Worker's source is `worker/worker.js` (a copy of what is deployed; keep it in step). It was originally edited in the Cloudflare dashboard, which is impractical on a phone, so changes are made here and deployed through the API.

### Deploying the Worker from a cloud session

1. The owner creates an API token (Cloudflare dashboard, profile, API Tokens, template "Edit Cloudflare Workers", limited to their account) and adds it to the Claude environment as the secret/variable `CLOUDFLARE_API_TOKEN` (**never ask them to paste it in chat**; on 1 Nov 2026 they pasted one in chat anyway, I used it once and told them to delete it). Ask them to delete or roll it when done.
2. `api.cloudflare.com` is reachable from a cloud session. The account id comes from `GET /accounts` with the token; the Worker is `flip-leaderboard`.
3. Upload (keeps the D1 binding and the secret; `compatibility_date` was `2026-10-27`):
   ```
   echo '{"main_module":"worker.js","compatibility_date":"2026-10-27","keep_bindings":["d1","secret_text"],"observability":{"enabled":true,"head_sampling_rate":1}}' > meta.json
   curl -X PUT -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" https://api.cloudflare.com/client/v4/accounts/<account id>/workers/scripts/flip-leaderboard \
     -F "metadata=@meta.json;type=application/json" -F "worker.js=@worker/worker.js;filename=worker.js;type=application/javascript+module"
   ```
4. Check `GET /leaderboard` afterwards, and `GET /accounts/<id>/workers/scripts/flip-leaderboard/settings` to confirm the bindings `DB` (d1) and `GROUP_KEY` (secret_text) are still there. The previous code can be read back from `/workers/scripts/flip-leaderboard/content/v2` before overwriting (save it first).
5. Test changes to the Worker with a fake `env.DB` in Node before deploying (the aggregation is a pure function, `periodWindow`), and the page with Playwright and a mocked `/leaderboard` response.

## RedRune highscores (`redrune.html`, `redrune-highscores.json`)

Written by the game server on the VPS (`HighscoresPublisher` in redrune-server), which uploads the file here through GitHub's Contents API every 10 minutes when the rankings changed. That needs `data/highscores-github-token.txt` on the VPS (a fine-grained token with Contents read/write on this repo; not in git; the owner created it on 1 Nov 2026 and the page updated; it was exposed in chat once, so the owner was told to regenerate it). Each "Update RedRune highscores" commit in this repo's history is that upload. Administrator accounts are left off and names show as in game (both in the server code, deployed by the owner); individual accounts can be hidden with `data/highscores-hidden.txt` on the VPS. Never edit the JSON by hand.

## State at the end of the 1 Nov 2026 session

- Flipping: 3 day / 7 day / all time tabs are live (Worker deployed through the Cloudflare API; the Worker source copy is `worker/worker.js`). Ranks are all-time based (0srsn0ob showed Iron at 7.35M). The thresholds were chosen for one day and may need raising once more history exists; the owner has not chosen new numbers.
- The real date was 1 Nov 2026 (earlier notes elsewhere said 2-3 Nov by mistake).
- The Cloudflare token the owner pasted in chat should be deleted by them; for future Worker deploys they add a fresh one as `CLOUDFLARE_API_TOKEN` in the environment settings.

## Downloads notes (4 Nov 2026)

- `downloads/RedRune-debug.jar` is a **test build, not a release** (no revision bump, no update prompt): the owner downloads it by hand to try client changes before a release (see the client CLAUDE.md, "Release policy"). It is overwritten by whatever was last tested, so do not rely on it. `RedRune.exe` is unsigned and Windows SmartScreen / Defender flags it (a new file every revision); `RedRune.jar` (Java 17+) is the alternative. A "Having trouble?" help section on the download page was offered, not written.
- The game server (World 1) pushes `redrune-highscores.json` here every 10 minutes, so run `git pull --rebase origin main` before every push from a cloud session or the push is rejected.

## Gotchas

- The page `flipping.html` must keep working with an older Worker response (missing `d3/d7/all`).
- `redrune-highscores.json` and everything in `downloads/` is written by other repos' tooling; do not reformat or hand-edit.
- GitHub Pages caches for a few minutes; hard refresh when checking.
