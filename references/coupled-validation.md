# Coupled thermal-stress validation

## Contents

1. Temperature mapping audit
2. Convergence diagnosis
3. Controlled cross-test
4. Reporting boundaries

## 1. Temperature mapping audit

For each stress step that imports temperatures:

1. Record the external ODB path.
2. Record `BSTEP/BINC/ESTEP/EINC` from the generated stress INP.
3. Open the actual thermal ODB and resolve the referenced step and frame.
4. Confirm the increment exists and is the intended endpoint.
5. Repeat after every thermal rerun because automatic increments can change.

Do not equate output-frame count with solver increment count. Saved frames do not themselves alter the solution, but a changed thermal increment path or a different referenced endpoint can change the imported field.

When a new thermal layer is added, CAE structure can be prepared before the ODB exists, but endpoint increments are provisional. State this limitation in the validation report.

## 2. Convergence diagnosis

Treat `Too many attempts made for this increment` as a symptom.

Read:

- failed job `.inp`, `.msg`, `.sta`, `.dat`, and last converged `.odb`;
- successful comparison equivalents when available;
- both thermal and stress CAEs and external ODBs;
- Abaqus version, precision, CPU/parallel settings, and restart status.

Compare generated INPs, not only CAE dialogs. Check:

- mesh counts and element types;
- material tables and dependencies;
- step sequence and increment controls;
- Model Change regions and activation order;
- BCs, contacts, ties, loads, outputs, and constraints;
- all `*TEMPERATURE, FILE=...` mappings.

At selected thermal step endpoints and at the start of the failing stress step, compare nodal temperature fields. Report average and maximum absolute differences and nodes with the largest differences.

Compare the inherited mechanical state immediately before failure: `U`, `S`, `LE`, and `PEEQ`. Temperature-dependent plasticity and element activation are path-dependent; equal later temperatures do not erase different residual states.

Inspect the failing increment for cutbacks, negative eigenvalues, excessive plastic strain increments, distorted elements, singularities, contact/constraint changes, and newly activated regions.

## 3. Controlled cross-test

Use a shared copied thermal ODB to isolate the cause:

1. Keep both stress models unchanged.
2. Point every predefined-temperature state in both models to the same ODB step and actual frame/increment.
3. Use the same Abaqus version, precision, parallel settings, and fresh non-restart jobs.
4. Write both INPs and compare analysis-relevant keywords before submission.
5. Interpret:
   - same stress history/outcome: stress models are functionally equivalent; investigate original thermal ODB or mapping;
   - one succeeds and one fails: find a remaining stress-input difference;
   - both fail or failure moves: thermal history contributes, but also inspect instability, material extrapolation, regions, and activation.

## 4. Reporting boundaries

Do not claim that the number of thermal increments alone caused a stress failure. State the evidence chain:

1. changed increment controls may change the converged temperature path or selected endpoint;
2. imported temperature history changes accumulated plasticity, deformation, and residual stress;
3. a model near a nonlinear stability boundary can then converge in one run and fail in another.

Do not claim executable ODB mapping validation when the referenced ODB is missing. Distinguish a structurally valid CAE from a fully execution-validated coupled workflow.
