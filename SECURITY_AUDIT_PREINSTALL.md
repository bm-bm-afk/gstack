# Security Audit — gstack (Pre-Installation)

Audit date: 2026-04-19 (UTC)
Scope audited: `setup`, `bin/`, `SKILL.md` + `SKILL.md.tmpl`, preamble generators under `scripts/resolvers/preamble/`, browse implementation under `browse/src/`, Supabase functions under `supabase/functions/`, and repository-level docs/config files for installation/network references.
Method: static source review with targeted grep for install/network/file-write/prompt-injection patterns.

---

## [SEVERITY: CRITICAL]

**Category:** Installation  
**File:** `setup`  
**Line(s):** 239-240, 299-300, 309-310, 319-320  
**Finding:** `./setup` runs dependency installation/build commands (`bun install`, `bun run build`, skill doc generation) without interactive confirmation.

**Code:**
```bash
bun install --frozen-lockfile 2>/dev/null || bun install
bun run build
...
bun run gen:skill-docs --host codex
...
bun run gen:skill-docs --host factory
...
bun run gen:skill-docs --host opencode
```

**Risk:** Executing unpinned/unreviewed postinstall hooks or scripts on install host; network dependency fetches occur automatically.

**Safe to remove?** No (for normal setup). Core setup fails or skips build/doc generation.

**Recommendation:** Replace with gated mode (`--dry-run` + `--approve-installs`), or run only after explicit user prompt.

---

## [SEVERITY: HIGH]

**Category:** Installation  
**File:** `setup`  
**Line(s):** 324-330, 344-349, 270-275  
**Finding:** `./setup` auto-installs Playwright Chromium and may run `npm install --no-save` and `brew install coreutils` in some environments.

**Code:**
```bash
bunx playwright install chromium
...
node -e "require('playwright')" 2>/dev/null || npm install --no-save playwright
node -e "require('@ngrok/ngrok')" 2>/dev/null || npm install --no-save @ngrok/ngrok
...
brew install coreutils >/dev/null 2>&1
```

**Risk:** Silent package/binary installs from external registries/system package manager.

**Safe to remove?** Partial. Removing `brew install coreutils` is usually safe (feature degradation only). Removing Playwright install breaks browse.

**Recommendation:** Gate each install with explicit AskUser approval; add env flags to disable each installer independently (`GSTACK_SKIP_PLAYWRIGHT`, `GSTACK_SKIP_NPM_FALLBACK`, `GSTACK_SKIP_COREUTILS`).

---

## [SEVERITY: HIGH]

**Category:** Telemetry / Network  
**File:** `bin/gstack-telemetry-sync` + `supabase/config.sh`  
**Line(s):** `bin/gstack-telemetry-sync` 23-27, 111-116; `supabase/config.sh` 7-8  
**Finding:** Telemetry sync POSTs batched analytics to Supabase endpoint using committed project URL + anon key.

**Code:**
```bash
SUPABASE_URL="${GSTACK_SUPABASE_URL:-}"
ANON_KEY="${GSTACK_SUPABASE_ANON_KEY:-}"
...
curl -s -w '%{http_code}' --max-time 10 \
  -X POST "${SUPABASE_URL}/functions/v1/telemetry-ingest" \
  -H "apikey: ${ANON_KEY}" \
  -d "$BATCH"
```

**Risk:** Outbound telemetry by default when telemetry tier is not off; sends usage metadata to remote service.

**Safe to remove?** Yes for privacy; telemetry upload breaks, local analytics still works.

**Recommendation:** Force `telemetry: off` in `~/.gstack/config.yaml` and/or remove execute bit from `bin/gstack-telemetry-log` and `bin/gstack-telemetry-sync`.

---

## [SEVERITY: HIGH]

**Category:** Telemetry / Data Collection  
**File:** `bin/gstack-telemetry-log`  
**Line(s):** 191-195, 117-139, 147-149  
**Finding:** Local telemetry JSON includes skill/runtime metadata and local-only repo/branch fields before sync.

**Code:**
```bash
printf '{"v":1,...,"skill":"%s",...,"sessions":%s,"installation_id":%s,"source":"%s","_repo_slug":"%s","_branch":"%s"}\n' ...
```

**Risk:** Sensitive project context (`_repo_slug`, `_branch`) is written locally; if file is exfiltrated, behavioral data is exposed.

**Safe to remove?** Yes with caveat: preamble/epilogue telemetry pipeline loses observability features.

**Recommendation:** Disable telemetry and strip local-only fields at write-time as well (not only at sync-time).

---

## [SEVERITY: HIGH]

