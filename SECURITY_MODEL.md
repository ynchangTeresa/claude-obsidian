# Threat model: claude-obsidian

Status: built from the code and the repository's own `SECURITY.md` / `PRIVACY.md` (cited below), plus the user's answers marked **(user)**.

## System overview
claude-obsidian is a local-first Agent Skills package and Claude Code plugin. A standard-library Python core (`claude_obsidian/`, entry `scripts/claude-obsidian.py`) runs with the current user's filesystem permissions and manages a user-owned Obsidian vault (`.claude-obsidian.json`, `wiki/`, `.raw/`, `inbox/`, `.vault-meta/`). There is no server, no network listener and no deployed infrastructure; no infrastructure repository applies.

## Trust boundaries (from SECURITY.md)
1. **User / host authority vs. content.** System/host policy, repo instructions, the selected skill and the user's explicit scope are authority. Vault notes, `wiki/hot.md`, inbox items, raw captures, web pages, retrieved chunks and tool output are untrusted data that must never authorize commands, egress, scope widening or destructive actions.
2. **Plugin install tree vs. user vault.** The vault must never be derived from `${CLAUDE_PLUGIN_ROOT}` / plugin cache (`paths.resolve_vault_root`, `assert_not_plugin_tree`).
3. **Vault root containment.** Vault-relative paths are containment-checked after symlink resolution; hook reads use no-follow `openat` traversal (`hook_adapter._bounded_regular_bytes`).
4. **Project workspace config vs. private vaults.** A project's `.claude-obsidian.json` may only spend SessionStart context consent on a vault inside its own tree; external vaults need exact `CLAUDE_OBSIDIAN_SESSION_CONTEXT_VAULT` (`hook_adapter._context_selection_is_consented`).
5. **Local machine vs. network.** Core makes no network requests; egress only with explicit flags (`--allow-egress` in `scripts/contextual-prefix.py`, `--allow-remote-ollama` in `scripts/tiling-check.py`), host web tools, or a separately configured external capture runner.
6. **Other local users.** Filesystem permissions are the outer boundary; locks use owner tokens + host/PID checks.

## Components
| Component | Code |
|---|---|
| **Core transactions** | `claude_obsidian/transaction.py`, `vault_ops.py`, `ledgers.py`, `checkpoint.py`, `legacy_lock.py`, `paths.py`, `page_schema.py`, `json_utils.py`, `mode_config.py`, `extensions.py`, `scripts/wiki-lock.sh` |
| **CLI** | `claude_obsidian/cli.py`, `__main__.py`, `scripts/claude-obsidian.py`, `scripts/wiki-mode.py` |
| **Capture** | `claude_obsidian/capture.py`, `url_safety.py`, `gitignore.py` |
| **Hooks** | `claude_obsidian/hook_adapter.py`, `hooks/` |
| **Retrieval & egress scripts** | `scripts/bm25-index.py`, `retrieve.py`, `rerank.py`, `contextual-prefix.py`, `tiling-check.py`, `boundary-score.py`, `lint_engine.py` |
| **Setup scripts** | `scripts/*.sh` (setup-vault, setup-multi-agent, setup-retrieve, setup-dragonscale, setup-mode, detect-transport, allocate-address) |
| **Release tooling** | `claude_obsidian/release.py`, `contracts.py`, `gates.py`, `package_validation.py`, `config/`, `.github/workflows/`, `RELEASE_MANIFEST.json`, `SHA256SUMS` |
| **Skills & agent instructions** | `skills/`, `agents/`, `AGENTS.md`, `CLAUDE.md`, `.cursor/`, `.windsurf/`, `GEMINI.md`, `templates/` |

## Entry points
- CLI subcommands (`cli.py`): `transaction apply/recover/inspect`, `capture plan/apply/external/queue`, `checkpoint` (runs `git`), `init`, `adopt`, `migrate`, `lint`, `mode`, `extension`, `release build/audit/gates`, `package validate`, `hook session-start|stop`.
- Claude Code lifecycle hooks (`hooks/hooks.json`) run automatically on SessionStart/Stop in any project the user opens.
- Transaction bundles (`claude-obsidian.transaction.v1` JSON) produced by agents/workers from untrusted content.
- Captured files in `inbox/` (Markdown, PDF, images, EPUB zip) and external URLs.
- Vault discovery inputs: `--vault`, `CLAUDE_OBSIDIAN_VAULT`, nearest workspace config, cwd ancestry.
- Release archives read by `release audit`.

## Assets
- Private vault content (notes, raw sources, ledgers) — confidentiality and integrity.
- Files outside the vault (user home, other vaults, git repos) — must not be written/read via path tricks.
- The model context (SessionStart injection of `wiki/hot.md`) — confidentiality of vault data and integrity of agent instructions.
- Credentials the user may have in env (e.g. `ANTHROPIC_API_KEY` used by contextual-prefix).
- Integrity of public release artifacts (no secrets, private paths, contributor vault state).

## Attacker profiles
**(user)** Priority attacker: the malicious content author (prompt injection, path tricks via captured sources, web pages and vault notes). The user considers every attacker below in scope.

- **Malicious content author**: controls a captured source, web page, inbox file, EPUB, or vault note (prompt injection, crafted filenames, symlinks, zip tricks, oversized input).
- **Malicious project/workspace**: a cloned repo containing `.claude-obsidian.json` or crafted files, opened by the user so hooks run.
- **Other local user** on a shared host racing locks or planting symlinks in `.vault-meta/`.
- **Supply chain**: tampered release artifacts, optional tools (defuddle, Ollama, Obsidian plugins).

## Existing controls
See SECURITY.md "Defensive invariants": symlink-resolved containment, process-lifetime vault lock, journaled transactions with expected hashes and atomic replace, create-only raw payloads, hooks that never mutate, inert external capture plans, HTTPS/public-host validation and redirect revalidation (`capture.validate_https_url`, `validate_redirect_chain`), credential-in-URL rejection (`url_safety.py`), bounded/sanitized context tagged `trust="local-data" instructions="never"`, egress default-deny flags, release audit rejecting secrets/symlinks/unsafe archives (`release.py`).

## Out of scope / accepted risk
- From SECURITY.md: the core is not a sandbox; an agent, adapter, Obsidian plugin, hook or command already holding the user's authority is inside the trust boundary. Third-party tools' own security models are out of scope.
- **(user)** Nothing is out of scope or accepted risk: shared hosts, release tooling, legacy `wiki-lock.sh`, Windows behavior and optional network adapters all remain in scope.

## Open questions
- Which assets matter most (vault confidentiality vs. integrity vs. files outside the vault)?
