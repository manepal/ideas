# Bagh Chal — Build Prompt

Copy-paste this entire prompt into Claude Code in a fresh, empty directory to kick off the build.

---

## Setup

First, copy the spec and build plan into this project's docs folder:

```
mkdir -p docs
cp /Users/maheshn/Work/ojastech/ideas/projects/bagh-chal/SPEC.md docs/SPEC.md
cp /Users/maheshn/Work/ojastech/ideas/projects/bagh-chal/BUILD_PLAN.md docs/BUILD_PLAN.md
```

Then initialize CLAUDE.md with project context:

```
Write a CLAUDE.md for this project. It's a Bagh Chal (Nepali tiger-goat strategy board game) built with React Native (Expo SDK 54, managed workflow). Key dependencies: @shopify/react-native-skia for board rendering, react-native-reanimated for animations, react-native-gesture-handler for touch input, expo-haptics, expo-av for sound, expo-file-system for persistence. Pure local app — no backend, no auth, no network. The spec is at docs/SPEC.md and the build plan is at docs/BUILD_PLAN.md.
```

---

## Build

Read the spec at `docs/SPEC.md` to understand what we're building. Then read the build plan at `docs/BUILD_PLAN.md` and execute it step by step, verifying each step before moving to the next.

**Use relevant skills throughout.** Invoke any useful and relevant skills you have access to — especially for testing, debugging, code review, and UI polish. Don't skip skill checks just because you're following a build plan.

**Pause at phase boundaries.** The build plan is organized into 6 phases. After completing each phase, stop and ask the developer to test what's been built so far before continuing to the next phase. Do NOT build everything in one go — each phase must be tested and signed off before moving on.
