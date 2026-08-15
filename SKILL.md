---
name: abaqus-skill
description: Evidence-driven Abaqus/CAE mesh replacement and coupled thermal-stress diagnosis. Use an existing CAE as the authoritative master, import a changed orphan-mesh INP, preserve model/part/instance and analysis-object names, add or remove deposition layers with terminal-step inheritance, rebuild and validate sets/surfaces, maintain mechanical versus thermal element families, audit generated INPs and ODB temperature mappings, diagnose Abaqus/Standard convergence failures, or deliver a verified replacement CAE.
---

# Abaqus CAE mesh update

Treat the original `.cae` as authoritative for analysis intent and the new `.inp` as authoritative only for nodes, elements, connectivity, component membership, and element topology.

## Read supporting resources selectively

- Read [references/mesh-replacement-patterns.md](references/mesh-replacement-patterns.md) before changing a CAE part, instance, layer count, picked region, or surface.
- Read [references/coupled-validation.md](references/coupled-validation.md) when the model imports temperatures from an ODB or when diagnosing coupled thermal-stress convergence.
- Run `scripts/inspect_inp_mesh.py` before Abaqus editing to inventory a raw INP, component element sets, element types, coordinate bounds, and candidate layer bands.
- Run `scripts/audit_generated_inp.py` after generating validation INPs to detect missing layers, unexpected importer names, critical load or mapping errors, and terminal-step mistakes.

## Input contract

Inventory the supplied directory before editing.

Required:

- Master `.cae` with the authoritative model objects.
- New mesh `.inp`.
- Requested output directory and filename.
- An Abaqus version able to open the master.

Strongly recommended:

- Master `.jnl`, `.rpy`, latest generated job `.inp`, and session `.log/.rec`.
- Successful `.odb/.sta/.msg/.dat` files for behavioral validation.
- For ODB-driven stress models, the thermal ODB and the intended `BSTEP/BINC/ESTEP/EINC` mapping.

Inspect what is present before requesting anything. Missing journal/replay files weaken reconstruction evidence but do not block inspection of a readable CAE. Missing master CAE or new mesh INP blocks replacement.

## Non-destructive working rule

1. Make a byte-for-byte working copy of the master before opening it in Abaqus.
2. Open only the working copy during transformation and version conversion.
3. Save the result under the requested new name.
4. Compare the master hash before and after when exact source preservation matters.

Opening or converting a CAE can change file metadata. Never use the only master as the active working database.

## Core workflow

1. Inventory every model, part, instance, material, section, set, surface, step, load, predefined field, interaction, constraint, BC, output request, amplitude, job, and propagation/deactivation state.
2. Generate temporary master job INPs when useful; use them as keyword snapshots, not as substitutes for the master CAE.
3. Parse the new INP. Compare node/element counts, labels, connectivity, coordinate bounds, element types, component membership, layer bands, and exterior topology.
4. Stop for confirmation when the new mesh changes the number of physical activation layers. Adding or deleting a layer changes steps, loads, model changes, ODB mappings, and terminal states; it is not a routine mesh-only edit.
5. Import the new orphan mesh into temporary Abaqus models. Preserve the analysis-specific element family in each destination model; for example, use `C3D8R` for stress and `DC3D8R` for heat transfer when that is the master's pairing.
6. Prefer part/instance replacement inside copied master models:
   - rename the old part and instance temporarily;
   - copy in the new mesh part with the exact original part name;
   - create the exact original instance name;
   - rebuild same-named assembly sets and surfaces on the new instance;
   - rebind or recreate affected objects;
   - delete the old instance and part only after all replacement regions exist.
7. Preserve surviving object names and ordering. Do not accept importer-generated replacements such as `BODYFLUX-1`, `Field-1`, `Model_Change-1`, split BCs, or one output object per step.
8. Assign the original section to the complete replacement mesh and verify every element is assigned.
9. Apply layer-count inheritance rules, then rebuild every affected set, picked set, surface, load, BC, interaction, predefined field, and mapping.
10. Remove all temporary import models and Job objects from the delivered CAE unless the user explicitly requests jobs.
11. Save to a scratch CAE first. Reopen it in Abaqus noGUI mode, create temporary validation jobs without saving them, generate both thermal and stress INPs, and run keyword audits.
12. Write the final CAE only after every validation gate passes.

## Mandatory preservation rules

### Materials and sections

- Preserve every populated material subproperty, table, temperature dependency, field dependency, and option recursively.
- Check Elastic, Plastic and its suboptions, Density, Conductivity, Specific Heat, Expansion, Latent Heat, and any other populated category.
- Verify section names, material links, assignment counts, and complete new-mesh coverage.

### Layer-count changes and terminal inheritance

Treat deposition/cooling pairs as stateful sequences.

