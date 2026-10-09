---
ontology: http://www.example.com/project/bundle
---
# Solar Oven: Analysis Dashboard

The editors keep each file correct on its own. This page asks questions of the whole model:
what is missing, and what the model can and cannot answer. Every query runs over the project
bundle, so it sees all the files at once.

---

## Gap detection

Each editor only sees its own file, so a rule that spans two files cannot be enforced while
typing. The checks below look for elements that are missing a link. There is one group per
pattern. Every table lists all the elements of its kind with a `check` column: ✅ when the
link is there, ⚠️ GAP when it is missing. Gaps sort to the top.

### Stakeholders

**Does every stakeholder raise a requirement?** The method models a stakeholder only if it
raises at least one requirement. A `GAP` is a stakeholder nobody needed, or a requirement that
forgot to name them.

```table
PREFIX requirement: <http://www.example.com/method/requirement#>

SELECT ?stakeholder
       (GROUP_CONCAT(DISTINCT STRAFTER(STR(?requirement), "#"); separator=", ") AS ?raises)
       (IF(COUNT(DISTINCT ?requirement) = 0, "⚠️ GAP", "✅") AS ?check)
WHERE {
  ?stakeholder a requirement:Stakeholder .
  OPTIONAL { ?requirement requirement:isRaisedBy ?stakeholder }
}
GROUP BY ?stakeholder
ORDER BY ?check ?stakeholder
```

### Testing strategies

**Does every strategy verify a requirement?** A `GAP` is a strategy that was defined and never
used.

```table
PREFIX requirement: <http://www.example.com/method/requirement#>

SELECT ?strategy ?method
       (GROUP_CONCAT(DISTINCT STRAFTER(STR(?requirement), "#"); separator=", ") AS ?verifies)
       (IF(COUNT(DISTINCT ?requirement) = 0, "⚠️ GAP", "✅") AS ?check)
WHERE {
  ?strategy a requirement:TestingStrategy .
  OPTIONAL { ?strategy requirement:hasVerificationMethod ?method }
  OPTIONAL { ?requirement requirement:isVerifiedBy ?strategy }
}
GROUP BY ?strategy ?method
ORDER BY ?check ?strategy
```

### Requirements

**Does every requirement have all its links?** A requirement needs exactly one condition, at
least one stakeholder and at least one testing strategy. The editor checks this while typing;
this query checks it again over the whole model, so a requirement written by hand is caught
too. The three counts show which link a `GAP` is missing.

```table
PREFIX requirement: <http://www.example.com/method/requirement#>

SELECT ?requirement
       (COUNT(DISTINCT ?condition)   AS ?conditions)
       (COUNT(DISTINCT ?stakeholder) AS ?stakeholders)
       (COUNT(DISTINCT ?strategy)    AS ?strategies)
       (IF(COUNT(DISTINCT ?condition) = 1 && COUNT(DISTINCT ?stakeholder) > 0 && COUNT(DISTINCT ?strategy) > 0,
           "✅", "⚠️ GAP") AS ?check)
WHERE {
  ?requirement a requirement:Requirement .
  OPTIONAL { ?requirement requirement:hasCondition ?condition }
  OPTIONAL { ?requirement requirement:isRaisedBy ?stakeholder }
  OPTIONAL { ?requirement requirement:isVerifiedBy ?strategy }
}
GROUP BY ?requirement
ORDER BY ?check ?requirement
```

### Conditions

**Does every condition tree belong to a requirement?** A condition with no parent is the root
of a tree; the table lists the roots. A `GAP` is a root that no requirement points to: logic
somebody wrote that states nothing.

