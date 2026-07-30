# Runner, Catalog, and Analytics

Use this reference for Runner telemetry and stage control, project pipeline
analytics, and CI/CD Catalog component adoption.

## Export Runner job telemetry with OTLP

GitLab Runner 19.0 adds:

- instrumentation feature negotiation;
- an OTLP export client; and
- the first Runner span, `job_execution`.

Plan the initial collector and trace queries around the `job_execution` span.
Do not assume that the initial instrumentation describes unrelated Runner
stages.

## Configure the Runner prepare-stage timeout

GitLab Runner 19.0 makes the prepare-stage timeout configurable in Runner
configuration. Use it to bound environment preparation independently of the
job's script execution.

## View project CI/CD analytics

A redesigned project CI/CD analytics view is in limited availability on GitLab
Dedicated (since 18.0). The project UI exposes pipeline performance trends and
reliability metrics.

Confirm limited-availability access on the target Dedicated environment before
depending on the view for an operational workflow.

## Inspect CI/CD Catalog component usage

Ultimate customers on GitLab.com, GitLab Self-Managed, and GitLab Dedicated can
open a catalog resource's usage details (since 19.0). The details show:

- the projects that consume each component;
- the component version selected by each project; and
- whether the selected version is current.

Consumers on outdated versions appear first, making the view useful for
prioritizing component upgrades.
