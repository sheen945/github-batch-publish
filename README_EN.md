# github-batch-publish

A WorkBuddy / Claude Code skill for batch-publishing self-built skills from `~/.workbuddy/skills/` to GitHub — one skill per standalone repository — covering secret sanitization, bundling companion MCP source code, backfilling repo descriptions and topics, and a two-level post-push verification protocol.

## Introduction

This is a playbook-style skill that teaches a WorkBuddy agent how to publish skills in bulk. Rather than relying on a fire-and-forget automation script, it distills a real batch-publishing campaign (run between 2026-10-01 and 2026-10-08, with multiple pitfalls hit and fixed along the way) into a written record of **network environment facts, rate-limit behavior, sanitization rules, a six-step standard pipeline, and verification discipline**, so the agent can sidestep every known trap on each run.

The problem it solves: you've accumulated dozens of self-built skills locally and want to sync them to GitHub for backup, sharing, or version control — but a naive `git push` runs into three walls: blocked HTTPS, GitHub's secondary rate limit, and plaintext API tokens hiding inside the skills. This skill codifies the workaround and ordering for all three.

Who it's for: users who maintain a library of self-built WorkBuddy / Claude Code skills and need to push them to GitHub regularly, and the AI agents assigned to carry out the job.

## Features

- **Four iron rules up front**: clarify scope before acting; sanitize before pushing, never after; all pushes go over SSH; never touch the user's git proxy config (if the user's `127.0.0.1:7890` proxy is down, route around it with SSH instead of "fixing" it).
- **Three known pitfalls with field-tested conclusions and fixes**:
  - HTTPS push is dead: on this machine, `github.com:443` times out / gets reset, while `ssh github.com:22` and `api.github.com` work fine — so the `gh` CLI works but `git push` over HTTPS always fails. Remotes must use the `git@github.com:` form. Both `git -c http.proxy=` and `GIT_CONFIG_GLOBAL=<empty>` bypass tricks are confirmed useless — the playbook explicitly warns against retrying them.
  - GitHub secondary rate limit: after creating roughly 10 repos in a row, the account gets temporarily blocked from content creation. REST and GraphQL share the same limit pool, so switching APIs doesn't help. The fix is a wait-and-retry loop — one attempt every 10 minutes, up to 8 rounds (in practice, round 2 succeeded).
  - Plaintext secret sanitization: five must-scan regex patterns (`sk-` tokens, `gh[pousr]_` tokens, Bearer strings, KEY/SECRET assignments, phone numbers), a mandated replacement order (code assignment lines first — converted to `os.environ.get(...)` so the code still runs — then leftover tokens in docs replaced with placeholders), plus a documented false-positive case (`sk-based-asynchronous-programming`) and how a length threshold avoids it.
- **Six-step standard pipeline**: inventory & scope confirmation → build a staging area and copy skills while excluding build artifacts → sanitize and generate companion files (README.md generated from SKILL.md with the YAML frontmatter stripped, .gitignore, .env.example when needed) → bundle companion MCP source into an `mcp/` subdirectory (never databases, vector indexes, or embedding models) → create repo + push over SSH (`gh repo create --source=. --push` is explicitly banned; repo creation and push must be two separate steps) → verify.
- **Visibility criteria**: repos containing real tokens, platform-account operation skills (Xianyu, Xiaohongshu, Ocean Engine, etc.), or personal memory archives go private; generic methodology with no credentials goes public.
- **Specs for already-published MCP-bearing repos**: records the exclusion rules and layout conventions for `powerbi-crawler-data-prep` (the 97 MB `adomd/` DLL folder must be excluded, and `mcp/README.md` must tell users to fetch the DLLs from NuGet themselves), `obsidian-notes-rag`, and `memory-vault-sync`, so future updates follow the same spec.
- **Two-level post-push verification** (hard-won addition dated 2026-10-08): repo descriptions lie, file trees don't. Verification requires ① a file-tree reconciliation (is `mcp/` actually there?), ② a blob-SHA fingerprint diff (`git ls-files -s` against the remote tree; `comm -3` must be empty), and ③ sampling a high-risk file from the remote to confirm sanitization really took effect.
- **Troubleshooting quick reference**: six known failures and their fixes — `src refspec main does not match any` (rename the branch), missing git user.name, non-ASCII paths needing URL encoding, the leading-slash `gh api` endpoint quirk in Git Bash, `shutil.copy2` failing when the target subdirectory doesn't exist yet, and more.

