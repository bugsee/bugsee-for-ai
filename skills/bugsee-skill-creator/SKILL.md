---
name: bugsee-skill-creator
description: Author or update a Bugsee SDK setup skill for this plugin. Use when contributing a new platform skill, refreshing an existing one against the latest docs/SDK, or fixing skill drift. Internal contributor tool.
license: MIT
category: internal
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > Skill Creator

# Bugsee Skill Creator

Internal tool for contributors. Produces or refreshes a single SDK skill bundle (`skills/bugsee-<platform>-sdk/SKILL.md`) that meets this repo's conventions and passes validation.

## Step 0 — Study Existing Skills

Read two or three existing SDK skills end to end (e.g. `skills/bugsee-ios-sdk/SKILL.md`, `skills/bugsee-android-sdk/SKILL.md`). Every SDK skill follows the same four-phase shape: **Detect → Install → Initialize → Configure**, then **Verification** and **Documentation Links**.

## Step 1 — Identify the SDK

Pin down the exact platform, package/artifact identifiers, the launch API, and the per-platform app-token story. Note anything platform-specific (manifest entries, build-phase scripts, permissions).

## Step 2 — Research From Official Docs

Fetch the canonical pages and prefer them over memory (APIs change):

```bash
curl -sL https://docs.bugsee.com/sdk/<platform>/installation/
curl -sL https://docs.bugsee.com/sdk/<platform>/configuration/
```

Capture install commands, the `Bugsee.launch()` placement, and the real option names from the configuration page.

## Step 3 — Verify Against SDK Source

Confirm package names, the launch signature, and option keys against the SDK source. The Bugsee SDK repositories live under <https://github.com/bugsee> (several are private — request read access). Never invent API names; verify each one.

## Step 4 — Write the Bundle

Create `skills/bugsee-<platform>-sdk/SKILL.md` with this frontmatter:

```yaml
---
name: bugsee-<platform>-sdk
description: Full Bugsee SDK setup for <Platform>. Use when asked to add Bugsee to <Platform>, install <package>, or set up bug reporting, crash reporting, and video recording for <Platform> applications.
license: MIT
category: sdk-setup
parent: bugsee-sdk-setup
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---
```

Then, as the **first body line**, a breadcrumb:

```
> [All Skills](../../SKILL_TREE.md) > [SDK Setup](../bugsee-sdk-setup/SKILL.md) > <Label>
```

Follow with the H1 title and the four phases. End with a **Verification** step and a **Documentation Links** list pointing at `docs.bugsee.com`.

## Step 5 — Register in the Router

Add the skill to the **SDK Skills** table and the **Quick Lookup** table in `skills/bugsee-sdk-setup/SKILL.md`, and to the relevant table in `AGENTS.md`. Do this *before* validating — the validator requires every skill with a `parent` to be listed in its parent router.

## Step 6 — Validate

```bash
scripts/build-skill-tree.sh            # regenerate SKILL_TREE.md + validate
scripts/build-skill-tree.sh --check    # CI mode: fail if stale or invalid
grep -rn "TODO\|FIXME" skills/bugsee-<platform>-sdk/
```

Also confirm the bundle contains no references to external monitoring vendors — the **Validate Skill Tree** CI job hard-fails on the primary competitor's name; reviewers enforce the broader Bugsee-only rule.

## Conventions

- Single-file skills: one `SKILL.md` per bundle (reference files only if a platform truly needs them).
- Keep `allowed-tools` in every skill (required by Cursor, harmless in Claude Code).
- All wording, URLs, and links are Bugsee-only. Do not reference other vendors' tools or hosts.
