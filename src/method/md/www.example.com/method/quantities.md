---
template:
  id: http://www.example.com/method/quantities
  name: "Quantities and Units"
  rank: 4
  expose:
    - kind: compose
  params:
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
# Quantities and Units

**Author-owned.** This page is maintained by the method author. Users do not edit it: to
measure something new, request the quantity kind and unit from the method author, who
reviews it for physical correctness and adds it here. The rules below keep every entry
consistent; the review keeps it true.

Every quantity kind is either **Dimensional** or **Dimensionless**:

- Dimensional quantities have a physical dimension, written as exponents of the ISQ base
  dimensions in the order M, L, T, I, H, N, J, omitting zero exponents (e.g. power is
  "M1 L2 T-3", temperature is "H1").
- Dimensionless quantities (ratios, efficiencies, statistical metrics such as a coefficient of
  variation) have dimension "1". Their units also have dimension "1" (e.g. "1", "%", "ppm").

## Quantity kinds

```table-editor
---
columns: { this: { label: "Quantity kind" } }
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix base: <http://www.example.com/method/base#> .
@prefix expression: <http://www.example.com/method/expression#> .

expression:QuantityKindShape
    a sh:NodeShape ;
    sh:targetClass expression:QuantityKind ;
    sh:property [
        sh:path expression:hasDimensionality ;
        sh:name "Dimensionality" ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "A quantity kind must be either Dimensional or Dimensionless." ;
        sh:order 1 ;
    ] ;
    sh:property [
        sh:path expression:hasDimension ;
        sh:name "Dimension" ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "A quantity kind needs exactly one dimension (e.g. \"M1 L2 T-3\" for power, \"1\" if dimensionless)." ;
        sh:order 2 ;
    ] ;
    sh:property [
        sh:path base:description ;
        sh:name "Description" ;
        dash:editor dash:TextAreaEditor ;
        sh:maxCount 1 ;
        sh:order 3 ;
    ] ;
    sh:sparql [
        sh:message "A Dimensionless quantity kind has dimension \"1\"; a Dimensional one has a physical dimension other than \"1\"." ;
        sh:select """
            PREFIX expression: <http://www.example.com/method/expression#>
            SELECT $this WHERE {
                $this expression:hasDimensionality ?dimensionality ;
                      expression:hasDimension ?dimension .
                FILTER (
                    (STR(?dimensionality) = "Dimensionless" && STR(?dimension) != "1") ||
                    (STR(?dimensionality) = "Dimensional" && STR(?dimension) = "1")
                )
            }
        """ ;
    ] ;
    .
```

## Units

```table-editor
---
columns: { this: { label: "Unit" } }
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix base: <http://www.example.com/method/base#> .
@prefix expression: <http://www.example.com/method/expression#> .

expression:UnitShape
    a sh:NodeShape ;
    sh:targetClass expression:Unit ;
    sh:property [
        sh:path expression:unitSymbol ;
        sh:name "Symbol" ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "A unit needs exactly one symbol (e.g. \"kW\")." ;
        sh:order 1 ;
    ] ;
    sh:property [
        sh:path expression:measuresQuantity ;
        sh:name "Measures" ;
        sh:class expression:QuantityKind ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "A unit measures exactly one quantity kind." ;
        sh:order 2 ;
    ] ;
    sh:property [
        sh:path expression:hasDimension ;
        sh:name "Dimension" ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "A unit needs exactly one dimension, the same as the quantity kind it measures (\"1\" for a dimensionless unit such as %)." ;
        sh:order 3 ;
    ] ;
    sh:property [
        sh:path base:description ;
        sh:name "Description" ;
        dash:editor dash:TextAreaEditor ;
        sh:maxCount 1 ;
        sh:order 4 ;
    ] ;
    sh:sparql [
        sh:message "A unit's dimension must match the dimension of the quantity kind it measures (e.g. bar is \"M1 L-1 T-2\", so it cannot measure Temperature, \"H1\")." ;
        sh:select """
            PREFIX expression: <http://www.example.com/method/expression#>
            SELECT $this WHERE {
                $this expression:measuresQuantity ?kind ;
                      expression:hasDimension ?unitDimension .
                ?kind expression:hasDimension ?kindDimension .
                FILTER (?unitDimension != ?kindDimension)
            }
        """ ;
    ] ;
    .
```
