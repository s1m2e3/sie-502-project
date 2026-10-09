# Analysis: Solar Oven Requirements

This file records what the analysis layer found: each question, the evidence, and the finding.
The live views are in the [Dashboard](src/model/md/Solar%20Oven/Dashboard.md); the results are
written out here so they can be read without running it. [METHOD.md](METHOD.md) explains the
method the analysis runs over.

Every question ends in an answer or in a diagnosed gap. A gap names its source: the vocabulary,
a pattern, the model's content, or something outside the model.

## The five project questions

These are the questions from Project Deliverable 1.

| # | Question | Evidence | Finding |
| --- | --- | --- | --- |
| 1 | Are the safety requirements on the inputs traced to a verification strategy? | Table of safety requirements with their inputs and strategies; matrix Input x Safety requirement | **Answered: yes.** R2, R4 and R6 are each verified. **Gap (model content):** no safety requirement mentions sunlight |
| 2 | Are the safety requirements on the outputs traced to a verification strategy? | Matrix Output x Safety requirement | **Answered: yes for the four temperature outputs.** **Gap (pattern):** R3 protects the reactor from oxygen but is not counted as safety |
| 3 | How do the components react to changes in the input? | Graph: inputs, entities, outputs | **Diagnosed gap (vocabulary):** the method has no components, no ports, and no link from an input to what receives it |
| 4 | How does the reactor capacity react to changes in the sunlight? | Python block: sunlight sweep | **Answered with a stand-in:** capacity grows linearly with sunlight. **Finding:** one lens cannot meet R5; the system needs two. **Gap (vocabulary):** the model cannot say "two lenses" |
| 5 | Does the architecture allow modular replacement of the components? | Tree of entities and parts | **Diagnosed gap (vocabulary):** there is no architecture layer |

### 1. Safety requirements on the inputs

A safety requirement is one raised by a Regulator. The query follows each one to the leaves of
its condition tree and collects the inputs they mention.

| Requirement | Inputs | Verified by | Method |
| --- | --- | --- | --- |
| R2 | OperationMode | TS_ThermalSimulation | Analysis |
| R4 | MaintenanceMode | TS_MaintenanceDemonstration | Demonstration |
| R6 | OperationMode | TS_InsulationInspection | Inspection |

All three are traced. In the matrix, the `SolarIrradiance` row is all zeros: no safety
requirement mentions sunlight, although sunlight is what heats the pipe. R2 must hold at any
irradiance, and its thermal simulation has no stated sunlight range to cover. The pattern
already lets a condition name its inputs, so this gap is in the model's content.

### 2. Safety requirements on the outputs

| Output | R2 | R4 | R6 |
| --- | --- | --- | --- |
| AccessibleSurfaceTemperatures | 0 | 1 | 0 |
| BarePipeSurfaceTemperature | 1 | 1 | 0 |
| InsulatedPipeSurfaceTemperature | 0 | 1 | 1 |
| OutletNitrogenTemperature | 1 | 0 | 0 |
| CapturedSolarPower | 0 | 0 | 0 |
| OutletGasComposition | 0 | 0 | 0 |
| OutletNitrogenMassFlow | 0 | 0 | 0 |

The four temperature outputs are bounded by a verified safety requirement. `OutletGasComposition`
is the zero row that matters. R3 keeps oxygen out of the pyrolysis reactor, which is a safety
concern, but only the reactor managers raised it. The method decides "safety" by who raised a
requirement, not by what it protects. That is a gap in the pattern.

### 3. How the components react to the inputs

The graph has three columns: inputs, entities, outputs. An input is joined to an entity only
when a condition mentions that input and bounds one of the entity's outputs.

- The model has no components. The middle column is entities, and it mixes parts of the system
  (the lens, the pipe sections) with `NitrogenStream`, which is the flow crossing the boundary.
- Only the outflow side of the nitrogen is modeled: all three of its attributes are outlet
  values.
- Every input-to-entity link is derived from the requirements. The system description never
  says an entity receives an input.
- Sunlight reaches the lens and the nitrogen stream, but not the bare pipe section it heats.

**Diagnosed gap (vocabulary).** To answer this, the method needs:

1. a component concept, separate from the flows that pass through it;
2. ports: the place on a component where a flow enters or leaves;
3. a way to say that a component receives an input;
4. inlet as well as outlet attributes on a flow.

The vocabulary also makes entity, input and output mutually exclusive, so the nitrogen stream
cannot be declared as an input on one side and an output on the other. How large the reaction
is cannot come from a structural model at all; that part is outside the model and needs a
simulation.

### 4. Reactor capacity against sunlight

The reactor is outside the system boundary, so its capacity is not in the model. The stand-in
is how much nitrogen the oven can deliver at the R1 minimum temperature. The script reads the
design sunlight (800 W/m2), the lens area (1.5 m2) and the requirement bounds from the model,
then sweeps the sunlight from 0 to 1500 W/m2.