```table
PREFIX expression:  <http://www.example.com/method/expression#>
PREFIX requirement: <http://www.example.com/method/requirement#>

SELECT ?condition ?kind ?requirement ?check WHERE {
  VALUES ?kind { expression:CompoundCondition expression:QuantifiedCondition
                 expression:Comparison expression:Predicate }
  ?condition a ?kind .
  FILTER NOT EXISTS { ?condition expression:isSubconditionOf ?parent }
  OPTIONAL { ?requirement requirement:hasCondition ?condition }
  BIND(IF(BOUND(?requirement), "✅", "⚠️ GAP") AS ?check)
}
ORDER BY ?check ?condition
```

**Is every threshold used by a comparison?** A `GAP` is a value that bounds nothing.

```table
PREFIX expression: <http://www.example.com/method/expression#>

SELECT ?threshold ?value
       (GROUP_CONCAT(DISTINCT STRAFTER(STR(?comparison), "#"); separator=", ") AS ?usedBy)
       (IF(COUNT(DISTINCT ?comparison) = 0, "⚠️ GAP", "✅") AS ?check)
WHERE {
  ?threshold a expression:Threshold .
  OPTIONAL { ?threshold expression:thresholdValue ?value }
  OPTIONAL { ?comparison expression:hasThreshold ?threshold }
}
GROUP BY ?threshold ?value
ORDER BY ?check ?threshold
```

### Quantities and units

**Is every unit used by a threshold?** A `GAP` is a unit the method author added and no
requirement needed.

```table
PREFIX expression: <http://www.example.com/method/expression#>

SELECT ?unit ?symbol
       (COUNT(DISTINCT ?threshold) AS ?thresholds)
       (IF(COUNT(DISTINCT ?threshold) = 0, "⚠️ GAP", "✅") AS ?check)
WHERE {
  ?unit a expression:Unit .
  OPTIONAL { ?unit expression:unitSymbol ?symbol }
  OPTIONAL { ?threshold expression:hasUnit ?unit }
}
GROUP BY ?unit ?symbol
ORDER BY ?check ?unit
```

### System

**Is every input and output mentioned by a condition?** An output with no condition is measured
but never bounded. An input with no condition is imposed on the system but no requirement
depends on it.

```table
PREFIX expression: <http://www.example.com/method/expression#>

SELECT ?element ?kind
       (COUNT(DISTINCT ?condition) AS ?conditions)
       (IF(COUNT(DISTINCT ?condition) = 0, "⚠️ GAP", "✅") AS ?check)
WHERE {
  {
    ?element a expression:Output .
    BIND("output" AS ?kind)
    OPTIONAL { ?condition expression:constrains ?element }
  }
  UNION
  {
    ?element a expression:Input .
    BIND("input" AS ?kind)
    OPTIONAL { ?condition expression:hasInput ?element }
  }
}
GROUP BY ?element ?kind
ORDER BY ?check ?kind ?element
```

**Does every entity have something observed on it?** An entity exists so that inputs and
outputs can be observed on it. `Pipe` is the one `GAP`: it only groups its insulated and bare
sections (the `parts` column), and the temperatures are observed on those. That is a container
by design, not an error. Any other `GAP` would be an entity the model does not need yet.

```table
PREFIX expression: <http://www.example.com/method/expression#>

SELECT ?entity
       (GROUP_CONCAT(DISTINCT STRAFTER(STR(?observable), "#"); separator=", ") AS ?observables)
       (COUNT(DISTINCT ?part) AS ?parts)
       (IF(COUNT(DISTINCT ?observable) = 0, "⚠️ GAP", "✅") AS ?check)
WHERE {
  ?entity a expression:Entity .
  OPTIONAL { ?entity expression:exhibits|^expression:observedOn ?observable }
  OPTIONAL { ?entity expression:hasPart|^expression:isPartOf ?part }
}
GROUP BY ?entity
ORDER BY ?check ?entity
```

---

## Safety requirements and verification

Two of the questions the project started from are about safety. A safety requirement is one
raised by a Regulator; the queries apply that definition directly, because the live view does
not classify defined concepts.

### Are the safety requirements on the inputs traced to a verification strategy?

