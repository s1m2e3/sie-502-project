---
template:
  id: http://www.example.com/method/stakeholders
  name: "Stakeholders"
  rank: 1
  expose:
    - kind: compose
  params:
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
# Stakeholders

List every party with a stake in the solar oven. A stakeholder is never created directly:
pick its kind (Regulator, Internal client, External client, Press, or Provider) and describe
who they are. Requirements raised by a Regulator are classified as SafetyRequirements by the
reasoner.

```table-editor
---
columns: { this: { label: "Stakeholder" } }
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix base: <http://www.example.com/method/base#> .
@prefix requirement: <http://www.example.com/method/requirement#> .

requirement:StakeholderShape
    a sh:NodeShape ;
    sh:targetClass requirement:Regulator ;
    sh:targetClass requirement:InternalClient ;
    sh:targetClass requirement:ExternalClient ;
    sh:targetClass requirement:Press ;
    sh:targetClass requirement:Provider ;
    sh:property [
        sh:path rdf:type ;
        sh:name "Kind" ;
        sh:order 1 ;
    ] ;
    sh:property [
        sh:path base:description ;
        sh:name "Description" ;
        dash:editor dash:TextAreaEditor ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "A stakeholder must have a description saying who they are." ;
        sh:order 2 ;
    ] ;
    .
```
