# Bugsee SDK Skill Creator

Agent that authors a brand-new SDK skill bundle for a platform that does not yet have one. Used for manual `workflow_dispatch` runs and as the reference behavior for the `bugsee-skill-creator` skill.

## Procedure

1. **Study** two or three existing skills (`skills/bugsee-ios-sdk/SKILL.md`, `skills/bugsee-android-sdk/SKILL.md`) to learn the four-phase shape: Detect → Install → Initialize → Configure, then Verification and Documentation Links.
2. **Research** the official docs: `https://docs.bugsee.com/sdk/<platform>/installation/` and `.../configuration/`. Capture the real install commands, the `Bugsee.launch()` placement, and the actual option names.
3. **Verify** package identifiers and APIs against the SDK source under `github.com/bugsee`. Never invent API names.
4. **Write** `skills/bugsee-<platform>-sdk/SKILL.md` with the standard frontmatter (`name`, `description`, `license: MIT`, `category: sdk-setup`, `parent: bugsee-sdk-setup`, `disable-model-invocation: true`, `allowed-tools`), a breadcrumb first body line, and the four phases.
5. **Register** the skill in `skills/bugsee-sdk-setup/SKILL.md` (SDK Skills + Quick Lookup tables) and in `AGENTS.md`.
6. **Validate**: `scripts/build-skill-tree.sh` then `scripts/build-skill-tree.sh --check`.

## Rules

- Edit only `skills/` and the router/`AGENTS.md` registration entries.
- **Bugsee-only:** no references to other monitoring vendors, their skill sites, or their MCP endpoints. The **Validate Skill Tree** CI job hard-fails on the primary competitor's name; the broader no-other-vendor rule is reviewer-enforced.
