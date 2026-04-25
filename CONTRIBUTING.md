# Contributing

Welcome to the OpenTelemetry GenAI Semantic Conventions repository!

Before you start — see the OpenTelemetry general
[contributing](https://github.com/open-telemetry/community/blob/main/guides/contributor/README.md)
requirements and recommendations.

## Prerequisites

You need [Docker](https://docs.docker.com/get-docker/) (or a compatible
runtime such as Podman aliased as `docker`). The Makefile pins and runs
Weaver via the `otel/weaver` container image.

## 1. Modify the YAML model

Refer to the
[Semantic Convention YAML Language](https://github.com/open-telemetry/weaver/blob/main/schemas/semconv-syntax.md)
to learn how to make changes to the YAML files.

### Code structure

```
├── docs
│   ├── gen-ai/            # hand-written docs (spans, metrics, events)
│   │   └── mcp.md         # hand-written MCP doc
│   └── registry/          # auto-generated attribute registry pages
├── model
│   ├── manifest.yaml      # dependency on core semantic conventions
│   ├── gen-ai/
│   │   ├── registry.yaml  # attribute definitions
│   │   ├── spans.yaml     # span conventions
│   │   ├── metrics.yaml   # metric conventions
│   │   ├── events.yaml    # event conventions
│   │   └── deprecated/    # deprecated conventions
│   ├── mcp/
│   └── openai/
```

All attributes must be defined in `registry.yaml` files under the matching
namespace folder in `model/`.

### Stability level

Every new group and attribute must declare a `stability:` level. New
proposals should start at `development`. Promotion to `release_candidate`
or `stable` should be a separate PR.

## 2. Regenerate the docs

After updating the YAML, run:

```bash
make generate-docs
```

This regenerates the attribute registry pages under `docs/registry/` and
refreshes the generated tables embedded in the hand-written docs
under `docs/gen-ai/`.

## 3. Validate

Run the full validation suite:

```bash
make check
```

This validates the model against shared policies from
[opentelemetry-weaver-packages](https://github.com/open-telemetry/opentelemetry-weaver-packages).

## 4. Update reference scenarios

Changes under `model/` or `docs/` typically require updating the runnable
reference scenarios under `reference/`. Ideally at least two real
libraries should demonstrate the new convention end-to-end. See
[reference/CONTRIBUTING.md](reference/CONTRIBUTING.md).

## 5. Update the changelog

Add an entry under `Unreleased` in [CHANGELOG.md](CHANGELOG.md) for any
change a consumer of these conventions would need to notice. Editorial
changes (typos, rewording, non-normative clarifications) don't need an
entry.

## Keep PRs small

Small, focused PRs are much easier to review, and therefore much more
likely to land quickly. Even when changes are related, prefer phasing
them across multiple narrow PRs over one large change.

## Opening the PR

The [pull request template](.github/PULL_REQUEST_TEMPLATE.md) asks for
user journey, prior art, and a prototype — reviewers will look for these.

## Driving a PR forward

Design questions and proposals are best discussed with the
[GenAI SIG](https://github.com/open-telemetry/community#sig-genai-instrumentation).
When a PR needs more eyes, post in
[#otel-genai-instrumentation](https://cloud-native.slack.com/archives/C06KR7ARS3X)
on [CNCF Slack](https://slack.cncf.io/) or raise it at the next SIG meeting.

If the review surfaces contentious or difficult points, consider splitting
them into follow-up PRs so the uncontroversial parts can land and each
harder point gets its own focused discussion and review.
