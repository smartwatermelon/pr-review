---
name: pr-review
description: |
  Deep-dive review of a teammate's GitHub pull request, for cases that deserve
  more than a skim: infra/Terraform changes, IAM or access changes, or
  anything where "looks fine" isn't good enough. Traces claims against the
  actual repo (usages, history, other consumers of the same code) instead of
  reasoning from the diff alone, cross-checks against existing PR discussion
  so comments don't duplicate what's already been said, triages findings down
  to the single most important issue (rarely two), drafts and personifies
  that, and stages it as a GitHub pending review for explicit approval before
  anything gets posted. Never posts, submits, or merges without the user's
  go-ahead at each checkpoint. Use for "/pr-review 123", "review PR 123",
  "take a look at PR 123", or similar requests to review someone else's PR
  (not your own working diff — for that, use /code-review).
metadata:
  version: "1.1.0"
---

# PR Review: deep-dive on someone else's PR

This is a slower, more skeptical alternative to a quick `gh pr view` skim. It
exists because surface-level review misses the interesting bugs: the ones
where the diff looks reasonable in isolation but is wrong (or already being
discussed, or actually fine) once you trace how the changed code is really
used. Budget for several tool calls per finding — that cost is the point.

The person receiving this review is (usually) a human, not another agent, and
a wall of findings is a burden, not a service. Do the full investigation, but
narrow what actually gets posted to the PR down to the one issue that matters
most — see Phase 5. Investigation depth doesn't change; what changes is how
much of it becomes a comment on someone else's PR.

## Terminology and staying current

- **"The user"** below always means whoever is running the agent that invoked
  this skill — not Andrew Rich, this skill's author and maintainer. Apply the
  review criteria, staging workflow, and approval gates for whoever that is;
  don't assume Andrew's personal preferences or repos unless the invoking
  user's own project docs (Phase 1) say so.
