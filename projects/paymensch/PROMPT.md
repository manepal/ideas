# Paymensch — Build Prompt

Copy-paste this entire prompt into Claude Code in a fresh, empty directory to kick off the build.

---

## Setup

First, copy the project files into this directory:

```
mkdir -p docs
cp /Users/maheshn/Work/ojastech/ideas/projects/paymensch/CLAUDE.md CLAUDE.md
cp /Users/maheshn/Work/ojastech/ideas/projects/paymensch/SPEC.md docs/SPEC.md
cp /Users/maheshn/Work/ojastech/ideas/projects/paymensch/BUILD_PLAN.md docs/BUILD_PLAN.md
```

Then initialize graphify for knowledge graph:
```
/graphify
```

---

## Build

Read `CLAUDE.md` first — it defines conventions, non-negotiable rules, and required skills. Then read the spec at `docs/SPEC.md` to understand what we're building. Then read the build plan at `docs/BUILD_PLAN.md` and execute it milestone by milestone.

**Use relevant skills throughout.** Invoke any useful skills you have access to — especially `superpowers:test-driven-development` (EVERY feature starts with failing tests), `security-review` (all payment code), `impeccable` (dashboard UI), `ui-ux-pro-max` (UX decisions), and `graphify` (after every feature commit).

**Pause at milestone boundaries.** The build plan is organized into 6 milestones. After completing each milestone:
1. Announce milestone completion with a summary of what was built
2. Provide exact commands for the developer to test (curl commands, browser URLs, test credentials)
3. Wait for developer sign-off before starting the next milestone

Never build more than the developer can test in one sitting. Each milestone is a complete, working slice of the product.