**Category:** Prompt Injection / File Access  
**File:** `scripts/resolvers/preamble/generate-routing-injection.ts`  
**Line(s):** 4-5, 17-43  
**Finding:** Skill preambles instruct agent to create/append `CLAUDE.md` and commit changes.

**Code:**
```text
Check if a CLAUDE.md file exists in the project root. If it does not exist, create it.
...
Then commit the change: `git add CLAUDE.md && git commit -m "chore: add gstack skill routing rules to CLAUDE.md"`
```

**Risk:** Agent modifies project policy files automatically during normal skill use.

**Safe to remove?** Yes. Removes routing auto-injection only; core skills still usable.

**Recommendation:** Remove this generator section for hardened installs.

---

## [SEVERITY: MEDIUM]

**Category:** Network / Update Check  
**File:** `bin/gstack-update-check`  
**Line(s):** 21, 173-178, 183  
**Finding:** Update checker contacts GitHub raw VERSION and optionally Supabase `update-check` function.

**Code:**
```bash
REMOTE_URL="${GSTACK_REMOTE_URL:-https://raw.githubusercontent.com/garrytan/gstack/main/VERSION}"
curl -sf ... -X POST "${_SUPA_URL}/functions/v1/update-check" ...
REMOTE="$(curl -sf --max-time 5 "$REMOTE_URL" ... )"
```

**Risk:** Background network calls during skill preambles/upgrade checks.

**Safe to remove?** Yes. You lose upgrade notifications/community metrics.

**Recommendation:** `gstack-config set update_check false` and `gstack-config set telemetry off`.

---

## [SEVERITY: MEDIUM]

**Category:** File Access  
**File:** `setup` + `bin/gstack-settings-hook` + `bin/gstack-team-init`  
**Line(s):** `setup` 989, 1007; `bin/gstack-settings-hook` 15, 29-53; `bin/gstack-team-init` 101-103, 114-137, 140-173  
**Finding:** Setup/team commands modify files outside repo: `~/.claude/settings.json`, project `CLAUDE.md`, `.claude/hooks`, `.claude/settings.json`.

**Code:**
```bash
"$SETTINGS_HOOK" add "$HOOK_CMD"
SETTINGS_FILE="${GSTACK_SETTINGS_FILE:-$HOME/.claude/settings.json}"
...
echo "$SNIPPET" >> "$CLAUDE_MD"
cat > "$HOOKS_DIR/check-gstack.sh" << 'HOOK_EOF'
```

**Risk:** Global and project-level agent behavior altered by scripts.

**Safe to remove?** Yes for most core skills; team-mode automation/enforcement features break.

**Recommendation:** Avoid `--team` and avoid running `gstack-team-init` in hardened setup.

---

## [SEVERITY: MEDIUM]

**Category:** Browse / File Access  
**File:** `browse/src/browser-manager.ts` + `browse/src/server.ts`  
**Line(s):** `browser-manager.ts` 189, 208, 286-287; `server.ts` 12, 187, 383, 452-454  
**Finding:** Browse stores persistent browser/session state under `~/.gstack/` and may disable sandbox in CI/containers/Windows.

**Code:**
```ts
if (process.env.CI || process.env.CONTAINER) launchArgs.push('--no-sandbox');
chromiumSandbox: process.platform !== 'win32'
const userDataDir = path.join(process.env.HOME || '/tmp', '.gstack', 'chromium-profile');
const SESSIONS_DIR = path.join(process.env.HOME || '/tmp', '.gstack', 'sidebar-sessions');
```

**Risk:** Cookie/session persistence at rest; weaker sandbox in some environments.

**Safe to remove?** Partial. Removing persistence/sandbox exceptions may break compatibility.

**Recommendation:** Periodically wipe `~/.gstack/chromium-profile` + session dirs; run in trusted local environment only.

---

## [SEVERITY: MEDIUM]

**Category:** Browse / Cookie Access  
**File:** `browse/src/cookie-import-browser.ts`  
**Line(s):** 2-3, 11-17, 125-143, 195-199  
**Finding:** `/setup-browser-cookies` and browse cookie import read local Chromium cookie databases/profiles.

**Code:**
```ts
read and decrypt cookies from real browsers
... ~/Library/Application Support/<browser>/<profile>
... ~/.config/<browser>/<profile>
...
const prefs = JSON.parse(fs.readFileSync(prefsPath, 'utf-8'));
const email = prefs?.account_info?.[0]?.email;
```

**Risk:** Access to authenticated browser sessions and profile metadata.

**Safe to remove?** Yes; only cookie import/authenticated browsing convenience is lost.

**Recommendation:** Remove `/setup-browser-cookies` skill and disable `cookie-import-browser` command for hardened install.