## How It Works / Tech Stack

This skill is a pure methodology playbook with no executable scripts. At runtime the agent follows the manual step by step, driving system tools directly:

- **GitHub CLI (`gh`)**: creating repos, adding SSH keys, listing repos, backfilling descriptions/topics, and calling the API for file-tree reconciliation (goes through `api.github.com`, which is directly reachable).
- **git over SSH**: every push uses the `git@github.com:` form on port 22, routing around the blocked port 443.
- **Regex-based sanitization**: scanning and rewriting is done in Python (the playbook notes that Git Bash's `grep --include` is buggy on this machine — don't use recursive grep).
- **Prerequisites**: a Windows machine, a `~/.workbuddy/skills/` skill directory, an authenticated `gh` CLI, and an SSH key at `~/.ssh/id_ed25519`.

## Installation & Usage

Copy this repository's directory into your WorkBuddy skills folder:

```
~/.workbuddy/skills/github-batch-publish/
```

Or into a project-level `.workbuddy/skills/` directory; Claude Code users should use `~/.claude/skills/`.

**Triggering**: say things like "upload my skills to GitHub", "publish skills to GH in bulk", "sync my skills to GitHub", or "push the companion MCP up too", and the agent will load this playbook and execute it.

**Pipeline overview**:

1. Inventory: `ls ~/.workbuddy/skills/`, `gh repo list` to avoid name collisions, `gh auth status` to confirm login. Skills with a `__skillhub` suffix are marketplace downloads and excluded by default (but confirm with the user).
2. Ask the user four things: scope, public/private, sanitization policy, and how to handle MCPs.
3. Create a staging area at `<project>/gh-publish/`, copy each skill into its own subdirectory, excluding `build`, `dist`, `__pycache__`, `.git`, `node_modules`, `.venv`, `chroma`, `logs`, `models`, plus `*.pyc/.log/.db/.exe/.dll/.zip` and `.env` files.
4. Sanitize, then generate README.md / .gitignore / .env.example.
5. For skills with a companion MCP, place the source under `mcp/` and write an `mcp/README.md`.
6. `gh repo create` the remote → `git init -b main` → commit → add the SSH remote → push. For existing repos, clone first, wipe the working tree (keep `.git`), overwrite with new content, then push — preserving commit history.
7. Two-level verification: file-tree reconciliation + blob-SHA fingerprint diff + high-risk file sampling. The job isn't done until all three pass.

## Project Structure

```
github-batch-publish/
├── SKILL.md       # Skill definition (frontmatter + full playbook; the agent's entry point)
├── README.md      # Chinese documentation
├── README_EN.md   # This file (English documentation)
└── .gitignore     # Standard exclusion rules
```

The repo contains no scripts or references — all knowledge lives inline in SKILL.md / the READMEs, and the agent invokes system commands directly at runtime.

## Notes & Caveats

- **Much of this skill records machine-specific field observations** (port 443 blocked, SSH port 22 open, proxy at `127.0.0.1:7890`). Re-verify the network section before applying it in a different environment.
- **Sanitization is a one-way red line**: once a secret hits GitHub, deleting it later doesn't remove it from history and caches. Scan first, push second — and scan the remote again after pushing.
- **Don't hammer the API during a secondary rate-limit block**: REST and GraphQL share the same pool, so switching endpoints is pointless. Retry at 10-minute intervals.
- **Never modify the user's git or proxy configuration on your own initiative**, and don't take the `gh repo create --source=. --push` shortcut (it pushes over HTTPS internally and will fail).
- **Repo descriptions are not trustworthy during verification** — always go by the remote file tree and blob fingerprints. There was a real incident where a description claimed an "MCP add-on pack included" while `mcp/` had never actually been pushed — unnoticed for nearly 40 days.
- The username `sheen945` in the examples is the author's account — substitute your own GitHub username when using this playbook.

## License

MIT

## Author

sheen945
