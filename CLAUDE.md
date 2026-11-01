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
- **3 days, 7 days and all time (added 3 Nov 2026)** are not stored: the Worker adds up the `h24` rows (one per player per Eastern day). 3 days = today and the 2 days before, 7 days = today and the 6 before, all time = every day kept. Profit and flips are summed, best flip / worst loss are the biggest of any day, names that differ only by case are merged. The 1h/12h rows are deleted after 3 days; **`h24` rows are kept forever** (before 3 Nov 2026 they were deleted after 3 days, so the 7 day and all time boards fill in from that date).
- The page shows the extra tabs only when the Worker returns those windows (so page and Worker can be deployed in either order), shows no reset countdown for them, and keeps the tier badges (Bronze to Dragon) based on the 24h board.

The Worker's source is `worker/worker.js` (a copy of what is deployed; keep it in step). It was originally edited in the Cloudflare dashboard, which is impractical on a phone, so changes are made here and deployed through the API.

### Deploying the Worker from a cloud session

1. The owner creates an API token (Cloudflare dashboard, profile, API Tokens, template "Edit Cloudflare Workers", limited to their account) and adds it to the Claude environment as the secret/variable `CLOUDFLARE_API_TOKEN` (**never ask them to paste it in chat**; on 3 Nov 2026 they pasted one in chat anyway, I used it once and told them to delete it). Ask them to delete or roll it when done.
2. `api.cloudflare.com` is reachable from a cloud session. The account id comes from `GET /accounts` with the token; the Worker is `flip-leaderboard`.
3. Upload (keeps the D1 binding and the secret; `compatibility_date` was `2026-10-27`):
   ```
   echo '{"main_module":"worker.js","compatibility_date":"2026-10-27","keep_bindings":["d1","secret_text"],"observability":{"enabled":true,"head_sampling_rate":1}}' > meta.json
   curl -X PUT -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" https://api.cloudflare.com/client/v4/accounts/<account id>/workers/scripts/flip-leaderboard \
     -F "metadata=@meta.json;type=application/json" -F "worker.js=@worker/worker.js;filename=worker.js;type=application/javascript+module"
   ```
4. Check `GET /leaderboard` afterwards, and `GET /accounts/<id>/workers/scripts/flip-leaderboard/settings` to confirm the bindings `DB` (d1) and `GROUP_KEY` (secret_text) are still there. The previous code can be read back from `/workers/scripts/flip-leaderboard/content/v2` before overwriting (save it first).
5. Test changes to the Worker with a fake `env.DB` in Node before deploying (the aggregation is a pure function, `periodWindow`), and the page with Playwright and a mocked `/leaderboard` response.

## Gotchas

- The page `flipping.html` must keep working with an older Worker response (missing `d3/d7/all`).
- `redrune-highscores.json` and everything in `downloads/` is written by other repos' tooling; do not reformat or hand-edit.
- GitHub Pages caches for a few minutes; hard refresh when checking.