**Evidence.** The table follows each safety requirement to the leaves of its condition tree,
collects the inputs those leaves mention, and shows the strategy that verifies it.

```table
PREFIX requirement: <http://www.example.com/method/requirement#>
PREFIX expression:  <http://www.example.com/method/expression#>

SELECT ?requirement
       (GROUP_CONCAT(DISTINCT STRAFTER(STR(?input), "#"); separator=", ") AS ?inputs)
       (GROUP_CONCAT(DISTINCT STRAFTER(STR(?strategy), "#"); separator=", ") AS ?verifiedBy)
       (GROUP_CONCAT(DISTINCT STR(?method); separator=", ") AS ?methods)
       (IF(COUNT(DISTINCT ?strategy) = 0, "⚠️ GAP", "✅") AS ?check)
WHERE {
  ?requirement a requirement:Requirement ;
               requirement:isRaisedBy ?regulator .
  ?regulator a requirement:Regulator .
  OPTIONAL {
    ?requirement requirement:hasCondition ?root .
    ?leaf expression:isSubconditionOf* ?root ;
          expression:hasInput ?input .
  }
  OPTIONAL {
    ?requirement requirement:isVerifiedBy ?strategy .
    OPTIONAL { ?strategy requirement:hasVerificationMethod ?method }
  }
}
GROUP BY ?requirement
ORDER BY ?check ?requirement
```

The matrix shows the same links from the input side. A `1` means the safety requirement
mentions that input; a `0` means it does not.

```matrix
---
rowColumnLabel: Input / Safety requirement
stylesheet:
  - selector: cell [Number(value) === 0]
    style:
      background-color: black
  - selector: cell [Number(value) > 0]
    style:
      background-color: darkblue
---
PREFIX requirement: <http://www.example.com/method/requirement#>
PREFIX expression:  <http://www.example.com/method/expression#>

SELECT ?row ?column (COALESCE(?n, 0) AS ?value)
WHERE {
  # 1. all rows: every input
  { SELECT DISTINCT ?row WHERE { ?row a expression:Input . } }

  # 2. all columns: every safety requirement
  { SELECT DISTINCT ?column WHERE {
      ?column a requirement:Requirement ;
              requirement:isRaisedBy ?regulator .
      ?regulator a requirement:Regulator .
  } }

  # 3. 1 where a leaf of the requirement's condition mentions the input
  OPTIONAL {
    SELECT DISTINCT ?row ?column (1 AS ?n)
    WHERE {
      ?column requirement:hasCondition ?root .
      ?leaf expression:isSubconditionOf* ?root ;
            expression:hasInput ?row .
    }
  }
}
ORDER BY ?row ?column
```

**Interpretation.** Yes: R2, R4 and R6 are each verified by a strategy. The matrix shows what
they depend on: only the two modes. The `SolarIrradiance` row is all zeros, so no safety
requirement mentions sunlight, although sunlight is what heats the pipe. R2 has to hold "during
operation" at any irradiance, and its thermal simulation has no stated sunlight range to cover.
The gap is in the model's content, not in the method: the pattern lets a condition name its
inputs, and R2's conditions do not name this one.

### Are the safety requirements on the outputs traced to a verification strategy?

**Evidence.** Outputs as rows, safety requirements as columns. A `1` means a leaf of the
requirement's condition constrains that output. R4 constrains the set of accessible surface
temperatures, so the members of that set count as covered too. The strategies are in the table
above.

