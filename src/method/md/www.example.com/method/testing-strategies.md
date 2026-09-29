---
template:
  id: http://www.example.com/method/testing-strategies
  name: "Testing Strategies"
  rank: 2
  expose:
    - kind: compose
  params:
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
# Testing Strategies

Describe how the solar oven's requirements will be verified. Each testing strategy uses
exactly one verification method (Analysis, Inspection, Test, or Demonstration) and says
what is done. Requirements point to the strategies that verify them.

```table-editor
---
columns: { this: { label: "Testing Strategy" } }
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix base: <http://www.example.com/method/base#> .
@prefix requirement: <http://www.example.com/method/requirement#> .

requirement:TestingStrategyShape
    a sh:NodeShape ;
    sh:targetClass requirement:TestingStrategy ;
    sh:property [
        sh:path requirement:hasVerificationMethod ;
        sh:name "Method" ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "A testing strategy must have exactly one verification method (Analysis, Inspection, Test, or Demonstration)." ;
        sh:order 1 ;
    ] ;
    sh:property [
        sh:path base:description ;
        sh:name "Description" ;
        dash:editor dash:TextAreaEditor ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "A testing strategy must have a description saying what is done." ;
        sh:order 2 ;
    ] ;
    .
```
