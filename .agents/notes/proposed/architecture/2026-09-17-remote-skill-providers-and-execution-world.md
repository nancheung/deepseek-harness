# Agent Note: Remote skill providers and the execution world

Status: proposed

English | [中文](2026-09-17-remote-skill-providers-and-execution-world.zh.md)

## Problem

The skill capability already abstracts where skill bodies come from. `SkillProvider.list()` and `get()` are asynchronous, `locator` is opaque provider state, and `SkillResourceBase` admits `url` and `opaque` alongside `directory`. A provider backed by HTTP, a database, or an object store needs no change to `ctx.skills`, `dsh-tool-skill`, or the agent loop.

What is not abstracted is the relationship between two independent axes. Where a skill is stored is a provider concern; where the calling agent executes its `read`, `bash`, and `edit` tools is a `ctx.fs` and `ctx.shell` concern. `dsh-fs-local`, `dsh-fs-e2b`, and `dsh-fs-sandbox` already make the second axis swappable, and `dsh-skill-filesystem` already probes `.git` through `ctx.fs` so a sandboxed workspace is discovered in its own world. Both axes are solved in isolation.

The gap is at their intersection. `dsh-skill-filesystem` emits `resourceBase: { kind: 'directory', path: locator.directory }`, a bare path string that the model resolves relative references against and then reads with the `read` tool. That works only while the world holding the skill and the world `ctx.fs` points at are the same world. A team registry serving skills over HTTP to an agent whose `ctx.fs` is local, or a local skill directory driving an agent whose `ctx.fs` is an E2B sandbox, both hand the model a path that resolves to nothing. Progressive disclosure below `SKILL.md` silently stops working: the catalog still lists the skill, the `skill` tool still returns its body, and only the third level — `references/`, `scripts/`, `assets/` — fails, at the point where the model has already committed to the task.

Resource classes also differ in what they need, which a single `resourceBase` cannot express. A reference document needs its bytes in the model's context and does not care which world produced them. A script needs to exist in the world `ctx.shell` runs in, because the model was told to execute it. Treating both as one problem forces a choice between materializing everything, which pays transfer cost for documents that may never be read, and serving everything remotely, which leaves scripts unrunnable.

A remote provider additionally sits on a hot path. `dsh-tool-skill` calls `ctx.skills.snapshot()` at every eligible `agent/pre-step`, so a provider that reaches the network on each `list()` adds a round trip before every model request, not once per session. Providers are queried sequentially, so one slow provider delays every provider registered after it.

## Proposal

Keep the registry and the consumer unchanged. Add a provider package and one model-facing tool, and give the two resource classes different transports.

**Reference documents and assets are served by the provider.** A `skill_resource({ name, path })` tool reads one resource through the same provider that owns the skill body, returning its text as a tool result. The provider enforces path containment, authentication, and caching internally, so no filesystem world is involved and a `../` escape is rejected at the source rather than by the filesystem policy of whichever world happens to be mounted.

**Scripts are materialized into the execution world.** A provider that ships executable resources stages them through `ctx.fs` into a workspace-local directory on first use, and reports that directory's path in its resource guidance. Only scripts are staged, so an unread reference costs nothing.

**`resourceBase` becomes `opaque` for such providers.** The `opaque` kind promises no addressing semantics, only that the provider explains how to obtain resources. Its description names both transports:

```
Resources for this skill are served by provider "team-registry".
Read reference files with the `skill_resource` tool: skill_resource({ name: "…", path: "references/…" }).
Executable scripts are staged into the workspace at /workspace/.skill-cache/<name>/ on first use.
Load referenced resources only as needed.
```

**Remote discovery answers from memory.** A remote provider returns a cached candidate list from `list()` and refreshes in the background, calling the registration-scoped `invalidate()` from [skill catalog hot refresh](../../implemented/feature/2026-07-27-skill-catalog-hot-refresh.md) when the refresh observes a change. `list()` must not await the network on the pre-step path. A refresh failure returns `{ candidates: lastGood, complete: false }`, which the registry already handles: candidates stay loadable, the aggregate snapshot is uncacheable, and `dsh-tool-skill` publishes no catalog message and retains the model's last-good view.

### Two independent axes

| Axis | Question | Seam | Implementations |
|---|---|---|---|
| Skill storage | where `SKILL.md` is read from | `SkillProvider` | `dsh-skill-filesystem`, `dsh-skill-badge` |
| Execution world | where the model's file and shell tools act | `ctx.fs`, `ctx.shell` | `dsh-fs-local`, `dsh-fs-e2b`, `dsh-fs-sandbox` |

All four combinations are real deployments: local storage with local execution is the shipped CLI; local storage with sandboxed execution is local authoring against E2B; remote storage with local execution is a team registry; remote storage with remote execution is a hosted agent.

### Resource classes and their target worlds

| Class | Needs | Transport | Cost when unused |
|---|---|---|---|
| `references/*.md` | bytes in model context | `skill_resource` tool | none |
| `assets/*` | bytes in model context | `skill_resource` tool | none |
| `scripts/*` | executable in the `ctx.shell` world | staged through `ctx.fs` | staging on first use |

