# Moving to a GitHub organization: migration checklist

Status: **draft, nothing done** (2026-10-14). Proposal from the community: create an org (working name "OpenMPC") for the
structural repos, and optionally take in other developers' repos with their consent. This file is the plan and the
list of what would break. Mark items as they are done, with the date and whether it was checked offline or on a device.

## Decisions to make first
- [ ] **Org name.** Check availability on GitHub, and check the name against Akai's trademarks before it looks "official".
      (The site is already titled "Open MPC - Plugin Catalog".)
- [ ] **Owners.** At least two, so no single account is a point of failure. Decide who gets merge rights on the catalog.
- [ ] **What moves.** Proposed: `mpc-vst-plugins` (wrapper, tools, catalog, site, installer) and the ports `sd88me` owns.
      Other developers' repos stay where they are and stay listed; an invitation to join is optional and needs their consent.
- [ ] **Policy page** (review, yank, security contact, branding rule, who may merge). An official-looking org implies these.

## What is hard-coded to `sd88me` (found offline, 2026-10-14)
74 files mention it; 35 are `sd88me/mpc-vst-plugins`. The ones that matter at run time:

| Where | What | Breaks if not handled |
|---|---|---|
| `tools/desktop/catalog.go` `defaultCatalogURL` | `https://sd88me.github.io/mpc-vst-plugins/catalog.json` | **Every installed copy of the desktop app.** Pages URLs are not redirected on transfer. |
| `tools/mpc-store.sh` `URL=` | `.../catalog.tsv` (overridable with `MPC_STORE_URL`) | Copies already on devices. |
| `tools/catalog_site/index.template.html` | `SITE` fallback | Minor (only when not served over http). |
| `catalog/patches.json`, `docs/PATCHES.md` | `raw.githubusercontent.com/sd88me/mpc-vst-plugins/<commit>/...` | Raw URLs after a transfer: verify, do not assume. |
| `.github/workflows/{vst,addin}-release.yml` and the docs that show them | `uses: sd88me/mpc-vst-plugins/.github/workflows/...@<sha>` | **Other developers' release workflows** pin this path. |
| `.github/workflows/vst-release.yml:106`, `addin-release.yml:88` | `repository: sd88me/mpc-vst-plugins` in the "Check out mpc-vst-plugins (tools)" step | Every release run that uses the reusable workflow, until the owner is changed here. Whether the checkout follows a transfer redirect is unverified. |
| `tools/catalog_build.py` | `api.github.com/repos/<owner>/<name>/...` through `urllib` (follows redirects on GET) | Probably nothing, but untested. Update the registry entry anyway. |
| `catalog/plugins/*.json` | `repo` for ~23 `sd88me` entries | Catalog discovery if a repo moves and the entry is not updated. |
| `README.md`, `catalog/pages/*.md`, `docs/*` | links | Cosmetic once redirects work. |
| `docs/NOTES.md` | historical mentions | Leave alone; they are history. |

## What will and will not be resolved (2026-10-14, offline analysis; redirect behaviour unverified)

### Cannot be fully resolved
| Problem | Why |
|---|---|
| Old installs of the desktop app and `mpc-store.sh` | They default to `sd88me.github.io/...`. Pages URLs reportedly do not redirect on transfer. A copy can be kept at the old URL, but only by keeping something under `sd88me` for as long as old installs exist. |
| Old external links (forum, Discord, videos) to the Pages site | Repo links redirect; Pages links reportedly do not. |
| Stars, download counts shown on the catalog | Whether they carry over is unverified. |
| Reusing the old name | Creating anything at `sd88me/mpc-vst-plugins` later can break the redirect. A permanent constraint on the old account. |
| Third parties who never update their workflow | They stay on whatever the redirect (if any) gives them. |