---

## [SEVERITY: LOW]

**Category:** Conductor  
**File:** `README.md`, `conductor.json`  
**Line(s):** `README.md` 287; `conductor.json` 1-5  
**Finding:** Conductor is referenced in docs and has a local config file, but no automatic install code found.

**Code:**
```md
[Conductor](https://conductor.build) runs multiple Claude Code sessions in parallel ...
```

```json
{ "scripts": { "setup": "bin/dev-setup", "archive": "bin/dev-teardown" } }
```

**Risk:** Promotional/reference surface only.

**Safe to remove?** Yes. No core runtime dependency detected.

**Recommendation:** Delete `conductor.json` and redact Conductor paragraph in README for your fork.

---

## [SEVERITY: INFO]

**Category:** Bun install references in skill docs/templates  
**File:** `scripts/resolvers/browse.ts` (generator) and generated `SKILL.md` files  
**Line(s):** `scripts/resolvers/browse.ts` 117-137  
**Finding:** Skill instructions include optional Bun bootstrap snippet with checksum verification, after explicit user approval prompt.

**Code:**
```text
Tell the user ... "OK to proceed?" Then STOP and wait.
if ! command -v bun ...
  curl -fsSL "https://bun.sh/install" -o "$tmpfile"
  ... verify checksum ...
```

**Risk:** If agent follows instructions, Bun may be installed by command execution.

**Safe to remove?** Yes; browse setup helper text only.

**Recommendation:** Remove this block from templates if you want zero install prompts.

---

## Setup Script (`./setup`) — Step-by-step

1. Hard-fails if Bun missing; prints manual Bun install command (does not auto-run Bun installer).  
2. Parses host flags and may early-exit for `openclaw`, `hermes`, `gbrain`.  
3. Builds browse binary via `bun install` + `bun run build` when needed.  
4. Optionally installs `coreutils` via Homebrew on macOS when missing timeout/gtimeout.  
5. Ensures Playwright Chromium can launch; if not, runs `bunx playwright install chromium`.  
6. On Windows fallback path, runs `npm install --no-save playwright` and `npm install --no-save @ngrok/ngrok` if unresolved.  
7. Creates `~/.gstack/projects`.  
8. Installs/symlinks skills for selected host(s) under `~/.claude/skills`, `~/.codex/skills`, `~/.kiro/skills`, `~/.factory/skills`, `~/.config/opencode/skills`.
9. Writes migration/version markers in `~/.gstack/` and can register/remove SessionStart hook in `~/.claude/settings.json` when `--team`/`--no-team` used.

Offline behavior: setup generally fails or partially degrades once it needs dependency install, Playwright download, git pull/update checks, or doc generation from missing deps.

---

## Bin Directory Audit (every file)

Legend: N=network call detected, W=writes outside repo/gstack install root detected.