```matrix
---
rowColumnLabel: Output / Safety requirement
stylesheet:
  - selector: cell [Number(value) === 0]
    style:
      background-color: black
  - selector: cell [Number(value) > 0]
    style:
      background-color: darkblue
---
PREFIX requirement: <http://www.example.com/method/requirement#>
PREFIX expression:  <http://www.example.com/method/expression#>

SELECT ?row ?column (COALESCE(?n, 0) AS ?value)
WHERE {
  # 1. all rows: every output
  { SELECT DISTINCT ?row WHERE { ?row a expression:Output . } }

  # 2. all columns: every safety requirement
  { SELECT DISTINCT ?column WHERE {
      ?column a requirement:Requirement ;
              requirement:isRaisedBy ?regulator .
      ?regulator a requirement:Regulator .
  } }

  # 3. 1 where a leaf constrains the output, or constrains a set the output is a member of
  OPTIONAL {
    SELECT DISTINCT ?row ?column (1 AS ?n)
    WHERE {
      ?column requirement:hasCondition ?root .
      ?leaf expression:isSubconditionOf* ?root .
      { ?leaf expression:constrains ?row . }
      UNION
      { ?leaf expression:constrains ?set .
        ?set expression:hasMember ?row . }
    }
  }
}
ORDER BY ?row ?column
```

**Interpretation.** The four temperature outputs are each bounded by a verified safety
requirement. Three outputs have a row of zeros: captured power, mass flow and gas composition.
The one that matters is `OutletGasComposition`. R3 keeps oxygen out of the pyrolysis reactor,
which is a safety concern, but only the reactor managers raised it, so the method does not
count it as a safety requirement. This is a gap in the pattern: whether a requirement is
"safety" depends on who raised it, not on what it protects.

---

## System structure

### How do the components react to changes in the input?

**Evidence.** The graph reads left to right: inputs (gold), entities (green), outputs (blue).
The method has no component concept, so the middle column is its closest stand-in: the entities
that outputs are observed on. An entity `exhibits` an output; that link is in the model. The red
`affects` edges are not in the model: the query draws one from an input to an entity whenever a
condition mentions that input and bounds an output of that entity.

```graph
---
layout:
  mode: dag
  fit: true
  padding: 24
  dag:
    rankDir: LR
    nodeSep: 30
    rankSep: 140
stylesheet:
  - selector: "node [outgoing.some(e => e.value.endsWith('#affects'))]"
    style:
      fill: gold
  - selector: "node [incoming.some(e => e.value.endsWith('#affects') || e.value.endsWith('#hasPart')) || outgoing.some(e => e.value.endsWith('#exhibits') || e.value.endsWith('#hasPart'))]"
    style:
      fill: lightgreen
  - selector: "node [incoming.some(e => e.value.endsWith('#exhibits'))]"
    style:
      fill: deepskyblue
  - selector: "edge [value.endsWith('#affects')]"
    style:
      stroke: tomato
      stroke-width: 2
---
PREFIX expression: <http://www.example.com/method/expression#>

CONSTRUCT {
  ?input  expression:affects  ?affected .
  ?entity expression:exhibits ?output .
  ?whole  expression:hasPart  ?part .
}
WHERE {
  {
    # 1. derived: a condition mentions the input and bounds an output of the entity
    #    (directly, or through a set the output is a member of)
    {
      ?leaf expression:hasInput   ?input ;
            expression:constrains ?reached .
    }
    UNION
    {
      ?leaf expression:hasInput   ?input ;
            expression:constrains ?set .
      ?set expression:hasMember ?reached .
    }
    ?affected expression:exhibits|^expression:observedOn ?reached .
  }
  UNION
  {
    # 2. the outputs each entity exhibits
    ?output a expression:Output .
    ?entity expression:exhibits|^expression:observedOn ?output .
  }
  UNION
  { ?whole expression:hasPart ?part . }
}
```

**Interpretation.** The question cannot be answered as asked, because the model has no
components. What it has are entities, and the graph shows what that word is covering:

- **The green column mixes two different things.** The lens and the pipe sections are parts of
  the system. `NitrogenStream` is not: it is the flow that crosses the boundary, coming in from
  the generator and going out to the reactor. It is drawn like the lens because the vocabulary
  has one word, entity, for anything that has attributes to observe.
- **Only the outflow side of the nitrogen is modeled.** All three of its attributes are outlet
  values (temperature, mass flow, composition). Nothing describes the nitrogen coming in, so
  the model cannot say what the system does to it.
