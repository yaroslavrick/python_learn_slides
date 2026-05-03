---
title: GitFlow
---

![Multi-branch flow diagram](/assets/images/topics/git-gitflow.svg)
<!-- .element: class="title-illustration" -->

# Branching models

Git Flow, GitHub Flow, trunk-based — when each pays off.

---

## Why a branching model?

When more than one person commits, you need to agree on:

- Where new work happens (a branch? `main` directly?)
- How releases are cut
- How hotfixes reach production
- Who reviews what, when

A branching model is that agreement, written down.

---

## The three main flavors

| | Git Flow | GitHub Flow | Trunk-based |
| --- | --- | --- | --- |
| Long-lived branches | `main`, `develop` | `main` | `main` |
| Feature branches | yes (off `develop`) | yes (off `main`) | very short or none |
| Release branches | yes | no | no |
| Hotfix branches | yes | no (just a feature branch) | no |
| Best for | scheduled releases, multiple versions | continuous deployment | continuous deployment + feature flags |

---

## GitHub Flow — the simplest

```
main  ─────●─────●─────●─────●─────●─────────►
            \           \         /
             ●─●─●       ●─●─●─●
             feature-x   feature-y
```

1. Branch off `main` (`git switch -c feature-x`)
2. Commit on the branch
3. Open a Pull Request when ready
4. Reviewer comments / requests changes
5. Merge to `main`
6. Deploy `main`

Best for: web apps, libraries, **continuous deployment**, small teams.

---

## Trunk-based development

```
main  ───●─●─●─●─●─●─●─●─●─●─●─●─●─►
              ↑       ↑
         feature flag
         hides unfinished
         work
```

- Everyone commits to `main` (or very short-lived branches that merge daily)
- Unfinished features are hidden behind feature flags
- Every commit could ship
- CI is non-negotiable; tests gate every push

Best for: high-frequency deployment, mature teams, feature-flag infrastructure.

---

## Git Flow — the classic

The original Vincent Driessen model (2010):

```
main      ──●─────────●─────●────────►   (production releases)
             \         \   /
release       ●─●─●─●─●
             /         \
develop  ●──●───●─●─●─●─●────●─●────────►
              \  \         \  \
feature        ●─●         ●─●
                            \
hotfix                      ●─●
```

- `main` — production-ready, tagged for each release
- `develop` — the integration branch
- `feature/*` — branched off `develop`, merged back
- `release/*` — stabilization before going to `main`
- `hotfix/*` — emergency fix off `main`, merged to both `main` and `develop`

---

## When Git Flow fits

- You ship versioned releases (libraries, mobile apps, on-prem software)
- You support multiple released versions in parallel
- You have a QA / release stabilization phase

When it doesn't fit:

- Continuous-deployment SaaS (`develop` becomes vestigial)
- Solo / small teams (overhead with little payoff)
- "Always ship from main" cultures

For most modern web apps, **GitHub Flow** is the right default.

---

## Pull Request workflow

Whatever the model, PRs (or "merge requests" on GitLab) are the unit of review.

A good PR:

- **Small** — a few hundred changed lines, max
- **Focused** — one feature or one fix per PR
- **Tested** — green CI before requesting review
- **Documented in the description** — the *why*, not just the *what*
- **Reviewed by at least one other person** — fresh eyes

Big-bang PRs are review-resistant. Split them.

---

## Naming conventions

Pick a convention and stick to it:

```
feature/billing-export
feature/123-billing-export   # with ticket id
fix/null-pointer-on-login
chore/upgrade-django-5
hotfix/empty-cart-crash
```

Some teams prefer `<author>/<topic>`:

```
alice/billing-export
bob/upgrade-django-5
```

Either works. The point is to be predictable.

---

## Commit message conventions

The Conventional Commits style:

```
feat: add cart export endpoint
fix: handle empty cart on checkout
chore: bump django to 5.1
docs: update README quickstart
refactor: extract OrderService
test: add coverage for refund flow
```

Tools like `commitizen`, `release-please`, and `semantic-release` parse these to auto-generate changelogs and version bumps.

---

## Rebase vs merge

```bash
git switch feature-x
git rebase main             # replay your commits on top of main
```

vs

```bash
git switch feature-x
git merge main              # bring main's commits into the branch
```

- **Rebase** keeps history linear; rewrites commits (force-push needed)
- **Merge** preserves history exactly; can produce noisy "merge commits"

Common rule: **rebase your private branch before opening a PR**, **merge it via a PR** (squash or merge commit). Don't rebase shared branches.

---

## Squash, merge, or rebase-merge?

The three "merge a PR" options on GitHub/GitLab:

- **Merge commit** — keeps every commit on the branch (and adds a merge commit). Most history.
- **Squash and merge** — replays the branch as one commit on `main`. Cleanest `main`, loses branch detail.
- **Rebase and merge** — replays each branch commit linearly on `main`. No merge commit; per-commit detail.

A common policy: **squash for features, merge for releases**. Pick once and document.

---

## What's next

- **CI** — gate merges on green checks
- **Docker** — package what you ship
- **Deployment** — get the merged code to production