### Can be fixed
| Problem | Fix |
|---|---|
| Hard-coded owner in 74 files, including the two `repository:` checkouts above | One reviewed PR with tests (`test_patches.py`, desktop tests, `catalog_check.py --catalog`) and docs updated in the same PR. |
| Clients with one catalog URL | Release the desktop app and `mpc-store.sh` with both URLs (new first, old as fallback) before the transfer. |
| `repo` in the ~23 `sd88me` registry entries | Update each in the same step as that repo's transfer. |
| Pinned raw URLs in `catalog/patches.json` | Rewrite once the redirect behaviour is known. |
| Secrets, webhooks, Pages source, branch protection | Redo on the org (checklist item). |
| Trust and governance | Policy page, 2+ owners. |
| Name and trademark | Check before committing to the name. |

### What contributors will need to do
- **Authors who stay in their own repos:** possibly nothing, if a `uses:` reference follows the redirect. If not, change one
  line in their release workflow (`sd88me/mpc-vst-plugins/...@<sha>` to the org path). Their registry entry is unaffected.
- **Authors whose repo moves into the org:** consent to and accept the transfer (ownership and admin change), and update
  their `catalog/plugins/<id>.json` `repo` field in a PR.
- **Anyone with a fork or clone:** update the remote URL (the old one keeps working while the redirect stands).
- **New contributors:** catalog PRs go to the org's repo; the docs and PR template say so.

### Effect on the catalog
- **Discovery:** plugins that do not move are unaffected. For moved repos the API redirect should carry the call, but
  update the registry entry rather than rely on it.
- **Site URL:** becomes `<org>.github.io/mpc-vst-plugins/` (or a custom domain). The old one needs a redirect page or a copy.
- **`catalog.json`, `catalog.tsv`, `patches.json`:** must be live at the new URL before clients switch.
- **Version history and `tested.json`:** live in the plugin repos and the catalog repo, so they move with them. Check the
  nightly build after every transfer.
- **Risk window:** between the transfer and clients learning the new URL. This is why the dual-URL client release goes first.

## Facts to verify before relying on them
GitHub's docs were not reachable from the agent session (network policy), so these come from search snippets or memory.
Check each by hand, ideally on a throwaway repo and org first.
- [~] **Git and web links redirect after a transfer; GitHub Pages does not.** Git clone and web links: confirmed live
      (2026-10-19, see below). Pages: **not yet verified**, needs a browser (github.io is blocked in agent sessions); user to
      report what `<old-org>.github.io/scratch-tools/` and the new URL do.
- [ ] **Do creating a new repo at the old name, or recreating the old repo to host a Pages redirect, break the redirect?**
      The Boost thread says it can. Test before using that workaround.
- [x] **Does `uses: owner/repo/.github/workflows/x.yml@<sha>` follow a transfer redirect?** **No** (live, 2026-10-19).
      Scratch repos `mpc-vst-test/scratch-tools` (transferred to `sd88me`) and `mpc-vst-test/scratch-caller`. After the transfer,
      dispatching the caller with the old path was refused: `error parsing called workflow ... "mpc-vst-test/scratch-tools/.github/workflows/x.yml@<sha>": workflow was not found`.
      Control: the same caller with `sd88me/scratch-tools/...@<sha>` ran green, and the same old-path caller ran green before the
      transfer. **Every external release workflow pinning `sd88me/mpc-vst-plugins/.github/workflows/...@<sha>` needs a one-line
      edit after the transfer**, so this is the main thing to announce to plugin authors.
- [x] **Does `actions/checkout` with `repository: <old owner>/<name>` follow it?** **Yes** (live, 2026-10-19). A workflow in
      `scratch-caller` doing `actions/checkout@v4` with `repository: mpc-vst-test/scratch-tools` at the pinned sha, after the
      transfer, succeeded and read the file. The hard-coded `repository:` lines in `vst-release.yml:106` / `addin-release.yml:88`
      keep working through the redirect, but change them anyway.
