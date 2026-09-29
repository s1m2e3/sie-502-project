---
template:
  id: http://www.example.com/method/requirements
  name: "Requirements"
  rank: 0
  expose:
    - kind: compose
  params:
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
# Requirements

State what the solar oven shall do. Each requirement is one verifiable "shall" statement,
carries exactly one Condition that gives its logic, is verified by a testing strategy, and
must be raised by at least one stakeholder.

```table-editor
---
columns: { this: { label: "Requirement" } }
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix base: <http://www.example.com/method/base#> .
@prefix expression: <http://www.example.com/method/expression#> .
@prefix requirement: <http://www.example.com/method/requirement#> .

requirement:RequirementShape
    a sh:NodeShape ;
    sh:targetClass requirement:Requirement ;
    sh:property [
        sh:path base:description ;
        sh:name "Description" ;
        dash:editor dash:TextAreaEditor ;
        sh:maxCount 1 ;
    ] ;
    sh:property [
        sh:path base:expression ;
        sh:name "Statement" ;
        dash:editor dash:TextAreaEditor ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
    ] ;
    sh:property [
        sh:path requirement:isRaisedBy ;
        sh:name "Raised by" ;
        sh:class requirement:Stakeholder ;
        sh:minCount 1 ;
        sh:message "A requirement must be raised by at least one stakeholder." ;
    ] ;
    sh:property [
        sh:path requirement:hasCondition ;
        sh:name "Condition" ;
        sh:class expression:Condition ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
    ] ;
    sh:property [
        sh:path requirement:isVerifiedBy ;
        sh:name "Verified by" ;
        sh:class requirement:TestingStrategy ;
        sh:minCount 1 ;
        sh:message "A requirement must be verified by at least one testing strategy." ;
    ] ;
    .
```
