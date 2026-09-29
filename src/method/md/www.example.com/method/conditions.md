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

The logic behind each requirement. Work top to bottom: first define the thresholds, then build
the condition tree, then fill in the details of its leaves (comparisons and predicates).

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

## Condition tree

The structure of each condition. AND/OR conditions hold two or more operands; ForAll/Exists
conditions hold exactly one body. Add a child under its parent; the leaves (comparisons and
predicates) are detailed in the tables below.

```tree-editor
---
columns: { this: { label: "Condition" } }
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix base: <http://www.example.com/method/base#> .
@prefix expression: <http://www.example.com/method/expression#> .

expression:ConditionTreeShape
    a sh:NodeShape ;
    sh:targetClass expression:CompoundCondition ;
    sh:targetClass expression:QuantifiedCondition ;
    sh:targetClass expression:Comparison ;
    sh:targetClass expression:Predicate ;
    sh:property [
        sh:path rdf:type ;
        sh:name "Kind" ;
        sh:order 1 ;
    ] ;
    sh:property [
        sh:path expression:isOperandOf ;
        sh:name "Operand of" ;
        sh:class expression:CompoundCondition ;
        sh:maxCount 1 ;
        dash:composite true ;
    ] ;
    sh:property [
        sh:path expression:isBodyOf ;
        sh:name "Body of" ;
        sh:class expression:QuantifiedCondition ;
        sh:maxCount 1 ;
        dash:composite true ;
    ] ;
    sh:property [
        sh:path expression:connective ;
        sh:name "Connective" ;
        sh:maxCount 1 ;
        sh:order 2 ;
    ] ;
    sh:property [
        sh:path expression:quantifier ;
        sh:name "Quantifier" ;
        sh:maxCount 1 ;
        sh:order 3 ;
    ] ;
    sh:property [
        sh:path expression:quantifiesOver ;
        sh:name "Over" ;
        sh:class expression:Output ;
        sh:maxCount 1 ;
        sh:order 4 ;
    ] ;
    sh:property [
        sh:path base:description ;
        sh:name "Description" ;
        dash:editor dash:TextAreaEditor ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "A condition must have a description stating it in words." ;
        sh:order 5 ;
    ] ;
    .

expression:CompoundConditionShape
    a sh:NodeShape ;
    sh:targetClass expression:CompoundCondition ;
    sh:property [
        sh:path expression:connective ;
        sh:minCount 1 ;
        sh:message "An AND/OR condition needs its connective." ;
    ] ;
    sh:property [
        sh:path [ sh:inversePath expression:isOperandOf ] ;
        sh:minCount 2 ;
        sh:message "An AND/OR condition needs at least two operands." ;
    ] ;
    .

expression:QuantifiedConditionShape
    a sh:NodeShape ;
    sh:targetClass expression:QuantifiedCondition ;
    sh:property [
        sh:path expression:quantifier ;
        sh:minCount 1 ;
        sh:message "A ForAll/Exists condition needs its quantifier." ;
    ] ;
    sh:property [
        sh:path expression:quantifiesOver ;
        sh:minCount 1 ;
        sh:message "A ForAll/Exists condition needs the set it ranges over." ;
    ] ;
    sh:property [
        sh:path [ sh:inversePath expression:isBodyOf ] ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "A ForAll/Exists condition needs exactly one body." ;
    ] ;
    .

expression:NotCompoundShape
    a sh:NodeShape ;
    sh:targetClass expression:QuantifiedCondition ;
    sh:targetClass expression:Comparison ;
    sh:targetClass expression:Predicate ;
    sh:property [
        sh:path expression:connective ;
        sh:maxCount 0 ;
        sh:message "Only an AND/OR condition has a connective; leave Connective empty here." ;
    ] ;
    .

expression:NotQuantifiedShape
    a sh:NodeShape ;
    sh:targetClass expression:CompoundCondition ;
    sh:targetClass expression:Comparison ;
    sh:targetClass expression:Predicate ;
    sh:property [
        sh:path expression:quantifier ;
        sh:maxCount 0 ;
        sh:message "Only a ForAll/Exists condition has a quantifier; leave Quantifier empty here." ;
    ] ;
    sh:property [
        sh:path expression:quantifiesOver ;
        sh:maxCount 0 ;
        sh:message "Only a ForAll/Exists condition ranges over a set; leave Over empty here." ;
    ] ;
    .
```

## Comparisons

A comparison checks a numeric output (one with a quantity kind, e.g. outlet temperature)
against one threshold, with one operator (EQ, NE, GT, GE, LT or LE). The threshold's unit must
measure the same quantity as the output.

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

## Predicates

A predicate checks a categorical output (one with no quantity kind, e.g. gas composition)
against a value, with EQ or NE.

```table-editor
---
columns: { this: { label: "Predicate" } }
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix base: <http://www.example.com/method/base#> .
@prefix expression: <http://www.example.com/method/expression#> .

expression:PredicateShape
    a sh:NodeShape ;
    sh:targetClass expression:Predicate ;
    sh:property [
        sh:path expression:constrains ;
        sh:name "Constrains" ;
        sh:class expression:Output ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "A predicate constrains exactly one output." ;
        sh:order 1 ;
    ] ;
    sh:property [
        sh:path expression:comparisonOperator ;
        sh:name "Operator" ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "A predicate has exactly one operator, EQ or NE." ;
        sh:order 2 ;
    ] ;
    sh:property [
        sh:path expression:targetValue ;
        sh:name "Target value" ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "A predicate needs exactly one target value (e.g. \"nitrogen\")." ;
        sh:order 3 ;
    ] ;
    sh:property [
        sh:path expression:hasInput ;
        sh:name "Inputs" ;
        sh:class expression:Input ;
        sh:order 4 ;
    ] ;
    sh:property [
        sh:path base:description ;
        sh:name "Description" ;
        dash:editor dash:TextAreaEditor ;
        sh:maxCount 1 ;
        sh:order 5 ;
    ] ;
    sh:sparql [
        sh:message "A predicate compares a category, so only EQ or NE apply. For an ordering (GT, GE, LT, LE), use a Comparison." ;
        sh:select """
            PREFIX expression: <http://www.example.com/method/expression#>
            SELECT $this WHERE {
                $this expression:comparisonOperator ?operator .
                FILTER (STR(?operator) != "EQ" && STR(?operator) != "NE")
            }
        """ ;
    ] ;
    sh:sparql [
        sh:message "A predicate can only constrain a categorical output. This output has a quantity kind, so use a Comparison with a threshold instead." ;
        sh:select """
            PREFIX expression: <http://www.example.com/method/expression#>
            SELECT $this WHERE {
                $this expression:constrains ?output .
                ?output expression:hasQuantityKind ?kind .
            }
        """ ;
    ] ;
    .
```