| Script | Purpose | N | W | Safe to remove? |
|---|---|---:|---:|---|
| `bin/chrome-cdp` | Launch local Chrome with CDP | local-only curl | writes `~/.gstack/cdp-profile` | Yes (debug helper). |
| `bin/dev-setup` | Local contributor setup | No remote | writes repo `.claude/skills` | Yes (dev-only). |
| `bin/dev-teardown` | Undo local contributor setup | No | removes repo `.claude` links | Yes (dev-only). |
| `bin/gstack-analytics` | Local analytics viewer | No | reads `~/.gstack/analytics` | Yes (dashboard only). |
| `bin/gstack-builder-profile` | Shim to developer profile | No | reads/writes `~/.gstack` | Yes. |
| `bin/gstack-codex-probe` | Codex helper + local telemetry write | No remote | writes `~/.gstack/analytics` | Yes (Codex helper breaks). |
| `bin/gstack-community-dashboard` | Pull aggregate stats | **Yes** Supabase | none local besides stdout | Yes. |
| `bin/gstack-config` | Read/write config | No | writes `~/.gstack/config.yaml` | No (core config). |
| `bin/gstack-developer-profile` | Developer profile storage | No | writes `~/.gstack/developer-profile.json` | Optional. |
| `bin/gstack-diff-scope` | Diff categorization | No | temp/local only | Optional. |
| `bin/gstack-extension` | Extension helper text/state lookup | No remote | reads `.gstack/browse.json` | Optional. |
| `bin/gstack-global-discover.ts` | Global repo/session discovery | No network in file | reads `~/.claude/projects` etc | Optional. |
| `bin/gstack-learnings-log` | append learnings | No | writes `~/.gstack/projects/*/learnings.jsonl` | Optional. |
| `bin/gstack-learnings-search` | query learnings | No | reads `~/.gstack/projects/*` | Optional. |
| `bin/gstack-model-benchmark` | provider benchmark CLI | provider-dependent | writes benchmark output | Optional. |
| `bin/gstack-open-url` | open URL helper | opens browser | none | Optional. |
| `bin/gstack-patch-names` | patch skill frontmatter names | No | edits skill files | Setup helper. |
| `bin/gstack-platform-detect` | detect installed agents | No | none | Optional. |
| `bin/gstack-question-log` | log AskUser events | No | writes `~/.gstack/projects/*` | Optional. |
| `bin/gstack-question-preference` | read/write preference | No | writes `~/.gstack/projects/*` | Optional. |
| `bin/gstack-relink` | rebuild skill symlinks | No | writes `~/.claude/skills/*` | Setup maintenance. |
| `bin/gstack-repo-mode` | detect solo/team mode | No | caches `~/.gstack/projects/*` | Optional. |
| `bin/gstack-review-log` | review event log | No | writes `~/.gstack/projects/*/reviews.jsonl` | Optional. |
| `bin/gstack-review-read` | read review log | No | reads `~/.gstack` | Optional. |
| `bin/gstack-session-update` | background git pull/setup | **Yes** (`git pull`) | writes `~/.gstack/*` | Yes if no team auto-update desired. |
| `bin/gstack-settings-hook` | mutate `~/.claude/settings.json` hooks | No remote | **writes `~/.claude/settings.json`** | Optional (team mode feature). |
| `bin/gstack-slug` | derive project slug cache | No | writes `~/.gstack/slug-cache` | Optional. |
| `bin/gstack-specialist-stats` | compute review stats | No | reads `~/.gstack` | Optional. |
| `bin/gstack-taste-update` | update taste profile | No | writes `~/.gstack/projects/*/taste-profile.json` | Optional. |
| `bin/gstack-team-init` | scaffold team policy files | No remote | **writes project `CLAUDE.md`, `.claude/*`, `.gitignore`** | Optional, avoid for strict mode. |
| `bin/gstack-telemetry-log` | local telemetry writer + trigger sync | Indirect via sync | writes `~/.gstack/analytics` | Yes (disables telemetry pipeline). |
| `bin/gstack-telemetry-sync` | upload telemetry batch | **Yes** Supabase | writes sync cursor/rate files | Yes (privacy-friendly). |
| `bin/gstack-timeline-log` | timeline logger | No | writes `~/.gstack/projects/*/timeline.jsonl` | Optional. |
| `bin/gstack-timeline-read` | timeline reader | No | reads `~/.gstack/projects/*` | Optional. |
| `bin/gstack-uninstall` | remove installs/state | No | removes many local dirs | Keep for cleanup. |
| `bin/gstack-update-check` | version + install ping | **Yes** GitHub raw + optional Supabase | writes cache in `~/.gstack` | Yes (disable update checks). |

---

## Prompt-injection / unsafe instruction vectors (skills/templates)

1. **Routing injection to project CLAUDE.md + commit command** (see finding above).  
2. **Telemetry run-last mandate** in generated completion sections (`PLAN MODE EXCEPTION — ALWAYS RUN`).  
3. **Installation helper blocks** in browse setup snippets include Bun install instructions (prompted, not unconditional).  
4. No direct instruction found to exfiltrate `.env`/API keys; however broad bash access + skill prompts can still execute destructive commands if user/agent accepts.

---

## Browse binary specific answers

- **What it does:** persistent Playwright Chromium daemon, local HTTP command server, snapshot/automation/cookie import support.  
- **Chromium flags/sandbox:** enables `--no-sandbox` in CI/container; disables chromium sandbox on Windows (`chromiumSandbox: process.platform !== 'win32'`).  
- **External transmit:** primary browsing traffic is to visited sites; optional tunnel/remote-agent paths exist in browse server/CLI flows; telemetry is handled by separate bin scripts.  
- **Cookie/session data:** stores browser profile/session artifacts under `~/.gstack/chromium-profile` and `~/.gstack/sidebar-sessions`; cookie import reads installed browser cookie DBs.  
- **Bun requirement:** core browse build/runtime path depends on Bun for setup/build and compiled binary generation.

---

## Summary Table 1: Software Installation Attempts