- **Every input-to-entity edge is red.** The system description never says an entity receives
  an input: sunlight is observed on the installation site and the modes on the system state,
  and neither is linked to the lens or the pipe. The only connection comes from the
  requirements.
- **Sunlight reaches only the lens and the nitrogen stream.** It does not reach either pipe
  section, although the bare section is what the sunlight heats.
- **`Pipe` is attached only through its two sections**, with no input and no output of its own.

This is a diagnosed gap in the vocabulary. The method needs a component concept that is
separate from the flows that pass through it, a way to say that a component receives an input,
and inlet as well as outlet attributes on a flow. Once flows pass through components, the
method also needs ports: the place on a component where a flow enters or leaves. A port is
what would let the model say that the nitrogen enters the pipe at one end and leaves at the
other, and that the lens delivers its power to the bare section. The vocabulary also makes entity, input and
output mutually exclusive, so the nitrogen stream cannot be declared as what it is: a flow that
is an input on one side and an output on the other. The size of the response is a separate
limit: it is outside a structural model and needs a simulation; the computed analysis below
takes a first step.

### Does the architecture allow modular replacement of the components?

**Evidence.** The tree is everything the model knows about structure: the entities, nested by
`hasPart`, with what is observed on each.

```tree
---
columns: { this: { label: "Entity" }, observables: { label: "Observed on it" } }
containment: [ expression:hasPart ]
---
PREFIX expression: <http://www.example.com/method/expression#>

CONSTRUCT {
    ?entity expression:hasPart ?part .
    ?entity expression:observables ?observables .
}
WHERE {
    {
      SELECT ?entity
             (GROUP_CONCAT(DISTINCT STRAFTER(STR(?observable), "#"); separator=", ") AS ?observables)
      WHERE {
        ?entity a expression:Entity .
        OPTIONAL { ?entity expression:exhibits|^expression:observedOn ?observable }
      }
      GROUP BY ?entity
    }
    OPTIONAL {
      ?entity expression:hasPart ?part .
    }
}
```

**Interpretation.** The question cannot be answered. The model has entities, which are things
to observe, and one level of parts under `Pipe`. It has no components, no interfaces between
them and no requirements allocated to them, so "can this be swapped as a module" cannot be
asked. This is a diagnosed gap in the vocabulary, and a deliberate one: the method describes
observable behavior and leaves the architecture open. Answering it needs the same
things the previous question was missing: a component concept, ports where flows enter and
leave each component, and connections between those ports. A component can be swapped as a
module when its replacement has the same ports. It also needs a link from each requirement to
the component that must meet it.

---

## Sunlight and capacity

### How does the reactor capacity react to changes in the sunlight?

The reactor is outside the system boundary, so its capacity is not in the model. The stand-in
used here is what the oven hands to the reactor: how much nitrogen it can deliver at the
minimum temperature R1 asks for.

**Evidence.** The script reads the design sunlight, the lens area and the requirement bounds
from the model, then sweeps the sunlight from 0 to 1500 W/m². Captured power is sunlight times
lens area. Deliverable nitrogen is captured power divided by the energy needed to heat one
kilogram to the R1 temperature. A query cannot do this: it has to multiply quantities across
units, repeat the calculation over a range and for several lens counts, and draw the curves.

