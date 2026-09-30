---
ontology: https://fireforce6.github.io/mission-control/bundle
---

# Assignment 5: Fire Force Analysis

## Conformance

Does the model follow the state-machine rule that Initial states have no incoming transitions and Final states have no outgoing transitions?

### Evidence

```table
---
orderBy: ["state asc"]
---
PREFIX oml: <http://opencaesar.io/oml#>
PREFIX state: <https://www.modelware.io/sierra/state#>

SELECT ?state ?problem
WHERE {
    {
        ?state a state:Initial .
        ?transition a state:Transition ;
            oml:hasTarget ?state .
        BIND("Initial state has an incoming transition" AS ?problem)
    }
    UNION
    {
        ?state a state:Final .
        ?transition a state:Transition ;
            oml:hasSource ?state .
        BIND("Final state has an outgoing transition" AS ?problem)
    }
}
ORDER BY ?state
```
### Interpretation

The query returned no violations. This means the Initial states in the model have no incoming transitions and the Final states have no outgoing transitions. If a violation existed, the table would identify the state and the rule it violated.

## Near Miss

Which transitions are close to being fully specified but are missing a trigger?

### Evidence

```table
---
orderBy: ["transition asc"]
---
PREFIX oml: <http://opencaesar.io/oml#>
PREFIX state: <https://www.modelware.io/sierra/state#>

SELECT ?transition ?source ?target ?problem
WHERE {
    ?transition a state:Transition ;
        oml:hasSource ?source ;
        oml:hasTarget ?target .

    FILTER NOT EXISTS {
        ?transition state:isTriggeredBy ?trigger .
    }

    BIND("Transition has a source and target but no trigger" AS ?problem)
}
ORDER BY ?transition
```
### Interpretation

The query identifies transitions that have both a source and target but no trigger. These transitions are close to being fully specified because the transition path is defined, but the event that causes the transition is missing.

## Orphan

Which capabilities are not connected to any process that describes them?

### Evidence

```table
---
orderBy: ["capability asc"]
---
PREFIX oml: <http://opencaesar.io/oml#>
PREFIX process: <https://www.modelware.io/sierra/process#>
PREFIX mission: <https://www.modelware.io/sierra/mission#>

SELECT ?capability ?problem
WHERE {
    ?capability a mission:Capability .

    FILTER NOT EXISTS {
        ?process process:describes ?capability .
    }

    BIND("Capability is not described by any process" AS ?problem)
}
ORDER BY ?capability
```
### Interpretation

The query identifies capabilities that are not described by any process. These are orphaned capabilities because they exist in the model but do not have a process relationship connecting them to operational behavior.

## Coverage

Which processes describe which capabilities?

### Evidence

```matrix
---
rowColumnLabel: Process / Capability
---
PREFIX oml: <http://opencaesar.io/oml#>
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX process: <https://www.modelware.io/sierra/process#>

SELECT ?row ?column (COALESCE(?n, 0) AS ?value)
WHERE {
    ?row a process:Process .
    ?column a mission:Capability .

    OPTIONAL {
        SELECT ?row ?column (COUNT(*) AS ?n)
        WHERE {
            ?row a process:Process .
            ?row process:describes ?column .
        }
        GROUP BY ?row ?column
    }
}
ORDER BY ?row ?column
```
### Interpretation

The matrix shows which capabilities are covered by each process. A value of 1 indicates that a process describes a capability, while an explicit 0 indicates that the relationship is missing.

## View Graph

How the processes are connected to the capabilities they describe.

### Evidence

```graph
---
layout:
  mode: force
  running: true
  fit: true
  padding: 24
  force:
    repulsion: 3000
    linkDistance: 130
    springStrength: 0.006
    gravity: 0.0012
    damping: 0.90
    maxSpeed: 4
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX process: <https://www.modelware.io/sierra/process#>

CONSTRUCT {
    ?process a process:Process .
    ?capability a mission:Capability .
    ?process process:describes ?capability .
}
WHERE {
    {
        ?process a process:Process .
    }
    UNION
    {
        ?capability a mission:Capability .
    }
    UNION
    {
        ?process a process:Process .
        ?process process:describes ?capability .
    }
}
```
### Interpretation

The graph shows processes and capabilities as nodes and the describes relationship as an edge. Connected capabilities show where a process describes a capability, while capabilities without an incoming process edge remain visible as gaps in the model.

## Scripted Analysis

How much of the capability set is covered by each process?

### Evidence

```python
include('src/method/py/utils.py')

import micropip
await micropip.install(['matplotlib'])

import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt

result = await query("""
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX process: <https://www.modelware.io/sierra/process#>

SELECT ?process ?capability
WHERE {
    ?process a process:Process .
    OPTIONAL {
        ?process process:describes ?capability .
    }
}
ORDER BY ?process ?capability
""")

rows = result["rows"]

processes = sorted({
    r.get("process", "").split("#")[-1]
    for r in rows
    if r.get("process")
})

total_capabilities_result = await query("""
PREFIX mission: <https://www.modelware.io/sierra/mission#>

SELECT (COUNT(?capability) AS ?total)
WHERE {
    ?capability a mission:Capability .
}
""")

total_capabilities = int(total_capabilities_result["rows"][0]["total"])

coverage = {}

for process in processes:
    covered = {
        r.get("capability")
        for r in rows
        if r.get("process", "").split("#")[-1] == process
        and r.get("capability")
    }
    coverage[process] = len(covered)

values = [coverage[p] for p in processes]

fig, ax = plt.subplots(figsize=(6, 4))
ax.bar(processes, values)

ax.set_title("Capabilities Covered per Process")
ax.set_ylabel("Capabilities Covered")
ax.set_xlabel("Process")
ax.set_ylim(0, total_capabilities)

for i, process in enumerate(processes):
    ax.text(
        i,
        coverage[process] + 0.15,
        f"{coverage[process]} / {total_capabilities}",
        ha="center"
    )

plt.tight_layout()
display(image_html(fig))
```

### Interpretation
The scripted analysis queries the process-to-capability relationships, computes the number of capabilities covered by each process, and renders the result as a chart.

## Capability Coverage
```compose
template: https://www.modelware.io/sierra/assignment5-analysis
```