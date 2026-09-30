# Method: Solar Oven Requirements

This file explains why the method is built the way it is: the choices, the alternatives that
were dropped, and what is still uncertain or open. The model itself is in `src/method/oml`, and
the [Notebook](src/model/md/Solar%20Oven/Notebook.md) walks through it.

## What the method prescribes

A requirement is built in five steps, and each step only points at things an earlier step
created, so every link is a pick from a list:

1. **Quantities and Units** (method author): quantity kinds and units, each with its dimension.
2. **Stakeholders**: always one of the concrete kinds (Regulator, Internal Client, External
   Client, Press, Provider).
3. **Testing Strategies**: one verification method each.
4. **Conditions**: thresholds (value + unit), then the condition tree (AND/OR, ForAll/Exists),
   then the leaves (comparisons for numeric outputs, predicates for categorical ones).
5. **Requirements**: a "shall" statement with one root condition, the stakeholders who raise
   it and the strategies that verify it.

Each step has one editor, exposed as a `compose` template so the same editor is reused on its
own page and in the Notebook. A comparison states exactly one bound, so a range is two
comparisons (lower GE/GT, upper LE/LT) under an AND. A requirement raised by a Regulator is
classified as a `SafetyRequirement` by the reasoner; nobody asserts it.

## Layout

```
src/method/oml/   vocabularies: base, units, expression, requirement, bundle
src/method/md/    one editor template per pattern
src/model/oml/    one description per pattern + a bundle that includes them all
src/model/md/     one page per pattern, the Notebook, an index
```

- **One file per pattern**, so each editor writes to exactly one description and a change stays
  local (adding a stakeholder never touches the requirements file).
- **Imports follow traceability**: `conditions` extends `quantities` and `system`;
  `requirements` extends `conditions`, `verification` and `stakeholders`.
- **Links are stored on the dependent side**: a requirement stores `isRaisedBy`, a child
  condition stores `isSubconditionOf`. Validation only sees a file and its imports, and the tree
  editor nests rows by a child-to-parent link. The reverse links are inferred.

## Decisions and alternatives

- **Abstract types, concrete editors.** OML has no abstract concepts or unions, so `Stakeholder`
  and `Condition` are kept abstract by the editors: each shape targets only the concrete kinds.
- **Measurements as data.** The first design had one comparison subtype per quantity
  (temperature, power, ...) with OML quantity properties. Every new measurement meant a
  vocabulary change, and the workarounds failed (OML does not allow specializing a quantity
  property; alternative paths and same-named columns do not merge in the editor). Quantity kinds,
  units and thresholds are now instances, so a project adds a measurement by adding data.
- **One parent link.** The tree first used `hasOperand` for AND/OR and `hasBody` for
  ForAll/Exists. The tree editor nests by a single link, so they were merged into
  `hasSubcondition`; a child's role follows from its parent.
- **One shape per condition kind.** While entering data, adding an AND condition showed the
  quantifier fields instead of the connective. The editor takes each kind's form from the last
  shape that targets it, so each kind now has one shape holding its own fields and rules.
- **Enumerations** (operators, connectives, verification methods) are OML `oneOf` scalars and
  are not repeated as `sh:in` lists in the shapes.

## Uncertainty

- **The reasoner does not replace validation.** When rules were broken on purpose (no
  threshold, a child under a leaf, a kW threshold on a temperature), `oml reason` passed every
  time and only `oml validate` caught them. OWL is open-world, so a missing value is "not
  stated", not an error. The vocabulary says what things are; the shapes say what a user must
  enter.
- **Units are checked by the method, not by OML.** The shapes compare a unit's quantity and
  dimension with the output's. A unit that is consistent but physically wrong still passes, so
  the Quantities page belongs to the method author and users ask for new units. SHACL cannot
  enforce who edits a page; this is an agreement, not a rule.
- **Dimensions** are written as exponents in a fixed order (M L T I H N J, e.g. `M1 L2 T-3`),
  and a dimensionless quantity has dimension `1`.

## Open issues

- **Live queries do not see defined classes.** The Notebook's live reasoning skips definitions
  like `SafetyRequirement`; only `oml reason -o` classifies them. The Notebook's safety column
  therefore restates the definition in SPARQL and must change if the definition does.
- **Rules across files are only queries.** "Every stakeholder raises a requirement" and "every
  strategy verifies one" span two files, so no editor enforces them; the Notebook checks them.
- **Dimensions are compared as text**, so writing one in a different order is a false alarm.
- **No unit conversion**: the same quantity in two units cannot be compared.
- **Hand-written OML bypasses the editors**: a plain `Stakeholder` or `Condition` written
  directly is not caught.