```python
include('src/method/py/utils.py')
import math, re
import micropip
await micropip.install(['matplotlib'])
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt

# ---------- ASSUMPTIONS: values the model does not hold ----------
T_INLET = 298.15       # K, inlet nitrogen temperature (the inflow side is not modeled)
CP_N2   = 1040.0       # J/(kg K), heat capacity of nitrogen (a material property)
LOSSES  = 0.0          # best case: all captured power reaches the gas
TO_SI   = {'K': 1.0, 'kW': 1000.0, 'kg/h': 1.0 / 3600.0}   # the method's units have no conversion factor

def num(v):
    return float(re.search(r'-?\d+(\.\d+)?', str(v)).group())

# ---------- 1. QUERY ----------
async def quantity(subject, prop):
    """Read an OML quantity value in SI units."""
    result = await query(f"""
      PREFIX oml:        <http://opencaesar.io/oml#>
      PREFIX expression: <http://www.example.com/method/expression#>
      PREFIX system:     <http://www.example.com/project/system#>
      SELECT ?value ?multiplier WHERE {{
        system:{subject} expression:{prop} ?qty .
        ?qty oml:value ?value .
        OPTIONAL {{ ?qty oml:unit ?unit . ?unit oml:multiplier ?multiplier }}
      }}
    """)
    rows = result['rows']
    if not rows:   # fall back to a plain literal
        result = await query(f"""
          PREFIX expression: <http://www.example.com/method/expression#>
          PREFIX system:     <http://www.example.com/project/system#>
          SELECT ?value WHERE {{ system:{subject} expression:{prop} ?value }}
        """)
        rows = result['rows']
    row = rows[0]
    return num(row['value']) * (num(row['multiplier']) if row.get('multiplier') else 1.0)

sunlight  = await quantity('SolarIrradiance', 'irradianceLevel')   # W/m2
lens_area = await quantity('FresnelLens', 'apertureArea')          # m2

result = await query("""
  PREFIX expression:  <http://www.example.com/method/expression#>
  PREFIX requirement: <http://www.example.com/method/requirement#>
  SELECT ?requirement ?output ?operator ?value ?symbol WHERE {
    ?comparison a expression:Comparison ;
                expression:constrains ?output ;
                expression:comparisonOperator ?operator ;
                expression:hasThreshold ?threshold .
    ?threshold expression:thresholdValue ?value ;
               expression:hasUnit ?unit .
    ?unit expression:unitSymbol ?symbol .
    ?comparison expression:isSubconditionOf* ?root .
    ?requirement requirement:hasCondition ?root .
  }
""")

# ---------- 2. COMPUTE ----------
def bounds(output, operators):
    """(value in SI, requirement) for every bound of that kind on the output."""
    return [(num(r['value']) * TO_SI[str(r['symbol'])], frag(r['requirement']))
            for r in result['rows']
            if frag(r['output']) == output and str(r['operator']) in operators]

power_min = max(bounds('CapturedSolarPower', ('GE', 'GT')))          # W
power_max = min(bounds('CapturedSolarPower', ('LE', 'LT')))          # W
temp_min  = max(bounds('OutletNitrogenTemperature', ('GE', 'GT')))   # K
flow_min  = max(bounds('OutletNitrogenMassFlow', ('GE', 'GT')))      # kg/s

usable        = 1.0 - LOSSES
heat_per_kg   = CP_N2 * (temp_min[0] - T_INLET)                  # J/kg to reach the R1 temperature
one_lens      = sunlight * lens_area * usable                    # W at the design point
power_for_r1  = flow_min[0] * heat_per_kg                        # W the gas actually needs
flow_one_lens = one_lens / heat_per_kg * 3600.0                  # kg/h deliverable with one lens
lenses_needed = math.ceil(power_min[0] / one_lens)
with_needed   = lenses_needed * one_lens
in_band       = with_needed <= power_max[0]

met      = "met" if one_lens >= power_min[0] else "NOT met by one lens"
position = "inside" if in_band else "above"
display(
    f"<b>Design point:</b> {sunlight:.0f} W/m² on one lens of {lens_area:.1f} m² captures "
    f"<b>{one_lens / 1000:.2f} kW</b><br>"
    f"<b>{power_min[1]} asks for at least {power_min[0] / 1000:.1f} kW:</b> {met}<br>"
    f"<b>Lenses needed:</b> {lenses_needed}, giving {with_needed / 1000:.2f} kW "
    f"({position} the {power_max[1]} limit of {power_max[0] / 1000:.1f} kW)<br>"
    f"<b>Deliverable nitrogen with one lens:</b> {flow_one_lens:.1f} kg/h at {temp_min[0]:.2f} K; "
    f"{flow_min[1]} asks for {flow_min[0] * 3600:.1f} kg/h, which needs only {power_for_r1 / 1000:.2f} kW"
)

# ---------- 3. RENDER ----------
sweep = [g * 10.0 for g in range(0, 151)]    # 0 to 1500 W/m2
fig, (left, right) = plt.subplots(1, 2, figsize=(11, 4))

for n, color in zip((1, 2, 3), ('tomato', 'mediumseagreen', 'goldenrod')):
    label = "1 lens" if n == 1 else f"{n} lenses"
    left.plot(sweep, [g * lens_area * usable * n / 1000 for g in sweep], color=color, label=label)
left.axhspan(power_min[0] / 1000, power_max[0] / 1000, color='grey', alpha=0.2,
             label=f"required: {power_min[0] / 1000:.1f} to {power_max[0] / 1000:.1f} kW")
left.axvline(sunlight, color='grey', linestyle='--', linewidth=1)
left.set_xlabel('Sunlight (W/m²)')
left.set_ylabel('Captured power (kW)')
left.set_title('Captured power against the required band', fontsize=10)
left.legend(fontsize=8)

right.plot(sweep, [g * lens_area * usable / heat_per_kg * 3600 for g in sweep], color='tomato',
           label='1 lens')
right.axhline(flow_min[0] * 3600, color='grey', linewidth=1,
              label=f"{flow_min[1]} minimum: {flow_min[0] * 3600:.1f} kg/h")
right.axvline(sunlight, color='grey', linestyle='--', linewidth=1)
right.set_xlabel('Sunlight (W/m²)')
right.set_ylabel(f'Nitrogen deliverable at {temp_min[0]:.0f} K (kg/h)')
right.set_title('Capacity: hot nitrogen the oven can deliver', fontsize=10)
right.legend(fontsize=8)

plt.tight_layout()
display(image_html(fig))
```