- [~] **Does `urllib` in `catalog_build.py` follow the API's redirect for a transferred repo** (and keep its auth header)?
      Partly verified (live, 2026-10-19). `GET api.github.com/repos/<old>/releases` answers `301 Moved Permanently` with
      `Location: https://api.github.com/repositories/<id>/releases`, so the redirect exists. **Not verified end to end**: the agent
      proxy rejects `/repositories/{id}/...` paths (and `sd88me/scratch-tools` was not in the session scope), so `urllib` got 403 from
      the proxy, not from GitHub. `urllib` follows a 301 on GET to the same host and resends its headers, which is expected to keep
      the auth header, but this was not observed. Re-run on a machine without that proxy, or just update the registry entry first.
- [x] **`releases/download/...` and `raw.githubusercontent.com` URLs after a transfer.** **Both still work under the old name**
      (live, 2026-10-19). `github.com/<old>/releases/download/v1/asset.txt` (curl -L) ends 200 with the asset;
      `raw.githubusercontent.com/<old>/<sha>/pinned.txt` returns 200 with the pinned content; the new name serves the same. The pinned
      raw URLs in `catalog/patches.json` therefore do not break on transfer, but rewrite them in the owner-change PR anyway.
      Also seen: `github.com/<old>` answers `301` to `github.com/<new>`, and the `git clone` of the old URL worked (remote stays the old URL).
- [ ] Which settings travel with a transfer: Pages config, Actions secrets and variables, webhooks (the Discord release
      workflow), branch protection, deploy keys, and the release-asset download counts the catalog shows.
- [ ] Org Pages URL form (`<org>.github.io/mpc-vst-plugins/`) and whether a custom domain is wanted instead.

## Sequence
1. **Rehearse** on a scratch org and a scratch repo that mimics `mpc-vst-plugins` (Pages site, a release with an asset,
   a pinned reusable workflow, a raw URL). Record what redirects.
2. **Before the transfer**, ship a release of the desktop app and `mpc-store.sh` that already knows both catalog URLs
   (try the new one, fall back to the old). Do this first because old installs cannot be updated after the old URL dies.
3. **Create the org**, add owners, set base permissions, enable 2FA requirement, set up secrets again where needed.
4. **Transfer `mpc-vst-plugins`.** Re-enable Pages from the Actions workflow, check the nightly `catalog.yml` run, the
   site, `catalog.json`, `catalog.tsv` and `patches.json`.
5. **Keep the old Pages URL alive** for as long as old clients exist: a redirect page published from the old owner
   (a `sd88me.github.io` repo) or a copy of `catalog.json` that points at new download URLs. Test before relying on it.
6. **One PR** replacing `sd88me/mpc-vst-plugins` in the table above, with a test that `tools/test_patches.py`,
   the desktop tests and `catalog_check.py --catalog` still pass. Update docs in the same PR (CLAUDE.md "Docs sync").
7. **Transfer the other `sd88me` ports** one at a time, each with its `catalog/plugins/<id>.json` `repo` update in the
   same step. Run the catalog build after each.
8. **Tell external plugin authors** (their entries and workflows keep working if redirects hold; give them the new `uses:` path).
9. **Announce** with the policy page. Mention that listing does not require moving.
10. **Archive the checklist** into `NOTES.md` (dated, with what was verified and where) and delete or trim this file.

## Rollback
Transfers can be reversed while the old name is free, so do not create anything at the old names until the new setup has run
for a few catalog cycles. Keep the old Pages redirect and the dual-URL clients for at least one release cycle.

## Prior art (from memory, not re-checked)
- VCV Rack's `library` repo: one thread or manifest per plugin, authors keep their repos, maintainers build and update.
  (README fetched 2026-10-14: updates are a version bump plus a commit hash posted to the plugin's thread; it does not
  describe hosting.)
- conda-forge: staged-recipes PR for first entry, then per-package repos kept up by bots.
- Homebrew: curated core plus third-party taps.
