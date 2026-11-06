# Personal Agent Skill Sync Across Agents and Machines

**Status: proposal (RFC) for issue #1755.** Nothing here is implemented yet.
The design builds on the shipped `backup-agents` / `restore-agents` commands
([`design-backup-agent-assets.md`](design-backup-agent-assets.md)), as the
maintainer asked in the issue: reuse their skill discovery, manifest, integrity
and restore guards, and add only a transport plus a conflict and version
protocol.

## 1. Problem

A user who runs several harnesses (Codex, Claude Code, GitHub Copilot CLI, ...)
on several computers (Linux, macOS, Windows, WSL) keeps personal Agent Skills
in step by hand today. They copy directories, symlink each agent's skill root to
a shared tree, or maintain a private git repository and pull it on every
machine. Installation targets, conflict handling and dependency checks stay
manual, and both sides of a pair of machines can drift offline and overwrite
each other.

ai-memory already knows where each harness reads skills from
(`install_skills.rs`, `backup_agents.rs`) and already knows how to restore a
skill package safely (`restore_agents.rs`). What it lacks:

- a way to say *which* personal skills are shared and which stay local;
- a shared store with history;
- a per-machine record of what was last synced, so a local edit and a remote
  edit can be told apart and neither is lost.

## 2. Goals and non-goals

Goals:

1. **Explicit opt-in per skill.** Nothing is shared until the user adds it.
   Skills that are not added are never read, moved or rewritten.
2. **Complete packages.** A skill is its whole directory: `SKILL.md` plus
   scripts, references and assets, with portable Unix modes.
3. **Per-agent install destinations** on Linux, macOS, Windows and WSL, with
   Codex, Claude Code and Copilot CLI first, and every other harness that
   `install-skills` knows as follow-up.
4. **No silent overwrite.** A local edit and a remote edit to the same skill
   are a conflict that the user resolves explicitly.
5. **Status and history.** The user can see what is in sync, what changed and
   where, list earlier versions of a skill, and roll one back.
6. **Declared dependencies.** A skill can declare the commands, environment
   variables and agents it needs; the tool reports what is missing on this
   machine instead of pretending a copy makes the skill usable.
7. **Secrets stay out.** Pushing refuses content the sanitizer flags and
   files that look like credentials.

Non-goals:

- Syncing MCP configs, hooks, plugins or instruction files. `backup-agents`
  covers those as a snapshot; live sync of them is a different problem.
- Taking over plugin-owned skills (`~/.claude/plugins`, `~/.codex/plugins`,
  marketplace installs) or the ai-memory managed skills that `install-skills`
  writes. Their installers own them.
- Real-time sync. Sync runs when the user runs it (or from a hook they opt
  into later).
- A hosted service. The user brings the remote.

## 3. Where it lives: core CLI, client side

`backup-agents` and `restore-agents` are client-side commands in the core CLI
that never touch the server. Skill sync is the same kind of operation and needs
their internals (discovery tables, path validation, symlink checks, atomic
write with rollback), so it belongs next to them in `ai-memory-cli`, not in a
companion. A companion would have to copy that code, which is what the
maintainer asked to avoid.

It adds **no MCP tool and no server route** in the first phases, so the tool
count, `MEMORY_INSTRUCTIONS`, the routing snippet and the TypeScript SDK stay
unchanged. An agent can still run the CLI.

## 4. Backing store: a git repository the user owns

The shared store is a **skill collection**: a git working copy whose remote the
user chooses (a private GitHub/GitLab repository, a bare repo on a NAS, a
directory on a synced drive). Git gives history, restore, offline commits,
three-way merges and authentication through the user's existing SSH keys and
credential helpers.

Alternatives considered:

| Option | Why not first |
| :--- | :--- |
| ai-memory server as the store (wiki pages or a new table) | The server binds to loopback by default; syncing two machines would require exposing it. Skills carry binary assets and executable scripts, which the page model and sanitizer are not built for. Could be a later transport. |
| `backup-agents` tarballs on a synced drive | No history, no merge base, last writer wins. |
| A dotfiles manager | Leaves per-agent destinations, conflict and dependency handling to the user, which is the issue. |

### Collection layout

```
<collection>/
├── .gitattributes            # "* -text": bytes are stored verbatim, no CRLF rewriting
├── collection.toml           # schema version + per-skill metadata
└── skills/
    └── <skill-name>/
        ├── SKILL.md
        └── scripts/...
```

