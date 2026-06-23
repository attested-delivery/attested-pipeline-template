---
diataxis_type: reference
title: Documentation index
description: Index of all attested-pipeline-template documentation, organized by Diátaxis type.
---

# attested-pipeline-template — documentation

Documentation for `attested-pipeline-template`, organized by
[Diátaxis](https://diataxis.fr/) type.

---

## How-to guides

Task-oriented. Follow these when you want to accomplish a specific goal.

| Guide | Summary |
|---|---|
| [Instantiate with Copier](how-to/instantiate-with-copier.md) | Stand up a new project from this template and pull future improvements with `copier update` |

---

## Reference

Austere, factual machinery description. Consult these when you need to know
exactly what a workflow does, what predicates it produces, or what commands to
run.

| Reference | Summary |
|---|---|
| [Quality gates and workflows](reference/gates.md) | Every workflow (`ci.yml`, `quality-gates.yml`, `release.yml`, `dast.yml`) — jobs, triggers, predicate types, signing identities, and verification commands |

---

## Explanation

Discursive background. Read these to understand why the template is structured
the way it is.

| Explanation | Summary |
|---|---|
| [The attested pipeline model](explanation/attested-pipeline.md) | The build→attest→verify→publish invariant, the SLSA Build L3 seam, why fail-closed verification matters, and how merge-time and release-time gates differ |

---

For consumer-facing verification instructions see [`SECURITY.md`](../SECURITY.md).
For the org-wide attestation architecture and ecosystem concepts see the
[attested-delivery docs](https://attested-delivery.github.io/docs/).
