# /red-team

A Claude Code command plus dedicated `rt-*` agents for an adversarial hunt of a codebase. The main session coordinates: it spawns hunters in parallel, dedupes their claims, sends each selected claim to a fresh skeptic, and writes a report with verified severity (Critical/High/Medium/Low/Info) and priority (P0-P4).

Key properties:
- **Static only.** Every agent has `tools: Read, Grep, Glob`, `permissionMode: dontAsk`, and `disallowedTools: Write, Edit, NotebookEdit, Bash`. Nothing runs code, tests, or commands.
- **Blind rating.** The skeptic gets one claim block with the proposed severity, priority, rationale, and confidence removed. It rates independently and returns REAL, REFUTED, or UNCERTAIN. The command then assigns CONFIRMED / UPGRADED / DOWNGRADED by comparing priorities, mechanically.
- **Claim gate.** Hunters cap at about 8 claims each. After dedupe, if there are more than 12 claims (`--max-claims N` changes this; `--all` skips the gate), the run pauses once and shows a table of all claims. Reply with IDs, `top N`, a priority floor like `P2+`, or `all`. Each selected claim costs one opus/xhigh skeptic. Under the threshold, everything is verified. Deselected claims are listed in the report as "Not verified (deselected)" with their proposed ratings only. Verification runs in parallel batches of about 10.
- **Models.** `rt-recon` sonnet/medium; hunters opus/high; `rt-skeptic` opus/xhigh; the command runs on opus.

## Layout

```
commands/red-team.md                  orchestrator (runs in the main session)
agents/rt-recon.md                    attack-surface mapper (broad scope only; no rubric)
agents/rt-adversary.md                assumptions, failure modes, feature abuse
agents/rt-security.md                 taint-trail security hunter
agents/rt-correctness.md              wrong results, data loss, outages
agents/rt-skeptic.md                  verifies exactly one claim, rates independently
skills/rt-rubric/SKILL.md             severity scale, priority matrix, claim block (hunters + skeptic)
skills/rt-security-playbook/SKILL.md  OWASP Top 10 (2025) / LLM Top 10 (2026) / API Top 10 / Agentic Top 10 hunting playbook (rt-security)
skills/rt-adversarial/                attack method preloaded by rt-adversary (SKILL.md + references/attack-vectors.md)
```

Reports go to `<target repo>/.context/red-team/YYYY-MM-DD_<slug>.md`. The command warns if that path isn't git-ignored and refuses to write through symlinks.

## Install (symlinks)

Install at user scope (`~/.claude`) only. A project-level install (`<repo>/.claude/agents/rt-*.md`) trips the command's Phase 0 shadow check, which stops the run.

Run from the repo root. Check that each symlink target doesn't already exist first.

```bash
SRC="$PWD"  # run from the repo root
mkdir -p ~/.claude/{commands,agents,skills}
ln -sn "$SRC/commands/red-team.md" ~/.claude/commands/red-team.md
for f in "$SRC"/agents/rt-*.md; do ln -sn "$f" ~/.claude/agents/"$(basename "$f")"; done
for d in rt-rubric rt-security-playbook rt-adversarial; do ln -sn "$SRC/skills/$d" ~/.claude/skills/"$d"; done
```

The symlinks break if the repo is moved; re-create them after moving it.

## Usage

```
/red-team can an unauthenticated user reach another tenant's data? src/api
/red-team find ways the payment retry flow double-charges or loses orders
/red-team the whole repo, focus on auth and file uploads
/red-team --max-claims 20 the checkout service
/red-team --all the auth module   # skip the claim gate, verify every claim
/red-team review the background job queue for races and data loss in services/jobs
```

## Caution

Don't run /red-team inside an untrusted repo whose `.claude/` you've trusted: a project-level agent whose frontmatter `name:` is one of the `rt-*` names would override the installed agent, and the repo's `.claude/` folder is trusted code. As a best-effort check, Phase 0 runs `grep -RlE 'rt-(adversary|correctness|recon|security|skeptic)' --include='*.md' <dir>/.claude/agents` over the session's working directory, the target root, and every `--add-dir` directory (observed in Claude Code 2.1.323: the loader scans the agents folder recursively, matches agents by `name:` rather than filename, follows symlinks (hence `-R`), and project agents override user agents), and stops if anything matches. It cannot catch every way a trusted repo can change agent behavior.

## Compatibility

Tested with Claude Code 2.1.323. Frontmatter such as `effort: xhigh`, `permissionMode: dontAsk`, `disallowedTools`, and `skills:` may need a recent version.

## OWASP

Organized by the OWASP Top 10 (2025), API Security Top 10 (2023), LLM Top 10 (2026) and Agentic Top 10 (2026) categories (owasp.org, CC BY-SA 4.0). Not affiliated with or endorsed by OWASP. The 2026 LLM and Agentic item order and names come from secondary sources; check them against the official PDFs.

