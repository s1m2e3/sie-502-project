---
template:
  id: http://www.example.com/method/requirement-trace
  name: "Requirement Trace"
  rank: 5
  expose:
    - kind: compose
  params:
    - id: requirement
      type: iri
      required: true
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
### Trace of [[${requirement}]]

Who raised it, how it is verified, and whether it counts as a safety requirement (raised by a
Regulator).

```table
PREFIX base:        <http://www.example.com/method/base#>
PREFIX requirement: <http://www.example.com/method/requirement#>

SELECT ?description ?statement
       (GROUP_CONCAT(DISTINCT STRAFTER(STR(?stakeholder), "#"); separator=", ") AS ?raisedBy)
       (GROUP_CONCAT(DISTINCT CONCAT(STRAFTER(STR(?strategy), "#"), " (", STR(?method), ")"); separator=", ") AS ?verifiedBy)
       (IF(COUNT(DISTINCT ?regulator) > 0, "yes", "no") AS ?safety)
WHERE {
  OPTIONAL { <${requirement}> base:description ?description }
  OPTIONAL { <${requirement}> base:expression ?statement }
  OPTIONAL { <${requirement}> requirement:isRaisedBy ?stakeholder }
  OPTIONAL {
    <${requirement}> requirement:isRaisedBy ?regulator .
    ?regulator a requirement:Regulator .
  }
  OPTIONAL {
    <${requirement}> requirement:isVerifiedBy ?strategy .
    ?strategy requirement:hasVerificationMethod ?method .
  }
}
GROUP BY ?description ?statement
```

What it bounds: every leaf of its condition tree, with the output, the bound and the inputs the
leaf mentions.

```table
PREFIX expression:  <http://www.example.com/method/expression#>
PREFIX requirement: <http://www.example.com/method/requirement#>

SELECT ?leaf ?output ?operator ?bound ?unit
       (GROUP_CONCAT(DISTINCT STRAFTER(STR(?input), "#"); separator=", ") AS ?inputs)
WHERE {
  <${requirement}> requirement:hasCondition ?root .
  ?leaf expression:isSubconditionOf* ?root ;
        expression:constrains ?output ;
        expression:comparisonOperator ?operator .
  OPTIONAL {
    ?leaf expression:hasThreshold ?threshold .
    ?threshold expression:thresholdValue ?value ;
               expression:hasUnit ?unitInstance .
    ?unitInstance expression:unitSymbol ?unit .
  }
  OPTIONAL { ?leaf expression:targetValue ?target }
  BIND(COALESCE(STR(?value), ?target) AS ?bound)
  OPTIONAL { ?leaf expression:hasInput ?input }
}
GROUP BY ?leaf ?output ?operator ?bound ?unit
ORDER BY ?leaf
```