- For layer deletion, move the old final-layer behavior to the new final layer.
- For layer addition, convert the old terminal cooling step into an ordinary cooling step and create a new deposition/cooling pair. Make the new final cooling step inherit the old terminal controls and terminal states.
- Move final BC/load/interaction deactivation from the old terminal step to the new terminal step.
- Extend Model Change, layer heat flux, predefined temperatures, and outputs consistently.
- Never infer the required sequence from element numbering alone; confirm physical layer intent with geometry and the user.

Example for old `l1/c1 ... l22/c22` and new 23-layer mesh:

- change old `c22` from terminal cooling to ordinary cooling using the preceding regular-layer pattern;
- add `l23/c23`;
- make `c23` inherit old `c22` terminal time and increment controls;
- add the `l23` flux and activation object;
- move final BC deactivation to `c23`;
- extend ODB mappings to thermal steps 47 and 48, then verify actual endpoint increments after the thermal run.

### Regions and internal picked objects

- Recreate same-named user sets first so existing objects can resolve by name.
- Rebuild picked sets and picked surfaces on the new instance; a copied region tuple is not evidence of valid membership.
- When replacing internal picked objects, create them as ordinary named regions, bind/recreate the dependent objects, delete the old instance, then mark the regions internal. Marking them internal too early can cause lookup or feature-deletion failures.
- For orphan-mesh surfaces, use Abaqus `face1Elements` through `face6Elements`, not `sideNElements`.
- Derive exterior faces from connectivity ownership and validate orientation, component, coordinate band, and nonempty membership.

### Outputs and BCs

- Preserve output request count, names, variables, frequencies, regions, and propagation. Keep one propagated `F-Output-1` when that is the master structure.
- Preserve BC granularity and DOF combinations. Recreate one same-named BC when its region or terminal state cannot be edited safely; do not split U1-U3 into separate objects.
- In Abaqus releases where output repositories are not exposed as expected, verify output structure from generated INPs.

### Coupled temperature fields

- Preserve one predefined-temperature object's propagation organization when that is the master structure.
- Use `setValuesInStep` for new or changed FROM_FILE mappings when supported.
- Treat inherited increment numbers as provisional until the regenerated thermal ODB exists. Resolve the actual last converged increment for every referenced step before stress execution.
- Report an absent thermal ODB as an execution limitation, even when the CAE structure itself is valid.

## Abaqus API lessons

- Create a temporary `Job` and call `writeInput`; a `Model` does not provide `writeInput` in common Abaqus/CAE releases.
- Do not save validation Job objects into the delivered CAE.
- Use the actual `HeatTransferStep` signature; do not pass mechanical-only arguments such as `nlgeom` when unsupported.
- Do not assume BC objects provide `resetToPropagated`. Recreate the exact BC and deactivate it in the correct new terminal step when needed.
- Expect repository visibility to change after `markSetInternal` or `markSurfaceInternal`; validate via `allInternalSets/allInternalSurfaces` and generated input.

## Validation gates

Require all of the following:

- Exact surviving model, part, instance, material, section, step, set, surface, load, field, interaction, BC, and output names.
- Correct new node/element counts and analysis-specific element types.
- Every layer set nonempty, mutually disjoint, and collectively equal to the deposition component.
- Substrate and deposition sets complete and non-overlapping as intended.
- Every surface-based object resolves to a nonempty surface on the new instance.
- Every set-based load, BC, Model Change, constraint, and predefined field resolves to a nonempty new region.
- Critical magnitudes, step controls, activation/deactivation states, and output variables match the accepted inheritance rule.
- Thermal ODB step/frame mappings are explicitly listed and validated against the actual ODB when available.
- Scratch CAE reopens; thermal and stress validation INPs generate successfully with consistency checking.
- Generated INPs contain no unexplained importer names and show the intended terminal-step behavior.
- Final CAE contains no temporary models, old parts/instances, scratch regions, or Job objects.

If a required setting cannot be mapped with evidence, stop and report the discrepancy instead of delivering a plausible-looking CAE.

## Coupled convergence diagnosis

Treat `Too many attempts made for this increment` as a symptom. Read the failed `.msg`, `.sta`, `.dat`, generated INP, and converged ODB frames before changing controls. Compare successful and failed generated INPs, imported thermal fields, and the inherited `U/S/LE/PEEQ` state. Read [references/coupled-validation.md](references/coupled-validation.md) for the full controlled cross-test workflow.

## Deliverables

Provide:

- final replacement `.cae` at the agreed path;
- concise validation report with sources, Abaqus version, counts, preserved/changed objects, region checks, terminal inheritance, ODB mappings, warnings, and unresolved limitations;
- reproducible Abaqus Python/configuration artifacts when the transformation is nontrivial or likely to recur.

Keep scratch CAEs, mapped INPs, temporary ODBs, validation jobs, lock files, and diagnostic logs separate from the final deliverables. Do not submit a full analysis unless requested or necessary and authorized.