- This skill is distributed from
  [smartwatermelon/pr-review](https://github.com/smartwatermelon/pr-review).
  If it's been a while since you installed or last checked it, it's worth
  confirming you're running the current version before relying on it for a
  security-sensitive review — compare your installed copy's frontmatter
  `metadata.version` against the repo's `SKILL.md`, or just run
  `/plugin marketplace update` if you installed it as a Claude Code plugin.
  Not required, just cheap insurance against acting on stale instructions.

## Inputs

Parse the invocation for a PR number and optionally an owner/repo. If no
repo is given, resolve it from the current directory's git remote
(`git remote -v`, parse `owner/repo` out of the SSH or HTTPS URL). If `gh`
can't resolve the repo ("Could not resolve to a Repository"), check
`gh auth status` for multiple logged-in accounts — the active account may
not have access — and `gh auth switch --user <other-account>` before retrying.

## Phase 1: Orient

1. If working in a local clone of the repo, `git pull` on the default branch
   first so local file reads reflect current `main` — you'll be cross-referencing
   the PR against the rest of the repo, and stale local state produces wrong
   conclusions.
2. Read the repo's own review contract if one exists: `CLAUDE.md`, `AGENTS.md`,
   `CONTRIBUTING.md`, or a nearer-scoped `AGENTS.md`/`README.md` in the touched
   directories. Repos with strict domain rules (e.g. a Terraform style guide,
   a "security-critical change" checklist, a list of high-risk paths) define
   what actually matters for that repo — apply those criteria, don't invent
   generic ones.
3. Fetch PR metadata and the full diff:
   - `gh pr view <N> -R <owner/repo> --json title,author,body,files,additions,deletions,url,state,mergeable,reviewDecision,commits`
   - `gh pr diff <N> -R <owner/repo>`
4. Fetch existing conversation so you don't duplicate it:
   - `gh api repos/<owner/repo>/pulls/<N>/comments --jq '.[] | {id,user:.user.login,created_at,path,in_reply_to_id,body}'`
   - `gh api repos/<owner/repo>/issues/<N>/comments --jq '.[] | {user:.user.login,created_at,body}'`
   - Reconstruct threads via `in_reply_to_id`. Note anything unresolved.
   - Also check for CI/plan output posted as PR comments (e.g. an Atlantis
     plan) — that's often better evidence than reasoning about the diff by
     eye, since it shows the actual resource-level effect.

## Phase 2: Classify

For each changed file/hunk, decide what kind of change it is before judging
it: narrow access grant, broad/shared surface (module, provider, boundary,
CI config), pure refactor, docs, etc. If the repo's docs define
"security-critical" or "high-risk path" categories, explicitly check the
diff against them. This classification determines how much Phase 3 digging
is warranted — a docs typo doesn't need it; a permission boundary change does.

## Phase 3: Verify, don't assume

This is the phase that's easy to skip and most valuable to do. For every
claim you're tempted to make about the PR's effect, ask "how would I know
that's true?" and go check, rather than inferring it from the diff text
alone:

- **Trace real usage.** `grep` for other consumers of the changed
  module/resource/policy across the whole repo, not just the touched files.
  A change to a shared primitive (module, boundary, policy) may be consumed
  by multiple roots/services with different blast radii.
- **Check for parallel/duplicated definitions.** Copy-pasted config (same
  SIDs, same comments, same structure in a different file) is a common
  source of silent drift — one copy gets updated, the sibling doesn't. Find
  siblings with `grep` for distinctive strings (a SID name, a resource ARN
  pattern), not just the filename.
- **Follow the actual wiring, not the apparent one.** A boundary or policy
  is only as consequential as what actually references it. Check how names
  are resolved (hardcoded vs. templated vs. computed), and trace which
  concrete environment/cluster/account each copy actually applies to before
  concluding two things are "the same."
- **Check history for precedent.** `git log --oneline -- <file>` and
  `gh pr list --search "<topic>"` reveal whether a pattern (e.g. "only update
  one of two copies") is new or an existing habit in the repo, which changes
  whether it's worth raising and how you phrase it.
- **Re-read the existing PR conversation for overlap.** If a reviewer has
  already raised a concern, don't restate it — note that you saw it and defer,
  and only add something if you have a genuinely different angle.

Write down each candidate finding with the concrete evidence for it
(file:line, the grep/log output that supports it) before moving to drafting.
A finding you can't point to hard evidence for gets cut or turned into a
question instead of an assertion.

## Phase 4: Adversarial self-check

Before drafting anything, go back over the candidate findings and actively
look for the innocent explanation, per the standing instruction to distinguish
real shortcomings from "something that might be intended on their side or
misinterpreted on ours":

- Is there a reason this might be intentional (a migration in progress, a
  deprecated module on its way out, an accepted tradeoff discussed
  elsewhere)?
- Would asking a direct question ("was X in scope here, or intentionally
  deferred?") serve better than an assertion ("X is missing")?
- For anything substantial (security-critical, or a finding you'd stake
  real credibility on), consider spawning a fresh-context fork to
  independently re-derive the finding from the code without seeing your
  framing — a second pass with no anchoring bias catches overstated claims.
  Don't skip this for convenience on anything you're about to tell the user
  is a real problem.

If the author or another reviewer responds to a posted finding with a
reasonable explanation, verify it against the repo rather than accepting it
at face value — check that referenced modules/paths actually exist in the
state they describe. Report back honestly if you can't fully verify (e.g.
the replacement doesn't exist yet), without treating that as grounds to
re-litigate a settled, reasonable-sounding answer.

## Phase 5: Triage, draft, personify, approve

1. **Triage down to one issue before drafting.** Rank surviving candidate
   findings by severity: correctness > security/access > design > nit. A
   well-scoped PR is usually telling one story even when it shows up in
   several places, so compact same-cause findings into a single item with
   multiple touchpoints ("methods X, Y, and Z all read the stale config
   key") rather than posting them separately.
   - Post only the single highest-severity compacted item. A second item is
     allowed only if it sits in the same top severity tier as the first,
     addresses a genuinely different concern, and shares no root cause with
     it — don't include a second finding just because it's also true.
   - If the candidate findings can't be compacted into one coherent item
     because the PR itself spans too many unrelated concerns, that's a
     scope problem with the PR, not a reason to post more comments. Note it
     to the invoker (next step) instead of routing it onto the PR author as
     extra review comments.
   - If nothing clears the bar for a comment, don't invent a nit to have
     something to say. Plan to stage an Approve-leaning review with a
     brief, specific positive callout instead (Phase 6 still applies: it
     stays a pending, unsubmitted review either way).
2. If any candidate findings were set aside during triage, tell the invoker
   what they were and why (lower severity than the headline item, same root
   cause, PR scope too broad to compact, etc.) before showing the draft —
   the investigation wasn't wasted, it just isn't all becoming PR comments.
   Skip this step entirely when nothing was set aside.
3. Draft the review as a short overall summary plus the triaged finding(s),
   anchored to a file/line, phrased as a question where genuine uncertainty
   remains and as a direct statement only where you have verified evidence.
   If everything was resolved to "nothing to flag," draft a short summary
   recommending Approve plus the positive callout instead.
4. Keep it short. A GitHub review is not the place for discussion, that
   happens in Slack or in person. The finding is either something that
   needs attention (a change request or a genuine question blocking
   approval) or a call-out of particularly clever/notable work, not a
   walkthrough of the code, the behavior, or the reasoning behind it. State
   the conclusion and the one-line reason; don't narrate the investigation
   that got you there, and don't restate the mechanism/logic you traced in
   Phase 3 unless the reader needs it to understand what to change. If the
   finding runs longer than 3-4 sentences, it's probably prose that belongs
   in Slack, not in the review, cut it down to the actionable core.
5. Run the draft through the `personify` skill, if it's installed. If it
   isn't, say so and show the plain draft instead of failing the whole
   review — personify improves the prose, it isn't load-bearing for the
   review's substance.
6. Show the human the draft (personified or plain) and wait for explicit
   approval. Accept edits and re-run as needed. Do not treat silence or a
   tangential reply as approval.

## Phase 6: Stage, don't publish

Never call `gh pr review`/`gh pr comment` directly and never set `event` on
the review (that would submit it immediately). Stage a **pending** review so
it stays private until the human submits it:

```
gh api --method POST /repos/<owner>/<repo>/pulls/<N>/reviews \
  --input review.json
```

where `review.json` has `body` (the summary) and `comments[]` of
`{path, line, side: "RIGHT", body}`, anchored to added lines on the PR's
current head SHA, and **no `event` field**. When Phase 5 concluded "nothing
to flag," `comments[]` may be empty or hold just the positive callout — the
review still stages as pending, not as a submitted approval; the human
chooses and submits the actual verdict.

To edit a pending comment: the `PATCH /pulls/comments/{id}` endpoint 404s on
unpublished comments. Delete the whole pending review
(`DELETE /pulls/<N>/reviews/<review_id>`) and recreate it instead of trying
to patch in place.

`review.json` is written to whatever repo you're reviewing (it may contain
excerpts of that repo's code), not to this skill's own directory — delete it
right after the API call, whether it succeeded or failed, so it doesn't
linger and get swept into an unrelated commit there. On a retry, regenerate
it fresh rather than reusing a copy that may reference a stale head SHA.

Tell the human the review is staged and pending, and stop. Submitting the
review, replying to follow-up comments, resolving threads, and merging are
all separate, explicitly-authorized actions — do not chain into them.

## Non-goals

- This skill does not review your own uncommitted/local diff — use
  `/code-review` for that.
- This skill does not merge, approve-and-merge, or push fixup commits to
  someone else's branch. If the review surfaces something you could fix
  yourself, ask before touching their branch.
- This skill does not run `terraform plan/apply`, cloud CLIs, or other live
  validation unless the repo's own docs say agents may do so locally — rely
  on static tracing and posted CI/plan output instead.