`collection.toml`:

```toml
version = 1

[skills.pr-review]
# Agents to install into; omitted = every agent selected on this machine.
agents = ["claude-code", "codex", "copilot-cli"]
# Extra excludes on top of the built-in deny list (section 8).
exclude = ["fixtures/large/**"]

[skills.pr-review.requires]
commands = ["gh", "jq"]
env = ["GITHUB_TOKEN"]        # names only; values never enter the collection
```

`.gitattributes` with `* -text` matters for the Windows/WSL case: without it a
`core.autocrlf=true` checkout on Windows rewrites line endings, every skill
looks locally modified, and digests stop matching.

### Transport: the system `git` binary

Push and pull shell out to `git` (fetch, merge, push) in the collection
directory. That reuses whatever authentication already works for the user
(SSH agent, `gh auth`, credential managers) without ai-memory handling
credentials. The wiki's `git2` (vendored libgit2) stays in use for local
operations that need no network: reading trees at a revision for `history` and
`restore`, and computing the merge base. If `git` is missing, local commands
still work and `push`/`pull` fail with a message naming the missing binary.

This would be the first runtime dependency of the core binary on the `git`
executable (today only tests call it). The alternative is network operations
through `git2`, which means implementing SSH-agent and credential-helper
callbacks inside ai-memory. Open question 6 asks which trade the maintainers
prefer.

## 5. CLI surface

One subcommand group, `ai-memory skills`:

```bash
# Create or adopt a collection, optionally with a remote
ai-memory skills init [--collection DIR] [--remote URL]

# Share a local skill (copied into the collection and committed)
ai-memory skills add <name> [--from-agent claude-code] [--agents claude-code,codex]
ai-memory skills remove <name> [--uninstall]        # stop sharing; local copies stay unless --uninstall

# What is shared, where it is installed, what differs, what is missing
ai-memory skills status [--json]

# Fetch + merge the remote, then install into each agent root (dry run by default)
ai-memory skills pull [--apply]
# Commit local edits made in agent roots back to the collection, then push
ai-memory skills push [--apply]

# Explicit conflict resolution
ai-memory skills resolve <name> --keep local|remote|<agent>

# Version history and rollback of one skill
ai-memory skills history <name>
ai-memory skills restore <name> --to <revision> [--apply]
```

Like `restore-agents`, every command that writes into an agent root is a dry
run unless `--apply` is passed, and prints the plan first.

The collection path comes from `--collection` or a new config key (for example
`[skills] collection = "..."` with an `AI_MEMORY_SKILLS_COLLECTION` env
equivalent), documented in `config.default.toml` and
`docs/config-properties.md` as AGENTS.md requires. The default could be
`<config_dir>/skills-collection`. **Open question** (section 12).

## 6. Destinations per agent

Destinations come from the tables that already exist; no third skill-root
list is introduced. The first three agents:

| Agent | Global root (Linux / macOS / WSL) | Windows | Source of truth today |
| :--- | :--- | :--- | :--- |
| Claude Code | `$CLAUDE_CONFIG_DIR/skills`, default `~/.claude/skills` | `%USERPROFILE%\.claude\skills` | `install_skills.rs`, `backup_agents.rs` |
| Codex | `~/.agents/skills` (shared with Qwen, Qoder) | `%USERPROFILE%\.agents\skills` | same |
| Copilot CLI | `$COPILOT_HOME/skills`, default `~/.copilot/skills` | `%USERPROFILE%\.copilot\skills` | `install_skills.rs` only |

Two gaps in the shipped commands surface here:

- `backup-agents` did not collect Copilot CLI's skill roots, although
  `install-skills --agent copilot-cli` writes there. Fixed alongside this
  proposal: it now archives `$COPILOT_HOME/skills` and `.github/skills`, and
  `restore-agents` restores them.
- Several agents share one physical root (`~/.agents/skills`). Sync installs
  into each **physical** root once, after canonicalising roots, so a shared
  root is never written twice per run.

Project-scope skills (`.claude/skills`, `.agents/skills`, `.github/skills` in a
repository) are out of scope: they already travel with the repository.

## 7. Sync protocol

### Identity and digests

A skill's content identity is a **tree digest**: SHA-256 over the sorted list
of `(relative path, portable mode, SHA-256 of file bytes)` for every file the
skill contains after excludes. The same function runs on the collection copy
and on every installed copy, reusing the per-file SHA-256 that
`backup-agents` already records.

