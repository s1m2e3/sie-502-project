---
ontology: http://www.example.com/project/bundle
---
# Solar Oven Requirements Method: Notebook

This notebook walks through the method in the order it is meant to be used. Each step says what
it captures and why it comes where it does, then embeds the same editor that has its own page.
The editors are not copies: every block below is a `compose` of the method's template, pointed
at the ontology that step writes to. Editing here and editing on the step's own page change the
same model.

The pipeline is ordered so that every step only points at things that already exist:

1. **Quantities and Units** (method author): the measurement vocabulary.
2. **Stakeholders**: who has a stake.
3. **Testing Strategies**: how things will be verified.
4. **Conditions**: the logic of each requirement, from thresholds up to the condition tree.
5. **Requirements**: ties it all together: one condition, raised by stakeholders, verified by
   testing strategies.

The last section runs live checks over the whole model.

---

## Step 1. Quantities and Units

Before anyone writes a threshold, the method has to know what can be measured. Quantity kinds
(temperature, power, mass flow rate) and units (K, kW, kg/h) are data, not vocabulary, so a
project can add a unit without changing the method. Every quantity kind and unit carries its
dimension as exponents of the ISQ base dimensions (e.g. `M1 L2 T-3` for power), and the page's
rules check that a unit has the same dimension as the quantity it measures.

This step belongs to the method author. A user who needs a unit that is missing asks for it
instead of adding it, because a wrong unit here would pass every later check.

```compose
template: http://www.example.com/method/quantities
ontology: http://www.example.com/project/quantities
```

---

## Step 2. Stakeholders

A requirement exists because someone needs it, so stakeholders come first. A stakeholder is never
just a "Stakeholder": the editor only offers the concrete kinds (Regulator, Internal Client,
External Client, Press, Provider). The kind matters: a requirement raised by a Regulator is
classified by the reasoner as a **Safety Requirement** without anyone asserting it.

```compose
template: http://www.example.com/method/stakeholders
ontology: http://www.example.com/project/stakeholders
```

---

## Step 3. Testing Strategies

Each requirement must be verifiable, so verification strategies are defined before the
requirements that use them. A strategy names exactly one verification method: Analysis,
Inspection, Test or Demonstration.

```compose
template: http://www.example.com/method/testing-strategies
ontology: http://www.example.com/project/verification
```

---

## Step 4. Conditions

A requirement's text says what it means to a person; its condition says what it means to the
model. Conditions are built bottom-up, in three parts:

- **Thresholds**: a value with a unit picked from Step 1 (e.g. 673.15 K).
- **The condition tree**: AND/OR conditions combine two or more subconditions; ForAll/Exists
  conditions apply exactly one subcondition to every (or some) member of a set. Each child
  points to its parent through a single link, "Subcondition of". When you add a row, pick its
  kind first: the form then shows only that kind's fields (a connective for AND/OR, a quantifier
  and a set for ForAll/Exists).
- **Leaves**: a *comparison* bounds a numeric output with one operator and one threshold; a
  *predicate* checks a categorical output (e.g. gas composition) with EQ or NE.

A comparison states exactly one bound. A range, such as a power between a minimum and a maximum,
is two comparisons (a lower GE/GT and an upper LE/LT) under an AND. The comparison rules also
check that the threshold's unit measures the same quantity as the output it bounds, so a
temperature cannot be compared against a value in kW.

```compose
template: http://www.example.com/method/conditions
ontology: http://www.example.com/project/conditions
```

---

## Step 5. Requirements

Everything comes together here. A requirement has a short name, a "shall" statement, exactly one
condition (the root of its tree), at least one stakeholder who raises it, and at least one
testing strategy that verifies it. Because Steps 2 to 4 already exist, each of these is a pick
from a list, not free text.

```compose
template: http://www.example.com/method/requirements
ontology: http://www.example.com/project/requirements
```

---

## Live checks

