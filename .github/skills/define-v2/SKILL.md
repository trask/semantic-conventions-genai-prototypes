---
name: define-v2
description: 'Use when writing or editing weaver v2 definition schema (file_format: definition/2*) for spans, metrics, events, attribute groups, or signal refinements in any semantic-conventions registry. Enforces minimal attribute groups, internal-by-default visibility, flat structure, and refinement of generic signals over re-declaration. Always validates by running the repo''s package target (typically `make package`).'
argument-hint: 'Describe the new or modified span/metric/event/entity and which generic signal it specializes (if any).'
---

# Define v2

Use this skill when adding or changing a weaver v2 definition file
(`file_format: definition/2` or `file_format: definition/2.*`) — spans,
metrics, events, entities, attribute groups, or signal refinements — in
any semantic-conventions registry.

## Goal

Produce a v2 definition that is minimal, flat, and resolvable by weaver
without errors. Reuse existing signals and attribute groups, and express
"signal X is an implementation of generic signal Y" as an explicit
refinement rather than a copy of Y's attributes.

This skill is not for deciding *whether* a convention should be added or
arguing about attribute names, requirement levels, or stability — those
are upstream design questions. It is also not a generator: bring the
names, descriptions, and examples; the skill shapes the structure around
them.

## Schema Reference

This skill encodes opinions, not the v2 grammar. For top-level sections,
required keys per signal, valid `kind` / `instrument` / `stability` /
`requirement_level` values, and override rules — read the doc directly:

- [Lifecycle and overview](https://raw.githubusercontent.com/open-telemetry/weaver/main/schemas/semconv-schemas.md)
- [Authoring syntax reference](https://raw.githubusercontent.com/open-telemetry/weaver/main/schemas/semconv-syntax.v2.md)

If anything here contradicts the doc, the doc wins.

## Known Weaver Limitations

Pinned-weaver behaviors that change what shape is achievable. If you
find yourself fighting one of these, link the issue in a TODO and use
the workaround:

- **[weaver#1411](https://github.com/open-telemetry/weaver/issues/1411)** —
  `ref_group` is silently ignored inside `*_refinements`. Inline the
  refs on each refinement, even when several share the same delta.
- **[weaver#1407](https://github.com/open-telemetry/weaver/pull/1407)** —
  per-attribute `note` overrides on a refinement's `ref:` don't apply
  for some attributes; workaround is a single-attribute internal group
  used via `ref_group` from each refinement that needs the override.
  This is a Rule 1 exception — TODO-link the PR and remove once it lands.

Add to this list when a workaround surfaces; prune when issues close.

## Authoring Rules

### 1. Minimal attribute groups

Only create an `attribute_groups` entry when the *same exact set* of
attributes is reused by two or more signals. A group used by exactly
one signal is dead weight — inline the attributes on the signal
directly. Avoid creating an attribute group with only one attribute.

### 2. Internal by default

Every authoring `attribute_groups` entry MUST have `visibility: internal`
unless the group is genuinely public. Public groups carry stricter
required-key obligations (see the syntax doc); if you don't need those,
you don't want a public group.

Sharper test for public: would you naturally write a `brief` and pick a
`stability` for the group *as a thing in its own right* (something
referenced outside the snippet that embeds it)? If it exists only to
compose attributes for a couple of signals, keep it internal. Being
rendered in a `<!-- semconv <group_id> -->` snippet is not the test —
visibility does not gate rendering.

### 3. Flat structure

The [upstream syntax doc](https://github.com/open-telemetry/weaver/blob/main/schemas/semconv-syntax.v2.md#attribute-group-reference)
says it is *NOT RECOMMENDED* to use `ref_group` on another attribute
group, citing readability. This registry departs deliberately: many
GenAI signals share large overlapping subsets (common, address_and_port,
usage, content, error), and composing them via `ref_group` is the only
way to keep per-signal yaml short without one mega-group.

Keep the chain shallow:

- **One level** (`base group → signal`) is the default.
- **Two levels** (`base → composite → signal`) only when 3+ signals
  share the same composite shape and inlining would duplicate a
  multi-line block. Justify in review; do not add speculatively.
- **Three+ levels** — almost never. Treat as a refactor signal.

Within an `attributes:` list, list every `ref_group` entry before any
`ref` entry — the group's own attributes appear after the inherited
ones, not interleaved.

```yaml
attribute_groups:
  - id: attributes.<domain>.common         # level 1
    visibility: internal
    attributes:
      - ref: <attr.a>

  - id: attributes.<domain>.client         # level 2
    visibility: internal
    attributes:
      - ref_group: attributes.<domain>.common
      - ref: <attr.b>
```

### 4. Refinement over redefinition

If the new signal is an implementation-specific variant of a generic
signal that already exists in this registry, declare it in the matching
`*_refinements` section and add only the delta.

Refinements inherit the parent's attributes — do **not** re-list them.
List an inherited attribute only to override its presentation (note,
brief, requirement_level, examples, sampling_relevant — see the syntax
doc for which keys are overridable per signal type).

Tempted to declare the variant in `spans:` and `ref_group:` the parent's
attribute group(s)? Stop and use a refinement. Same applies to implicit
signal groups (`ref_group: span.<parent>`) — see Rule 6.

### 5. Override on the signal, not on the attribute

For a different `requirement_level`, `note`, `brief`, etc. on an
inherited attribute, override at the reference site — do not edit the
underlying attribute definition. Don't duplicate the inherited
`brief` / `note` verbatim; leave them out so the inherited values show
through. Use `note: ""` only when you genuinely want to suppress an
inherited note.

### 6. Implicit signal groups

Weaver exposes each signal's full attribute set — every override and
every inlined ref — as an implicit group named `span.<type>`,
`metric.<name>`, or `event.<name>`. This is undocumented in the upstream
syntax doc; treat as undocumented behavior and verify with `make package`
whenever you rely on it.

Legitimate use: **cross-signal mirroring**, e.g.
`gen_ai.client.inference.operation.details` (an event) uses
`ref_group: span.gen_ai.inference.client` so it picks up every override
the span declaration carries. Re-stating each `ref_group` and override
would drift the moment the span changes.

Do **not** use this inside a new signal under
`spans:` / `metrics:` / `events:` to clone a generic signal — that's
Rule 4's anti-pattern in different syntax. Use `*_refinements` instead.
Symptoms that you are about to drift:

- A `spans:` entry whose `attributes:` is `ref_group: span.<other_type>`
  followed by a small delta.
- A new internal `attributes.<provider>.<thing>` group whose only purpose
  is to hold that delta.

## Refinement Decision Tree

For every new signal, walk this before writing yaml:

1. **Implementation-specific variant of an existing signal?**
   → declare it in `*_refinements` with `ref:` to the parent. Delta only.
   Per-attribute overrides for any inherited attribute whose framing
   changes. Done.
2. **Shares its attribute set with a signal already in the registry?**
   → reuse the existing group via `ref_group`. Delta inline on the signal.
3. **Shares its attribute set with another signal in this same change?**
   → one shared internal group. One level deep. No nesting.
4. **None of the above?** → inline the attributes on the signal. No group.

## Procedure

1. Identify what the change touches: spans, metrics, events, entities,
   attribute groups, refinements, or several.
2. **Capture a pre-change baseline.** Run `make package` and the
   committed-snapshot target so the resolved-schema and snapshot
   artifacts reflect HEAD; you'll diff against this after editing.
   Skip if the user mentioned a baseline or the working tree is in sync.
3. If syntax for any touched section isn't obvious from surrounding
   files, fetch the v2 syntax reference (see Schema Reference).
4. Walk the Refinement Decision Tree for each new signal.
5. Write the yaml; satisfy the rules.
6. Run validation (next section). Fix every error.
7. Re-read the diff for: groups used exactly once, `ref_group` chains
   deeper than two levels, `visibility: public` that should be internal,
   re-listed parent attributes on refinements, `ref_group: span.<...>`
   inside a new signal declaration. These are the common drifts.
8. Regenerate every committed artifact (see Output Format) and compare
   against the step-2 baseline.

## Validation

The non-negotiable gate: the resolved registry builds cleanly with zero
errors. Prefer the repo's Makefile wrapper:

```
make package
```

If a `check` / `check-policies` target also exists, run it — it catches
id collisions and naming-convention violations that `package` alone
does not.

Fall back to direct weaver invocation only when there is no wrapper:
`weaver registry package -r <registry-root> --v2`, or the pinned
container image. Note: `weaver registry resolve` is deprecated — use
`package`.

## Output Format

Do not write a prose summary of what changed. Run the repo's wrappers
and link the artifacts. Run in order — snapshot and docs both consume
the resolved model that `package` produces:

1. **`make package`** — builds the resolved schema and runs validation.
   Output is wrapper-specific (e.g. `./resolved`) and typically
   gitignored. Link the resolved schema file.
2. **Committed-snapshot target** — `make schema-snapshot` in this
   registry (`./schema-snapshot/registry.yaml`); `make generate` in
   other registries (typically `./generated/registry.yaml`).
   Single-file reviewable schema diff. CI's "snapshot in sync" check
   runs this.
3. **Docs regeneration** — here, `make generate-registry` rebuilds the
   per-namespace pages under `docs/registry/`, `make generate-docs`
   refreshes the `<!-- semconv ... -->` snippet tables in hand-authored
   docs. Other registries typically combine these under one target.

After running, link every file the regeneration changed: the snapshot,
regenerated `docs/registry/` pages, and any snippet refreshes elsewhere
under `docs/`.
