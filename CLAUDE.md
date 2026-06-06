# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

A catalog of weekend-scoped mobile app and game ideas. While some ideas draw from the Nepali context, ideas can span any market or culture. Ideas live in `IDEAS.md`. Selected ideas get elaborated into subfolders with detailed specs, then built by Claude Code agents.

## Project File Anatomy

Each elaborated idea has these files:

| File | Purpose | Audience | Required |
|------|---------|----------|----------|
| `SPEC.md` | What to build — requirements, architecture, data model | Human (design review) | Always |
| `BUILD_PLAN.md` | How to build it — ordered steps, verification gates | Claude Code agent | Always |
| `PROMPT.md` | One-shot paste into Claude Code to launch build | Human (copy-paste) | Always |
| `CLAUDE.md` | Agent conventions, skills, rules, workflow | Claude Code agent | SaaS projects |
| `BRAND.md` | Design system — colors, typography, spacing, icons, tone | Claude Code agent | SaaS projects |

The spec is for understanding. The build plan is for execution. The prompt is so you never retype instructions. For SaaS projects, CLAUDE.md and BRAND.md are copied into the project directory alongside the spec and build plan.

## Idea to Project Workflow

This repository is an idea catalog only. **No project code lives here.**

1. `IDEAS.md` is the master idea catalog — short descriptions only
2. When an idea is selected, create a subfolder: `projects/<idea-slug>/`
3. Write `SPEC.md` with detailed requirements, architecture, and component tree
4. Write `BUILD_PLAN.md` — granular, sequential, with verification gates at every step
5. **Build happens in a separate workspace.** Open Claude Code in a fresh directory and point it at the plan

## How to Feed a Plan to Claude Code

1. Open Claude Code in a fresh, empty directory:
   ```
   mkdir ~/Work/noodle-build && cd ~/Work/noodle-build && claude
   ```
2. Copy the entire contents of `PROMPT.md` and paste it into Claude Code.
3. The agent reads the spec, reads the build plan, and executes step by step.

No typing. Just paste and watch.

## Project Structure Conventions

```
IDEAS.md                        # Master idea catalog (no code)
projects/
  noodle/                       # Idea slug — spec and plan only, no source code
    SPEC.md                     # Detailed requirements
    BUILD_PLAN.md               # Agent-executable build sequence
    PROMPT.md                   # Copy-paste prompt to launch build
```

Actual Expo/Unity projects are created in separate directories outside this repo.

## Build Plan Rules

Build plans must follow these rules so agents can execute them without ambiguity:

- **Strict ordering.** Steps are numbered and sequential. Each step depends only on completed prior steps. No circular dependencies.
- **One thing per step.** A step creates/modifies one file, installs one dependency, or wires one interaction. Never "add gestures, sounds, and haptics" as one step.
- **Exact files listed.** Every step names the file to create or edit with its full path.
- **Verification gate per step.** Every step ends with a concrete, testable check. The agent verifies before moving to the next step. No "trust me" handoffs.
- **No design decisions left open.** The build plan resolves all ambiguity. The agent implements exactly what's written, nothing more.
- **Build order respects real dependency chains.** Rendering → gestures → feedback → state → widget → polish. Never skip ahead.

## Tech Stack Defaults

- **Apps & simple games:** React Native with Expo (managed workflow)
- **Complex 3D games:** Unity
- **2D assets:** Inkscape or Affinity Designer
- **3D assets:** Blender
- **Sound:** generated procedurally or sourced from royalty-free libraries
- **AI coding agent:** Claude Code with relevant MCPs (Expo, file system, etc.)

## Target Platform

**Mobile:** iOS + Android via Expo. All projects should work on both platforms unless an idea specifically requires platform-specific features.

**SaaS:** Web platforms with REST APIs. Defaults below unless an idea specifies otherwise.

## SaaS Tech Stack Defaults

- **API:** Node.js/TypeScript, Fastify, PostgreSQL, Redis
- **Dashboard:** Next.js (App Router), Tailwind CSS, shadcn/ui
- **Job queue / messaging:** BullMQ (Redis-backed)
- **Auth:** API keys for machine clients, JWT + refresh tokens for dashboard
- **Monorepo:** Turborepo with `packages/shared/`, `packages/api/`, `packages/dashboard/`
- **Hosting:** Dockerized, Hetzner or DigitalOcean (Singapore region for low Nepal latency)
- **AI coding agent:** Claude Code with relevant MCPs (GitHub, file system, etc.)
