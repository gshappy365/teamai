# Triage Labels

Triage uses two category roles and five state roles. This document maps each canonical role to the label string used in this repository's GitHub issue tracker.

## Category labels

| Role | Label in this repo | Meaning |
| --- | --- | --- |
| `bug` | `bug` | Something is broken or not working. |
| `enhancement` | `enhancement` | A new feature or improvement. |

## State labels

| Role | Label in this repo | Meaning |
| --- | --- | --- |
| `needs-triage` | `needs-triage` | A maintainer needs to evaluate this issue. |
| `needs-info` | `needs-info` | Waiting on the reporter to provide more information. |
| `ready-for-agent` | `ready-for-agent` | Fully specified and ready for an AFK agent. |
| `ready-for-human` | `ready-for-human` | Requires human implementation. |
| `wontfix` | `wontfix` | Will not be actioned. |

Every triaged issue should have exactly one category label and exactly one state label. When a skill mentions a role, use the corresponding label string from the tables above. If the tracker's label vocabulary changes, update the mapped label column to match it.
