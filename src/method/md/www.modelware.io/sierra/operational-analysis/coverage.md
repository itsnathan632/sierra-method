---
template:
  id: https://www.modelware.io/sierra/operational-analysis/coverage
  name: "Capability Coverage"
  rank: 0
  expose:
    - kind: compose
  params:
    - id: ontology
      defaultValue: ${context.ontology}
---

# Capability Coverage

```matrix
---
rowColumnLabel: Entity / Capability
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX entity: <https://www.modelware.io/sierra/entity#>

SELECT ?row ?column (COALESCE(?n, 0) AS ?value)
WHERE {
  ?row a entity:Entity .
  ?column a mission:Capability .

  OPTIONAL {
    SELECT ?row ?column (COUNT(*) AS ?n)
    WHERE {
      ?column entity:isAssignedTo ?row .
    }
    GROUP BY ?row ?column
  }
}
ORDER BY ?row ?column
```