| File | Software | Auto/Manual | Required? | Safe to Remove? |
|---|---|---|---|---|
| `setup` | Bun (required check + printed installer cmd) | Manual command printed; Bun itself hard requirement | Yes for normal setup | No (setup exits without Bun). |
| `setup` | Playwright Chromium (`bunx playwright install chromium`) | **Auto** when browser missing | Required for `/browse` | No for browse; yes if disabling browse entirely. |
| `setup` | `coreutils` (`brew install coreutils`) | **Auto** on macOS if timeout missing | Optional | Yes (degrades codex timeout protection). |
| `setup` | `playwright`, `@ngrok/ngrok` via npm (Windows fallback) | **Auto fallback** | Windows-specific optional fallback | Yes (may break Windows browse/pair-agent). |
| `scripts/resolvers/browse.ts` → generated skills | Bun installer snippet (`curl ... bun.sh/install`) | Manual, with explicit prompt in instructions | Optional helper | Yes. |
| `bin/dev-setup` | `bun install` | Auto when contributors run script | Dev-only | Yes (dev helper only). |
| `bin/gstack-team-init` | `git clone` instructions for gstack | Manual instructions | Team onboarding only | Yes. |
| `pair-agent/SKILL.md.tmpl` | ngrok install guidance | Manual instructions | Optional remote pairing | Yes (remote pairing disabled). |

---

## Summary Table 2: Network Calls

| File | Destination | Purpose | Can Disable? |
|---|---|---|---|
| `bin/gstack-telemetry-sync` | `${SUPABASE_URL}/functions/v1/telemetry-ingest` | Upload telemetry batch | Yes (`telemetry off`, remove script). |
| `bin/gstack-update-check` | `https://raw.githubusercontent.com/garrytan/gstack/main/VERSION` | Check latest version | Yes (`update_check false`). |
| `bin/gstack-update-check` | `${SUPABASE_URL}/functions/v1/update-check` | Community update ping | Yes (`telemetry off`). |
| `bin/gstack-community-dashboard` | `${SUPABASE_URL}/functions/v1/community-pulse` | Read aggregate stats | Yes (don’t run command). |
| `setup` | Bun/NPM/Playwright registries + GitHub/Homebrew (via invoked tools) | Dependency + browser/runtime install | Partially (flags/env + script edits). |
| `browse` runtime | Arbitrary user-requested URLs | Browser automation traffic | Only by not using browse. |

---

## Summary Table 3: Files Written Outside gstack/

| Path | Purpose | Created By | Can Prevent? |
|---|---|---|---|
| `~/.gstack/projects/` | project state/logs | `setup`, many `bin/*`, skill preambles | Yes (remove/patch telemetry+state features). |
| `~/.gstack/analytics/` | telemetry + analytics JSONL/cursors | skill preambles, `gstack-telemetry-*` | Yes (`telemetry off` + remove binaries). |
| `~/.gstack/sessions/` | session markers | generated preambles | Requires template edits. |
| `~/.gstack/chromium-profile/` | persistent browse profile | browse runtime | Avoid browse or patch storage path. |
| `~/.gstack/sidebar-sessions/` | sidebar session state/chat | browse server | Avoid sidebar features. |
| `~/.claude/skills/*` | skill symlinks/install | `setup`, `gstack-relink` | Don’t run setup (or run modified). |
| `~/.claude/settings.json` | SessionStart hook registration | `setup --team`, `gstack-settings-hook` | Avoid team mode / remove hook tool. |
| `<repo>/CLAUDE.md` | injected routing/team snippets | skill routing injection, `gstack-team-init` | Yes (decline prompts/remove generator). |
| `<repo>/.claude/settings.json` | PreToolUse enforcement hook | `gstack-team-init required` | Yes (don’t run required mode). |
| `<repo>/.claude/hooks/check-gstack.sh` | enforcement script | `gstack-team-init required` | Yes (don’t run required mode). |
| `<repo>/.gitignore` | add vendored ignore rule | `gstack-team-init` | Yes (don’t run team-init). |

---

## Summary Table 4: Conductor References

| File | Reference Type | Required? | Safe to Remove? |
|---|---|---|---|
| `README.md` | Documentation mention + link to `conductor.build` | No | Yes. |
| `conductor.json` | Local scripts mapping (`dev-setup`, `dev-teardown`) | No | Yes. |
| `docs/designs/GSTACK_BROWSER_V0.md` | design doc mention | No | Yes. |

---

## Direct answers to your strict preferences

- **No Conductor:** You can safely delete `conductor.json` and scrub Conductor docs/references; no core runtime dependency found.
- **Telemetry fully disabled:** Set `gstack-config set telemetry off`; additionally disable upload by removing execute bits from `bin/gstack-telemetry-log` and `bin/gstack-telemetry-sync` for defense-in-depth.
- **Approve installs manually:** Do not run stock `./setup` unattended. Use a patched setup that removes auto `bun install`, `bunx playwright install chromium`, `brew install`, npm fallback installs.

