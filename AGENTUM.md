# Agentum Code — fork maintenance notes

This is Agentum's **thin fork** of [opencode](https://github.com/anomalyco/opencode),
bundled inside the Agentum desktop app as the built-in, zero-subscription
coding assistant. Branch `agentum` (the default here) = upstream `dev` +
the smallest possible rebrand diff.

## The whole diff (keep it this small)

- `packages/tui/src/logo.ts`, `packages/tui/src/util/presentation.ts`,
  `packages/opencode/src/cli/ui.ts` — wordmark: plain-text "agentum"
  beside the upstream "code" glyphs (three duplicated logo definitions
  upstream; all three must stay in step).
- `packages/opencode/src/index.ts` — `scriptName("agentum-code")`.
- `packages/opencode/src/cli/upgrade.ts` — self-update defaults OFF
  (the binary ships inside Agentum.app and is updated by the app's
  release cycle). Explicit `autoupdate: true` config still works.
- `packages/opencode/src/session/instruction.ts` — project rules prefer
  `CLAUDE.md` over `AGENTS.md` (upstream order reversed). Agentum's
  handoff injects a lean ~5KB CLAUDE.md brief AND the 100KB AGENTS.md
  reference; first-match-wins upstream ingested the 100KB one into every
  session. Verified live (marker-file test): sessions see the brief.
- `packages/opencode/src/session/system.ts` +
  `packages/opencode/src/session/prompt/deepseek.txt` — DeepSeek models
  get a dedicated system prompt (3.4KB vs default.txt's 8.5KB), tuned
  for V4-flash with the terse-output rules intact. Deliberately NOT
  trimmed: the tool description .txt files — permission-denied tools are
  already excluded from requests (`Permission.visibleTools`), and the
  remaining descriptions are prefix-cache-friendly, so rewriting them is
  rebase burden for little gain.
- `packages/opencode/src/session/retry.ts` +
  `packages/tui/src/routes/session/index.tsx` — the Agentum Worker's
  budget gate (429 `free_tier_limit`/`free_tier_suspended`, User LLM
  Quota design) is classified like upstream's own free tier: a
  structured retry action drives a dialog ("AI budget used up", own
  don't-show kv keys) while the Worker's `retry-after: 3600` idles the
  schedule to one attempt an hour instead of a hot retry loop.
- `AGENTUM.md` (this file).

Known cosmetic leftover: the TUI default-command help line still says
"start opencode tui".

## Building a release

```bash
cd packages/opencode
OPENCODE_VERSION=<upstream>-agentum.<n> bun run build --single --skip-install
```

Bun 1.3.14 (pinned via `packageManager`), no Go — the TUI is opentui/TS.
Output: `dist/opencode-darwin-arm64/bin/opencode`. Rename to
`agentum-code`, zip as `agentum-code-darwin-arm64.zip`, publish a GitHub
release tagged `v<version>` on this repo, then update `OC_VERSION` +
both SHA-256 pins in the desktop repo's `scripts/download-runtime.sh`.

The pre-push hook runs a FULL monorepo typecheck (30+ packages, slow,
needs packages this fork never touches). For fork pushes, run the scoped
check and bypass the hook:

```bash
bun turbo typecheck --filter=opencode --filter=@opencode-ai/tui
git push --no-verify
```

## Rebasing on upstream

```bash
git remote add upstream https://github.com/anomalyco/opencode.git  # once
git fetch upstream dev
git rebase upstream/dev agentum
```

Conflicts should only ever touch the files listed above. If a rebase
grows beyond that, the fork is getting fat — move the change to the
Agentum side (config injection via `OPENCODE_CONFIG` carries a lot; see
the desktop repo's `opencodeFundedConfig.ts`) or reconsider it.

## License

Upstream is MIT (see `LICENSE`, © 2025 opencode); the rebrand keeps the
license and ships the notice alongside the binary in the app bundle.
"opencode" is upstream's name — this fork deliberately does not present
itself as opencode, and the Agentum app credits opencode in its docs.