These queries run over the whole bundle, after the editor's live reasoning (subclasses, inverses,
domains and ranges; not full OWL classification). They answer questions a single editor cannot,
because each editor only sees its own file.

### Traceability

Each requirement with who raised it, its condition, how it is verified, and whether it is a safety
requirement. The live view does not run full OWL classification, so the safety column applies the
SafetyRequirement definition directly (raised by some Regulator); `oml reason -o` gives the same
result (R2, R4, R6).

```table
PREFIX requirement: <http://www.example.com/method/requirement#>
PREFIX base: <http://www.example.com/method/base#>

SELECT ?requirement ?description
       (GROUP_CONCAT(DISTINCT STRAFTER(STR(?stakeholder), "#"); separator=", ") AS ?raisedBy)
       (SAMPLE(?condition) AS ?rootCondition)
       (GROUP_CONCAT(DISTINCT STRAFTER(STR(?strategy), "#"); separator=", ") AS ?verifiedBy)
       (IF(EXISTS { ?requirement requirement:isRaisedBy ?regulator . ?regulator a requirement:Regulator }, "yes", "no") AS ?safety)
WHERE {
  ?requirement a requirement:Requirement .
  OPTIONAL { ?requirement base:description ?description }
  OPTIONAL { ?requirement requirement:isRaisedBy ?stakeholder }
  OPTIONAL { ?requirement requirement:hasCondition ?condition }
  OPTIONAL { ?requirement requirement:isVerifiedBy ?strategy }
}
GROUP BY ?requirement ?description
ORDER BY ?requirement
```

### Stakeholders who raise nothing

The method says a stakeholder is only modeled if it raises at least one requirement. That rule
spans two files (stakeholders and requirements), so no single editor can enforce it. This table
should be empty.

```table
PREFIX requirement: <http://www.example.com/method/requirement#>
PREFIX base: <http://www.example.com/method/base#>

SELECT ?stakeholder ?description WHERE {
  ?stakeholder a requirement:Stakeholder .
  OPTIONAL { ?stakeholder base:description ?description }
  FILTER NOT EXISTS { ?requirement requirement:isRaisedBy ?stakeholder }
}
ORDER BY ?stakeholder
```

### Testing strategies that verify nothing

Same idea for verification: a strategy nobody uses is dead weight. This table should be empty.

```table
PREFIX requirement: <http://www.example.com/method/requirement#>
PREFIX base: <http://www.example.com/method/base#>

SELECT ?strategy ?method ?description WHERE {
  ?strategy a requirement:TestingStrategy .
  OPTIONAL { ?strategy requirement:hasVerificationMethod ?method }
  OPTIONAL { ?strategy base:description ?description }
  FILTER NOT EXISTS { ?requirement requirement:isVerifiedBy ?strategy }
}
ORDER BY ?strategy
```

### Every bound in the model

All comparisons with the output they bound, the operator, and the threshold with its unit. This
is the numeric content of the requirements in one place, useful for review: a wrong value or a
wrong direction (GE where LE was meant) shows up here faster than in the tree.

```table
PREFIX expression: <http://www.example.com/method/expression#>

SELECT ?comparison ?output ?operator ?value ?unit ?requirement WHERE {
  ?comparison a expression:Comparison ;
              expression:constrains ?output ;
              expression:comparisonOperator ?operator ;
              expression:hasThreshold ?threshold .
  ?threshold expression:thresholdValue ?value ;
             expression:hasUnit ?u .
  ?u expression:unitSymbol ?unit .
  OPTIONAL {
    ?comparison expression:isSubconditionOf* ?root .
    ?requirement <http://www.example.com/method/requirement#hasCondition> ?root .
  }
}
ORDER BY ?requirement ?comparison
```

---

## Dogfooding

R6, R7 and the stakeholders, strategies, thresholds and conditions behind them were entered
through the editors above. R6 was entered as a plain requirement; the Traceability table shows
it as a safety requirement because a Regulator raised it. What the pass changed in the method is
recorded in [METHOD.md](../../../../METHOD.md).