**Interpretation.** Capacity grows in a straight line with sunlight. At the design point of
800 W/m², one lens captures 1.2 kW, which in the best case heats about 11 kg/h of nitrogen to
the R1 temperature. R1 asks for 1.8 kg/h, so one lens covers it.

The captured-power requirements say something different. R5 asks for at least 2.0 kW, and one
lens gives 1.2 kW, so **one lens is not enough: the system needs two.** Two lenses give 2.4 kW,
inside the 2.0 to 3.0 kW band of R7; three would give 3.6 kW and break its upper limit. The
requirements only hold together with exactly two.

The model cannot say that. `FresnelLens` is one entity with one aperture area. It is not a
component the system can have two of, so there is nowhere to write the number of lenses and
nothing that connects a second lens to the pipe. This is the same diagnosed gap in the
vocabulary as in the system structure section: the method has no component concept. `oml lint`,
`oml validate` and `oml reason` all pass on the model as it is; only this computation shows
that the lens as described cannot meet R5.

The result also rests on assumed values the model does not hold, listed at the top of the script. Each
one is a gap:

- **Inlet nitrogen temperature (298.15 K).** The inflow side is not modeled.
- **No losses.** Efficiency is not modeled, so every number here is a best case.
- **Unit conversions (kW to W, kg/h to kg/s).** The method's units carry no conversion factor.
- **Heat capacity of nitrogen.** A material property, outside what a requirements model should
  hold.

---

## One requirement in full

The computed analysis points at R5: one lens cannot meet it. The block below is the method's
`requirement-trace` template, called with R5. It is defined once in the method and reused: the
[Notebook](Notebook.md) calls the same template with R4.

```compose
template: http://www.example.com/method/requirement-trace
requirement: http://www.example.com/project/requirements#R5
```
