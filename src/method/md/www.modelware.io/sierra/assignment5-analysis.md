---
template:
  id: https://www.modelware.io/sierra/assignment5-analysis
  name: "Assignment 5 Analysis"
  rank: 0
  expose:
    - kind: compose
---

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