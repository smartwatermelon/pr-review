# pr-review

An agent skill for deep-dive review of a teammate's GitHub pull request —
for cases that deserve more than a skim: infra/Terraform changes, IAM or
access changes, or anything where "looks fine" isn't good enough. It traces
claims against the actual repo (usages, history, other consumers of the
changed code) instead of reasoning from the diff alone, cross-checks against
existing PR discussion, triages findings down to the one issue that matters
most, drafts and runs it through
[personify](https://github.com/twistedmelonman/personify), and stages it
as a GitHub pending review. It never posts, submits, or merges without
explicit approval at each checkpoint.

It runs as a plain Markdown skill (`SKILL.md`), so any harness that supports
skill-style instructions can use it.

## Installation

### Claude Code plugin

```
/plugin marketplace add smartwatermelon/pr-review
/plugin install pr-review@pr-review
```

Once installed, invoke it as `/pr-review 123` (or `/pr-review:pr-review 123`
if you have another skill also named `pr-review`).

### Claude Code, project-local

```bash
mkdir -p .claude/skills/pr-review
cp SKILL.md .claude/skills/pr-review/
```

### Claude Code, global

```bash
mkdir -p ~/.claude/skills/pr-review
cp SKILL.md ~/.claude/skills/pr-review/
```

Reload or start a new session after installing.

### Any other harness

The entire runtime artifact is `SKILL.md`. Any agent harness that loads
Markdown-based skills can use it by copying that one file into wherever the
harness expects skill definitions.

## Usage

```
/pr-review 123
review PR 123
take a look at PR 456 in owner/repo
```

Not for your own working diff — use a plain code-review skill for that; this
one is specifically for reviewing someone else's PR.

## What it does

See `SKILL.md` for the full workflow: orient against the repo's own review
contract, classify the kind of change, verify claims against real usage
rather than trusting the diff, adversarially self-check candidate findings,
triage them down to the single highest-severity issue (rarely two), draft
and personify the review, then stage it as a pending GitHub review for a
human to submit.

## Works with personify

pr-review pairs with a sibling skill to get from "review this PR" to a posted
comment that reads like a person wrote it:

```text
pr-review    →    personify
(find it)         (write it)
```

pr-review does the substance: it finds the issue, verifies it, and triages down
to the one finding worth posting. Personify only touches how the draft reads.

- [personify](https://github.com/twistedmelonman/personify) drafts in your own
  register and, given a `VOICE.md`, makes the comment sound like you
  specifically rather than like generically clean prose. Its output is
  send-ready for the surface, so pr-review treats it as final and runs no
  further pass on it.

Phase 5 runs personify automatically when it's installed. If it isn't,
pr-review says so and shows the plain draft. Personify is not load-bearing for
the review's substance.

## License

MIT. See [LICENSE](LICENSE).
