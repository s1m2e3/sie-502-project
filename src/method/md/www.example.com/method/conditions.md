---
template:
  id: http://www.example.com/method/conditions
  name: "Conditions"
  rank: 3
  expose:
    - kind: compose
  params:
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
# Conditions

The logic behind each requirement. Work top to bottom: first define the thresholds, then the
comparisons that use them.

A comparison states exactly one bound: one output, one operator, one threshold. A range or
tolerance band is two comparisons, one for the lower bound (GE/GT) and one for the upper bound
(LE/LT), joined by an AND condition.

## Thresholds

A threshold is a value with its unit. Units come from the Quantities and Units page; if the
unit you need is not there, ask the method author to add it.

```table-editor
---
columns: { this: { label: "Threshold" } }
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix base: <http://www.example.com/method/base#> .
@prefix expression: <http://www.example.com/method/expression#> .

expression:ThresholdShape
    a sh:NodeShape ;
    sh:targetClass expression:Threshold ;
    sh:property [
        sh:path expression:hasUnit ;
        sh:name "Units" ;
        sh:class expression:Unit ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "A threshold needs exactly one unit, picked from the Quantities and Units page." ;
        sh:order 1 ;
    ] ;
    sh:property [
        sh:path expression:thresholdValue ;
        sh:name "Value" ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "A threshold needs exactly one numeric value." ;
        sh:order 2 ;
    ] ;
    sh:property [
        sh:path ( expression:hasUnit expression:measuresQuantity ) ;
        sh:name "Quantity" ;
        dash:readOnly true ;
        sh:order 3 ;
    ] ;
    sh:property [
        sh:path base:description ;
        sh:name "Description" ;
        dash:editor dash:TextAreaEditor ;
        sh:maxCount 1 ;
        sh:order 4 ;
    ] ;
    .
```

## Comparisons

```table-editor
---
columns: { this: { label: "Comparison" } }
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix base: <http://www.example.com/method/base#> .
@prefix expression: <http://www.example.com/method/expression#> .

expression:ComparisonShape
    a sh:NodeShape ;
    sh:targetClass expression:Comparison ;
    sh:property [
        sh:path expression:constrains ;
        sh:name "Constrains" ;
        sh:class expression:Output ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "A comparison constrains exactly one output." ;
        sh:order 1 ;
    ] ;
    sh:property [
        sh:path ( expression:constrains expression:hasQuantityKind ) ;
        sh:name "Kind" ;
        dash:readOnly true ;
        sh:order 2 ;
    ] ;
    sh:property [
        sh:path expression:comparisonOperator ;
        sh:name "Operator" ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "A comparison states exactly one bound, so it has exactly one operator. For a range, write one comparison for the lower bound (GE/GT) and one for the upper bound (LE/LT), joined by AND." ;
        sh:order 3 ;
    ] ;
    sh:property [
        sh:path expression:hasThreshold ;
        sh:name "Threshold" ;
        sh:class expression:Threshold ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "A comparison states exactly one bound, so it uses exactly one threshold. Define the threshold first, then pick it here." ;
        sh:order 4 ;
    ] ;
    sh:property [
        sh:path ( expression:hasThreshold expression:hasUnit ) ;
        sh:name "Units" ;
        dash:readOnly true ;
        sh:order 5 ;
    ] ;
    sh:property [
        sh:path ( expression:hasThreshold expression:thresholdValue ) ;
        sh:name "Value" ;
        dash:readOnly true ;
        sh:order 6 ;
    ] ;
    sh:property [
        sh:path expression:hasInput ;
        sh:name "Inputs" ;
        sh:class expression:Input ;
        sh:order 7 ;
    ] ;
    sh:property [
        sh:path base:description ;
        sh:name "Description" ;
        dash:editor dash:TextAreaEditor ;
        sh:maxCount 1 ;
        sh:order 8 ;
    ] ;
    sh:sparql [
        sh:message "A comparison can only constrain an output that has a quantity kind. For a categorical output (e.g. gas composition), use a Predicate instead." ;
        sh:select """
            PREFIX expression: <http://www.example.com/method/expression#>
            SELECT $this WHERE {
                $this expression:constrains ?output .
                FILTER NOT EXISTS { ?output expression:hasQuantityKind ?kind . }
            }
        """ ;
    ] ;
    sh:sparql [
        sh:message "The threshold's unit must measure the same quantity as the output (e.g. a temperature output needs a temperature unit)." ;
        sh:select """
            PREFIX expression: <http://www.example.com/method/expression#>
            SELECT $this WHERE {
                $this expression:constrains ?output ;
                      expression:hasThreshold ?threshold .
                ?output expression:hasQuantityKind ?outputKind .
                ?threshold expression:hasUnit ?unit .
                ?unit expression:measuresQuantity ?unitKind .
                FILTER (?outputKind != ?unitKind)
            }
        """ ;
    ] ;
    .
```