### Per-machine state

Each machine records, per skill and per physical destination root, the tree
digest it last installed or committed (the **base**), and the collection commit
it came from. The state file lives inside the collection's `.git` directory
(`.git/ai-memory/skills-state.json`), so it never reaches a commit and each
working copy (each `$HOME`, which is what a machine means here, see section 9)
has its own.

### Three-way decision per (skill, root)

With `B` = base digest, `L` = digest of the installed copy, `R` = digest in
the collection after `pull` merges the remote:

| Case | Meaning | Action |
| :--- | :--- | :--- |
| `L = B`, `R = B` | in sync | nothing |
| `L = B`, `R ≠ B` | remote changed | `pull --apply` installs `R` |
| `L ≠ B`, `R = B` | local edit | `push --apply` commits `L` to the collection |
| `L ≠ B`, `R ≠ B`, `L = R` | same edit on both sides | record the new base |
| `L ≠ B`, `R ≠ B`, `L ≠ R` | **conflict** | refuse; `resolve` decides |
| installed copy missing, `B` set | deleted locally | report; `push` records the removal only with `--propagate-deletes` |
| no `B`, installed copy exists and differs | unmanaged same-name skill | refuse, like `install-skills` refuses an unmanaged file |

The same skill installed into two roots on one machine (Claude Code and Codex)
can also diverge. `push` refuses when two local roots carry different edits of
one skill and asks `resolve --keep <agent>`.

Inside the collection, two machines that pushed different edits produce a git
merge. A clean textual merge is accepted; a conflicting one stops `pull`
before anything is installed, leaves the collection in the usual git conflict
state, and `status` points at it. ai-memory never auto-resolves a git conflict.

### Writing into agent roots

Installs reuse the `restore-agents` write path: allowlisted destination,
traversal and symlink checks below the root, `ai_memory_wiki::write_atomic`,
rollback of earlier writes when a later one fails, and a timestamped copy of
any file that is overwritten. Files removed from a skill upstream are moved
aside the same way, never deleted outright.

## 8. Secrets and active content

`push` refuses to commit a skill when:

- the existing sanitizer (`ai_memory_core::sanitize`) would change any UTF-8
  file in it, meaning it contains something shaped like a token, key or
  bearer header; the report names the file and the rule, never the value;
- it contains a file on the built-in deny list: `.env*`, `*.pem`, `*.key`,
  `id_rsa*`, `id_ed25519*`, `*.p12`, `credentials*`, `*.sqlite`, plus the
  build and cache directories `backup-agents` already skips (`node_modules`,
  `target`, `venv`, `__pycache__`, ...).

There is no override flag in the first version; the fix is to move the secret
into an environment variable and declare its name under `requires.env`.

Pulled skills are active content: scripts run on the machine and `SKILL.md`
instructs the model. `pull` prints which skills and files change before
`--apply`, the same warning `restore-agents` prints. The collection is trusted
exactly as much as its remote.

## 9. Links, Windows and WSL

From the sync's point of view each `$HOME` is a machine: a Windows profile and
the WSL distribution on the same computer are two machines with two working
copies (or one remote and two clones) and two state files. That keeps the
three-way protocol correct when both sides drift offline, which is the
scenario raised in the issue comments.

Link handling:

- Today `backup-agents` skips a skill root that is a symlink, and
  `restore-agents` refuses to write through one. Users who point
  `~/.claude/skills` at `~/.agents/skills` (a symlink on Unix, a junction on
  Windows) would get no sync at all.
- Sync accepts a **root** that is a link when its canonical target is a
  directory inside the same home (or `CLAUDE_CONFIG_DIR` / `COPILOT_HOME`), and
  treats it as the target's physical root. It writes into the target and
  **never replaces the link with a directory**. Symlinks *below* a root stay
  refused, as in `restore-agents`.
- Root canonicalisation is what deduplicates writes, so a junctioned
  `~/.claude/skills` and `~/.agents/skills` are one root and one write.
- Whether Rust reports a Windows junction as a symlink through
  `symlink_metadata` has to be confirmed in the Windows workflow before the
  link rule ships; the tests must cover a junction, not only a symlink.

## 10. Dependencies and machine-specific configuration

`collection.toml` may declare, per skill, `requires.commands`,
`requires.env` and `agents`. `status` checks each one on this machine:
commands on `PATH`, environment variable names set (values are never read
into output), agents detected. It reports missing items per skill. The skill is
still installed unless the user excludes it on this machine:

```bash
ai-memory skills status
#   pr-review   in sync   claude-code, codex, copilot-cli
#               missing: command `gh`
```

A per-machine exclusion (`skills pull --skip <name>`, remembered in the state
file) keeps a skill out of a machine where it cannot work, without removing it
from the collection.

## 11. Upgrade path and compatibility

- Purely additive: no change to the server, the wiki, SQLite, existing config
  keys or the `backup-agents` manifest schema. Existing installs see nothing
  until `skills init` runs.
- Adding the Copilot CLI root to `backup-agents` only adds entries to new
  archives; old archives restore as before.
- The first `skills add` of a skill that is already installed in several roots
  with different content refuses and lists the copies, so adopting an existing
  manual setup never picks a winner silently.
- Adopting an existing personal skills git repository: `skills init
  --collection <its clone>` accepts a repository without `collection.toml` and
  offers to generate one from the `skills/` directories it finds.
- `collection.toml` carries `version`; a newer version than the binary
  understands makes every write command refuse with an upgrade message.

## 12. Open questions

1. **Default collection location and the config key name.** `[skills]
   collection` under `config.toml`, or flag/env only?
2. **`push` scope.** Should `push` only commit edits to shared skills, or also
   offer to add new unshared skills it finds? Proposal: never; adding is always
   `skills add`.
3. **Deletes.** Is `--propagate-deletes` enough, or should removing a skill on
   one machine always be a `skills remove` in the collection?
4. **Agents beyond the first three.** Every root in `install_skills.rs`
   (Antigravity, Gemini, Devin, Grok, Qwen, Hermes, ...) can follow once the
   first three are proven. Kiro's `~/.kiro/agents/*.json` are not Agent Skills
   and stay out.
5. **Hook-driven sync.** A session-start `pull` (dry run, notice only) could
   come later; it must respect the hook time budget (≤200 ms), so it would be
   a background job, not part of the hook itself.
6. **Transport implementation.** Shell out to the user's `git` (reuses their
   authentication, adds a runtime dependency for `push`/`pull` only) or use
   `git2` network operations (self-contained, but ai-memory must handle SSH
   and HTTPS credentials)?

## 13. Phases

Each phase is a separate pull request with its own tests, CHANGELOG entry and
docs.

1. **Shared foundation (refactor + fix).** Move skill-root discovery, path
   validation, symlink checks and the atomic write/rollback out of
   `backup_agents.rs` / `restore_agents.rs` into a shared module, with no
   behaviour change, and add the tree-digest function, in the same pull
   request as their first caller (phase 2) so no unused surface ships. The
   Copilot CLI roots in `backup-agents` / `restore-agents` already landed as
   a standalone fix.
2. **Local collection.** `skills init`, `add`, `remove`, `status`, and `pull
   --apply` against a collection that has no remote: install from collection
   into roots, base tracking, the conflict table, secret gate on `add`.
   Usable already with manual `git push`/`git pull` in the collection.
3. **Transport and history.** `push`, `pull` with fetch/merge through `git`,
   `resolve`, `history`, `restore`, delete handling.
4. **Dependencies and per-machine exclusions.** `requires`, `--skip`,
   `status` reporting.
5. **Links and Windows/WSL.** Link-root acceptance, junction tests in the
   Windows workflow, `.gitattributes` enforcement.

## 14. Tests the implementation needs

- Round trip: `add` on machine A, `pull --apply` on machine B (two temp homes,
  one bare remote), byte-for-byte equal trees, modes preserved.
- Every row of the three-way table, including both conflict cases, with the
  assertion that no file in an agent root changed on refusal.
- Two local roots with divergent edits of one skill.
- Shared physical root (`~/.agents/skills` for Codex and Qwen) written once.
- Unmanaged same-name skill refused; plugin-owned and ai-memory managed
  skills never discovered.
- Secret gate: a fake token in a script and a `.env` file both block `add`
  and `push`, and the report contains no secret value.
- Traversal and symlink-below-root refusals reused from `restore-agents`
  (adversarial, with a legitimate control), and a link root accepted only when
  it resolves inside the home.
- Git fixtures follow the repository rule: `--no-gpg-sign` on every commit and
  no inherited global git config.
- Windows workflow: junction root, CRLF checkout with `* -text`.