### Team-shared skill registry

The remote-storage, local-execution combination is the first consumer, and needs no core change.

A `dsh-skill-http` provider package registers synchronously during `apply()` and performs authentication inside its awaited `list()`, taking credentials from `ctx.credentials` rather than reading the environment. Its `locator` carries `{ id, version }` so `get()` loads an exact revision and a stale selection is detectable. It registers at rank `150`: below `project-dsh` (100) so a project's own skill always wins, above `runtime` (250) and the user roots so the shared library overrides personal copies. Because rank decides duplicates only within one layer, a registry mounted in one agent preset's composition stays invisible to other presets.

Publishing is a repository of skill bundles plus a server that serves a catalog endpoint and a body endpoint. The catalog response carries `name`, `description`, `version`, and the invocation policy for every skill; the body endpoint returns one skill's Markdown by id and version. `skill_resource` reads from a third endpoint scoped to one skill id.

Freshness comes from the provider, not the registry, which has no TTL and cannot infer that a remote source changed. In order of decreasing timeliness: a webhook or server-sent event stream that calls `invalidate()` on publication; an ETag poll on the catalog endpoint; a fixed interval refresh. The poll timer is cleared from `control.signal`, and `invalidate()` is a no-op once that exact registration is disposed, so a late response from a replaced registration cannot dirty its successor.

## Alternatives considered

**Two-world `resourceBase`.** Extend `SkillResourceBase` with a world identifier and teach the `read` tool to route by it. This pushes multi-world addressing into the filesystem seam and every file tool, to serve one consumer. `ctx.fs` deliberately presents one world per agent; a skill subsystem is not the right reason to break that.

**Serve every resource as a URL.** Set `resourceBase: { kind: 'url' }` and let the model fetch resources with the web tool. This adds no mechanism, but the web tool carries no registry credentials, so private libraries are excluded, and a fetched script still cannot be executed in the agent's world. Viable for a public read-only library, insufficient as the general answer.

**Materialize the whole skill bundle.** Stage every resource into the execution world and keep `kind: 'directory'`. The model needs no new tool, which is the main attraction. It pays transfer and storage for references that are usually never read, needs eviction and per-session isolation, and scales badly for asset-heavy skills. Staging is kept for scripts alone, where nothing else works.

**Let the registry cache remote catalogs.** Move retry and freshness into `ctx.skills` so every remote provider inherits them. The registry has no way to know what invalidates an arbitrary backend, which is why `invalidate()` is provider-driven; a registry-level TTL would guess wrong for both a webhook-backed registry and a slow poll. The existing `complete: false` observation already carries transient failure without a cache.

**Prefetch resources alongside the body.** Return `references/` content inside the `skill` tool result. This collapses the third disclosure level into the second, which is exactly what the tool-result pruner then truncates: a body plus references over `thresholdChars` is cut to a head and tail with the middle dropped silently.

## Acceptance criteria

- A provider serving skills over HTTP contributes to the catalog, loads bodies, and serves reference documents with no change to `ctx.skills`, `dsh-tool-skill`, `dsh-fs`, or `agent-loop`.
- `list()` on a warm remote provider completes without a network round trip; a snapshot test pins that a pre-step adds no request.
- A registry failure leaves the previously published catalog visible to the model and publishes no replacement catalog message.
- `skill_resource` rejects a path escaping its skill's resource root, and rejects a skill whose `invocation.modelInvocable` is false, with distinct errors.
- Scripts staged into an E2B execution world are executable by `bash` in the same session; the staging directory is removed on session end.
- A skill published at two versions loads the version named by the candidate `locator`, and a body whose name no longer matches its candidate invalidates that provider.

## Risks

- **A second resource tool costs schema tokens in every request where it is visible.** The catalog already pays per-skill description tokens; this adds a fixed schema cost for sessions that mount a remote provider. Mounting it only where a remote provider is composed keeps local-only deployments unaffected.
- **The model must learn a second read path.** Resource guidance is prose, and a model may still attempt `read` on a staged-only path. The guidance names the tool explicitly, but nothing enforces the choice; a failed `read` is a recoverable error, not a silent wrong answer.
- **Staging writes into the user's workspace.** A `.skill-cache` directory appears in the execution world and can be committed by accident. It belongs under a harness-owned path, and removal on session end is best-effort: a killed process leaves it behind.
- **Sequential provider queries make a slow remote provider everyone's problem.** Background refresh keeps it off the pre-step path, but a provider that ignores this and awaits the network delays every provider registered after it. This is a property of the existing registry, surfaced rather than introduced by remote providers.
- **Version skew between a cached catalog and a loaded body.** A background refresh can replace the catalog between a model reading a summary and calling `skill`. The registry already rejects a definition whose name differs from its candidate; a version change under the same name is accepted, so the model may act on a body newer than the description that routed it there.
