# CLAUDE.md

Server config for the Livonia DayZ Clan Wars server. The repo root *is* the
server's mission/profile tree — the files here are uploaded verbatim to the game
server, so a merged change is not a shipped change until a release is published.

## Releases and deployments

### The one thing to remember

**Publishing a GitHub Release is what deploys.** Pushing to `main` deploys
nothing. Pushing a tag deploys nothing. `.github/workflows/deploy.yml` triggers
only on `release: [published]`, so a tag without a Release is a change that
never reached the server. (This happened to `v1.6.22`; its change did not go
live until it rode along in `v1.6.23`.)

### Versioning

`vMAJOR.MINOR.PATCH`, currently in the `v1.6.x` line. Every release is a patch
bump — `v1.6.23` → `v1.6.24`. Lightweight tags, always on `main`, one commit per
release in practice (batch only if the changes genuinely ship together).

### Procedure

1. **Commit the change** to `main`. Summary line is lowercase and describes the
   gameplay effect, not the edit ("disable NVGoggles and Plastic_Explosive",
   not "update types.xml"). Body is `- <file>: <before> -> <after>, and why`
   bullets — concrete values, since these are loot-economy numbers someone will
   want to diff later. End with the attribution lines
   (`Co-Authored-By:` / `Claude-Session:`) as in recent history.
2. **Tag it**: `git tag v1.6.N`
3. **Push both**: `git push origin main && git push origin v1.6.N`
4. **Publish the Release**: `gh release create v1.6.N --title v1.6.N --notes ...`
   This is the deploy trigger.
5. **Verify**: `gh run list -L 3` and confirm the `release` run for that tag
   shows `completed / success`. Do not report a release as shipped without it.

To audit for the failure mode above:

```sh
comm -23 <(git tag --sort=v:refname) \
         <(gh release list -L 100 --json tagName -q '.[].tagName' | sort -V)
```

Any tag it prints was never deployed.

### Release notes

Written for server admins and players, not for the diff. Markdown with a `##`
heading per themed change, the affected file in backticks, a sentence on the
in-game effect, then a before/after table when numbers change (nominal/min,
usages, values). Close with the Claude Code attribution block. See `v1.6.21` and
`v1.6.23` for the shape.

### ⚠️ `init.c` is not deployed from here any more

Since the clan-armbands change, **the Clan Wars bot owns `init.c`**. It renders the
whole file from the clan roster and uploads it to the mission root before every
scheduled restart, so the per-clan spawn armbands match who is actually in which clan.

`init.c` is therefore in `deploy.yml`'s exclude list, and a Release never ships it.
The copy in this repo is kept as a **reference for what the bot renders** — editing it
changes nothing on the server. The template the bot actually uses is
`apps/bot/src/init-c.ts` in the `clan-wars` repo; change the mission script there.

⚠️ The bot is the only writer. A hand-edit through Nitrado's file manager survives only
until the next restart slot (at most two hours), then is overwritten.

### How the deploy works

`SamKirkland/FTP-Deploy-Action` checks out the released tag and FTPs the repo
root to `${{ secrets.FTP_DIRECTORY }}`. Credentials are the `FTP_SERVER`,
`FTP_USERNAME`, `FTP_PASSWORD` and `FTP_DIRECTORY` repo secrets.

It is **incremental**: the action keeps `.ftp-deploy-sync-state.json` on the
server recording the last deployed state, and uploads only what changed against
it. Consequences worth knowing:

- **Never delete `.ftp-deploy-sync-state.json` from the server.** Losing it
  forces a full re-upload and loses the diffing that keeps deploys cheap.
- **`dangerous-clean-slate: false` must stay false.** True wipes the server
  directory, including files the game writes that are not in this repo.
- Because it diffs against server state rather than against the previous tag, a
  release also carries any earlier commit that was never deployed. That is why
  the `v1.6.22` recovery worked — but rely on the audit above, not on this.
- Excluded from upload: `.git/`, `.github/`, `.claude/`, `.superpowers/`,
  `docs/`, `node_modules/`, `vendor/`, all `*.md` (so this file never ships),
  `.gitignore`, `.gitattributes`. Add new tooling directories to the exclude
  list in `deploy.yml` — anything not excluded lands on the game server.

### After deploying

The server reads most of this tree on restart. A successful deploy means the
files are in place, not that the change is live in the current session — loot
economy edits (`db/types.xml`, `db/economy.xml`, `cfgspawnabletypes.xml`) take
effect on the next server restart and, for nominal/min changes, as the economy
re-settles after that.
