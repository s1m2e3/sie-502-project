# Solar Oven for Nitrogen Heating: An OML Model

An [OML](https://www.modelware.io/) model of a **solar oven that heats nitrogen for a pyrolysis
reactor**, built for SIE-502. It records the stakeholders, their requirements, what each
requirement means as logic, and how each one will be verified.

## What the system is

A Fresnel lens takes in sunlight and focuses it onto a bare section of pipe inside the oven.
Room-temperature nitrogen from a nitrogen generator runs through that pipe, heats up, and goes
out to a pyrolysis reactor. The rest of the pipe is insulated. The nitrogen keeps oxygen out of
the reactor.

**System boundary**

| Crosses the boundary | Direction | Modeled as |
| --- | --- | --- |
| Sunlight at the site (design point 800 W/m²) | in | `SolarIrradiance` (Input) |
| Room-temperature nitrogen from the generator | in | `NitrogenStream` (PhysicalEntity) |
| Operators switching the system between operation and maintenance | in | `OperationMode`, `MaintenanceMode` (Inputs on `SystemState`) |
| Hot, inert nitrogen to the pyrolysis reactor | out | `OutletNitrogenTemperature`, `OutletNitrogenMassFlow`, `OutletGasComposition` (Outputs) |

The model describes observable behavior only. It does not choose an architecture: the pipe
material, the dimensions and the thermal design are still open.

## What the model captures

| Layer | Contents |
| --- | --- |
| **Stakeholders** | Reactor managers, plant managers, maintenance operators, safety regulators |
| **Requirements** | R1 hot-nitrogen delivery (≥ 673.15 K, ≥ 1.8 kg/h) · R2 thermal safety (outlet ≤ 873.15 K, bare pipe ≤ 923.15 K) · R3 inert gas · R4 touch safety during maintenance (≤ 323.15 K) · R5 solar capture (≥ 2.0 kW) |
| **Conditions** | The logic behind each requirement: atomic comparisons with units, AND compounds (R1, R2), and a ForAll over the set of accessible surfaces (R4) |
| **Entities and observables** | The nitrogen stream, the pipe and its insulated and bare sections, the Fresnel lens (1.5 m²), the installation site, and the system state, each with the outputs it exhibits |
| **Verification** | Each requirement is verified by a testing strategy: thermal simulation (Analysis), design inspection (Inspection), or maintenance demonstration (Demonstration) |

## Business questions the model answers

These come from Project Deliverable 1:

| Question | How the model answers it |
| --- | --- |
| Are the safety requirements on the **inputs** traced to a verification strategy? | Follow `Requirement → hasCondition → hasInput`, then `Requirement → isVerifiedBy → TestingStrategy` |
| Are the safety requirements on the **outputs** traced to a verification strategy? | Follow `Condition → constrains → Output`, then `isVerifiedBy` (R2 and R4 are the safety requirements) |
| How do the components react to changes in the input? | *Partially.* The model states which outputs each input affects and what thresholds apply. The actual response needs a simulation (see below) |
| How does reactor capacity react to changes in sunlight? | *Partially.* R1 and R5 tie irradiance to the outlet temperature, the mass flow and the captured power. Working out the response curve needs a simulation |
| Can components be replaced as modules? | Not yet modeled. It needs an architecture layer (future deliverable) |

**About simulation.** Questions 3 and 4 ask how the system behaves, and a structural ontology
cannot compute that. What the model can do is **provide the inputs** for a simulation: the
irradiance, the lens aperture, the thresholds with their units, and which outputs to check.
A simulation run from a notebook could read those values and check the results against the
requirement conditions. That work is optional and outside the scope of the current
deliverables.

## How to read the repo

```
src/
├── method/oml/                      # vocabularies: the modeling language
│   ├── www.example.com/method/
│   │   ├── bundle.oml               # vocabulary bundle: closes the method so sibling concepts are disjoint
│   │   ├── base.oml                 # root aspects (Element, Expressible), description/expression properties
│   │   ├── expression.oml           # entities, inputs/outputs, and the condition hierarchy
│   │   ├── requirement.oml          # Stakeholder, Requirement, TestingStrategy and their relations
│   │   └── units.oml                # project units (W/m², kg/h, kW, m²)
│   └── opencaesar.io/               # imported SI / ISQ vocabularies
└── model/oml/                       # descriptions: the solar-oven system itself
    └── www.example.com/project/
        ├── bundle.oml               # description bundle: what `oml reason` checks against the closed method
        └── description.oml          # all instances: inputs, entities, outputs, stakeholders, R1-R5, conditions, strategies
```

A good reading order: `base.oml` → `expression.oml` → `requirement.oml` → `description.oml`.
In the description, start with the requirements R1–R5 and follow their links out to the
conditions, the outputs and the testing strategies.

## Build

Open this folder in VS Code with the **OML Code** extension, then run from the integrated
terminal:

```bash
oml lint                   # syntax and reference checks
oml reason                 # DL consistency and SHACL checks
oml export -o build/owl    # export to OWL
```

## Status and open issues

- **Bundles: done.** The vocabulary bundle closes `base`, `units`, `expression` and
  `requirement`. The reasoner therefore treats sibling concepts as disjoint: `Entity`,
  `Input`, `Output`, `Condition`, `Stakeholder`, `Requirement` and `TestingStrategy`
  cannot overlap, and neither can the condition subtypes. For example, a condition that
  `constrains` an Input now makes the model inconsistent. `oml lint` and `oml reason` both pass.
- **Design consequence:** `Input` and `Output` are now disjoint. An intermediate variable that
  is both (such as the bare-pipe temperature: an output of the lens and an input to heating
  the gas) would need a shared subconcept, for example `InternalVariable < Input, Output`.
- **Defined concept (Week 3 requirement): not yet added.** The candidate is a thermal-safety
  requirement defined by its condition's temperature comparison, so the reasoner classifies it.
- The architecture (pipe material, modular components) is intentionally left out so far.
- Checking behavior over the sunlight range needs a simulation. See *About simulation* above.