| Result at the design point | Value |
| --- | --- |
| Power captured by one lens | 1.2 kW |
| Nitrogen one lens can heat to 673.15 K | about 11 kg/h |
| R1 asks for | 1.8 kg/h, which needs about 0.2 kW: **met** |
| R5 asks for | at least 2.0 kW: **not met by one lens** |
| Lenses needed for R5 | 2, giving 2.4 kW |
| R7 allows | 2.0 to 3.0 kW: two lenses fit, three (3.6 kW) do not |

Capacity grows in a straight line with sunlight, and one lens is enough for R1. The
captured-power requirements only hold with exactly two lenses. The model describes one lens,
as one entity with one aperture area, and has nowhere to state how many there are.
`oml lint`, `oml validate` and `oml reason` all pass on the model; only this computation shows
that the lens as described cannot meet R5.

**Diagnosed gap (vocabulary):** the same missing component concept as in question 3. A lens
has to be a component the system can have two of.

The script also uses four values the model does not hold. Each is a gap:

| Value | Source of the gap |
| --- | --- |
| Inlet nitrogen temperature (298.15 K) | Vocabulary: the inflow side is not modeled |
| No losses (best case) | Vocabulary: efficiency is not modeled |
| kW to W, kg/h to kg/s | Pattern: the method's units carry no conversion factor |
| Heat capacity of nitrogen | Outside the model: a material property |

### 5. Modular replacement

The tree shows everything the model knows about structure: seven entities, with one level of
parts under `Pipe`. There are no components, no ports and no connections, and requirements are
not allocated to anything. A component can be swapped as a module when its replacement has the
same ports, so without ports the question cannot be asked.

**Diagnosed gap (vocabulary).** This one is deliberate: the method describes observable
behavior and leaves the architecture open. It needs the same extension as question 3, plus a
link from each requirement to the component that must meet it.

## Gap detection

One check per pattern. Each lists every element of its kind and marks it as present or as a
gap, so a clean result is visible and not just an empty table.

| Pattern | Check | Result |
| --- | --- | --- |
| Stakeholder | Raises at least one requirement | 7 of 7 pass |
| Testing strategy | Verifies at least one requirement | 5 of 5 pass |
| Requirement | Has one condition, a stakeholder and a strategy | 7 of 7 pass |
| Condition | Every condition tree belongs to a requirement | 7 of 7 roots pass |
| Condition | Every threshold is used by a comparison | 8 of 8 pass |
| Quantities and units | Every unit is used by a threshold | 3 of 3 pass |
| System | Every input and output is mentioned by a condition | 10 of 10 pass |
| System | Every entity has something observed on it | 6 of 7 pass; `Pipe` is a gap |

`Pipe` has nothing observed on it: it only groups its insulated and bare sections, and the
temperatures are observed on those. It is a container by design. The first two checks span two
files, so no editor can enforce them; a query is the only place they can be checked.

## Diagnosed gaps by source

| Source | Gap | What would close it |
| --- | --- | --- |
| Vocabulary | No components, ports or connections | Extend the vocabulary with them (questions 3, 4, 5) |
| Vocabulary | A flow has outlet attributes only; entity, input and output are exclusive | Model inlet attributes; let a flow be an input and an output |
| Vocabulary | No count of a component, no efficiency | Comes with the component concept |
| Pattern | "Safety" follows who raised a requirement | Let a requirement be marked safety by what it protects |
| Pattern | Units have no conversion factor | Add a multiplier to the unit pattern |
| Pattern | The system description (inputs, entities, outputs) has no editor | Add a pattern and editor for it |
| Model content | No safety requirement names sunlight | Add `SolarIrradiance` as an input of R2's conditions |
| Model content | Deliverable 1 lists the nitrogen generator managers; they are not in the model | They raise no requirement yet; add one for the inlet nitrogen or leave them out |
| Outside the model | Size of the response to an input; material properties | Simulation fed by the model's values |

None of these was fixed in this deliverable. The purpose here is to find and locate them; the
vocabulary gaps are one extension (components and ports) that belongs in the next iteration of
the method.

## What the analysis layer contains

| Piece | Where |
| --- | --- |
| Dashboard: narrative with `table`, `matrix`, `graph` and `tree` views | `src/model/md/Solar Oven/Dashboard.md` |
| Computed analysis (python: query, compute, render) | Dashboard, "Sunlight and capacity" |
| View template `requirement-trace` (`compose`, one `requirement` parameter) | `src/method/md/www.example.com/method/requirement-trace.md` |
| Template reuse | Dashboard (R5) and Notebook (R4) |
| Script helpers | `src/method/py/utils.py` |
