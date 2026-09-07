---
name: vasp-hubbard-u
description: Prepare, run, monitor, and analyze VASP linear-response Hubbard-U calculations with fixed inputs, isolated reference restarts, and auditable fit diagnostics. Use for single-representative-site U calculations from a POSCAR; extend the workflow explicitly for multiple sites or response matrices.
metadata:
  short-description: Auditable VASP linear-response Hubbard U
---

# VASP Linear-Response Hubbard U

Use this skill for a reproducible, evidence-backed calculation of a Hubbard U by the VASP linear-response method. The default mode is the official `LDAUTYPE=3` workflow for one representative correlated site:

1. obtain or validate a plain-PBE relaxed structure;
2. split one representative atom into its own species group;
3. calculate one converged plain-PBE Reference;
4. apply a symmetric local potential grid to that group;
5. calculate screened and fixed-charge responses;
6. fit the same PAW-projected occupation observable in both branches;
7. report `U = 1/chi - 1/chi0` together with numerical and physical diagnostics.

This skill is a workflow for calculating a response-based parameter, not a promise that every material has one transferable scalar U.

## Scope and modes

The default mode is `single-representative-site`. Use it only when the selected atom represents a meaningful symmetry or chemical-equivalence class. Defects, surfaces, interfaces, low-symmetry structures, inequivalent magnetic sites, and charge/orbital order may require separate representative sites.

Do not silently turn the default calculation into any of the following:

- a collective whole-element perturbation;
- an average over unrelated atoms;
- a full response matrix;
- an SOC or noncollinear workflow;
- a self-consistent U iteration;
- a DFT+U validation calculation.

If the user asks for several sites, use a separate isolated representative-site calculation for each site unless they explicitly request a response matrix. Treat `multi-representative-site` and `response-matrix` as future extensions with their own input and validation contracts; do not claim that the single-site result covers them.

## Authority, permissions, and evidence

Follow the current official VASP documentation and the user's explicit scientific choices. A user request for a U calculation authorizes preparation and analysis; submission, monitoring, and remote file operations remain bounded by the user's request and the configured server workflow.

Never cancel, delete, overwrite, or silently resubmit a job. Preserve failed evidence. A retry must use a clearly separate directory, follow a concrete diagnosis, and be authorized when the retry changes external state.

For any VASP, MPI, scheduler, or input failure, inspect the complete actual artifacts before suggesting a remedy:

- scheduler state, exit code, and job log;
- `INCAR`, `POSCAR`, `KPOINTS`, `POTCAR` metadata, and submission script;
- complete `OUTCAR` and `OSZICAR`;
- `CHGCAR` and `WAVECAR` status when required.

Distinguish a cluster-runtime failure from a VASP-input or physical-convergence failure. Never invent missing occupations, convergence, or fit points.

Do not print `POTCAR` contents. Record metadata only.

## Immutable calculation contract

Before preparing response points, record the calculation contract. Keep it identical across Reference, screened, and bare branches except for the tags that define the branch and the perturbation value:

- structure and coordinate order;
- PAW family, release, `TITEL`, `ZVAL`, `ENMAX`, element/group order, and preferably a file hash;
- `ENCUT`, `KPOINTS`, smearing, `EDIFF`, and any cutoff or precision settings;
- `ISPIN`, `MAGMOM`, `SAXIS`, SOC/noncollinear choices, and symmetry settings;
- target channel (`d` or `f`), representative atom index, and species-group mapping;
- perturbation grid, units, and sign convention;
- scheduler resources and VASP executable.

Do not infer a PAW variant from a prior failed calculation. If no project-specific PAW choice exists, use the standard non-suffixed PAW-PBE potentials by default. Treat `_pv`, `_sv`, and other variants as explicit scientific choices and record the reason. Once the Reference starts, use exactly the same PAW blocks in every stage.

For a split-site group, repeat the exact selected PAW block for the representative atom and the remaining atoms of that element. Never mix PAW variants between those two groups.

## Directory layout

Use a minimal, auditable root layout:

```text
<task>/
  01_relax/
  02_reference/
  03_screened/
    V_-0.20/
    V_-0.15/
    ...
  04_bare/
    V_-0.20/
    V_-0.15/
    ...
  <formula>_U_final_report.docx
```

Use ASCII directory names where possible. Keep each perturbation point in its own directory. Do not let one point read or overwrite another point's output.

## Workflow

### 1. Relax or validate the structure

Run a plain-PBE relaxation unless the user supplies a compatible completed relaxation. Validate the accepted `CONTCAR`:

- readable and nonempty;
- expected composition and atom count;
- coordinate order preserved;
- electronic convergence achieved;
- ionic convergence achieved when relaxation was requested;
- no fatal VASP or MPI error.

Do not proceed from a merely completed scheduler job if VASP did not converge or did not produce a valid `CONTCAR`.

### 2. Build the split-site Reference

Starting from the accepted relaxed `CONTCAR`, make the selected representative atom its own species group. Keep the remaining atoms of that element in a second group and retain all other groups. Preserve atom order.

For example, `M1 | Mrest | A | B` has species groups `M M A B`, so all species-indexed LDAU arrays have four entries:

```text
LDAUL = 2  -1  -1  -1   # d target
```

Use `3` for an f target. Set `-1` for every non-target group, including `Mrest`.

Run a converged plain-PBE static Reference with the user's approved settings plus the output needed for the response:

```text
LDAU      = .FALSE.
LORBIT    = 11
LDAUPRINT = 2
LMAXMIX   = 4            # 6 for an f target
LWAVE     = .TRUE.
LCHARG    = .TRUE.
```

The Reference is accepted only when the full output shows normal VASP completion, electronic convergence, a readable final `total charge` table for the representative atom, and nonempty usable `CHGCAR` and `WAVECAR` files. Record the Reference occupation from the same target `d` or `f` column used later; do not substitute a magnetization column.

### 3. Define and apply the perturbation

The default potential grid is:

```text
-0.20, -0.15, -0.10, -0.05, +0.05, +0.10, +0.15, +0.20 eV
```

Accept a user-specified symmetric grid when it spans zero and contains at least three distinct accepted points. A separate zero-potential run is unnecessary by default because the Reference supplies the zero point. If the user requests a zero run, treat it as a diagnostic and do not replace the accepted Reference without justification.

For every perturbation point, use `LDAUTYPE=3` and perturb only the representative group. For a d target in `M1 | Mrest | A | B`:

```text
LDAU     = .TRUE.
LDAUTYPE = 3
LDAUL    = 2  -1  -1  -1
LDAUU    = V   0   0   0
LDAUJ    = V   0   0   0
```

For an f target, use `3` in the first `LDAUL` entry. With `LDAUTYPE=3`, equal `LDAUU` and `LDAUJ` values are the spin-up and spin-down external potential convention; they are not the final Hubbard U and Hund J. Use the signed `V` exactly as defined. Do not reverse signs to force a positive result.

### 4. Isolate every restart

Every perturbation point must start from an immutable copy of the accepted Reference files. Never chain one potential point from the previous point.

For each `V`, create independent copies or links according to the server policy:

- screened point: Reference `CHGCAR` and `WAVECAR` as starting data;
- bare point: the same accepted Reference `CHGCAR` and `WAVECAR`;
- point-specific output: written only inside that point's directory.

Do not submit response points before the Reference passes. Do not reuse a response point's `CHGCAR` or `WAVECAR` as the starting point for another point. Record the source file identity or hash when practical.

### 5. Screened response

The screened response is self-consistent. Keep all non-perturbation settings identical to the Reference and use the appropriate restart settings:

```text
ISTART = 1
ICHARG = 1
LDAU   = .TRUE.
LDAUTYPE = 3
```

Use convergence settings appropriate to the material. Do not force `NELM=1` merely to create a bare response; a fixed-charge calculation should still produce the required output and a well-defined final projected occupation.

### 6. Bare response

The bare response uses the accepted Reference charge and wavefunction data while holding the charge density fixed:

```text
ISTART = 1
ICHARG = 11
LDAU   = .TRUE.
LDAUTYPE = 3
```

Leave `NELM` absent unless the user explicitly requests it or the selected VASP version and workflow require a documented setting. Keep all non-perturbation settings identical to the screened branch. Confirm that `ICHARG=11` actually uses the intended Reference `CHGCAR` and that the resulting `OUTCAR` contains the target projection.

## Stage gates and monitoring

After the first submitted job, create or update a 10-minute monitor unless the user asks to pause monitoring. At every gate, inspect both scheduler state and complete actual output. While a stage is queued or running, do not claim success or submit downstream stages.

A stage is accepted only if all relevant checks pass:

- scheduler reports `COMPLETED` with exit code `0:0`;
- VASP has a normal timing/footer termination;
- required electronic and ionic convergence is present;
- no fatal VASP, MPI, filesystem, or scheduler error is present;
- required files are readable and nonempty;
- the target `total charge` projection is present and unambiguous.

A failed or ambiguous stage stops downstream progression. Report the reason and preserve evidence; do not automatically retry.

## Extraction and fitting

Read the complete `OUTCAR` for every accepted point. Extract the representative atom's final `total charge` target column:

- `d` column for a d target;
- `f` column for an f target.

Use this identical observable for Reference, screened, and bare data. Label it explicitly as a VASP PAW-projected occupation, not as an absolute orbital population.

Fit:

```text
N(V)  = b  + chi  * V       # screened
N0(V) = b0 + chi0 * V       # bare
U = 1/chi - 1/chi0
```

Keep units consistent. If occupation is dimensionless and `V` is in eV, `chi` and `chi0` are in `1/eV`, and `U` is in eV.

For each branch, report the complete point table, slope and intercept, slope uncertainty, R², residuals, and any excluded point with a reason. Diagnose:

- nonlinearity or systematic residual curvature;
- asymmetry between positive and negative perturbations;
- magnetic-moment or electronic-state jumps;
- near-zero or sign-inconsistent slopes;
- sensitivity of U to one-point exclusion;
- insufficient accepted points or inadequate span around zero.

Do not call a weak, discontinuous, or strongly nonlinear fit a final usable U. Do not average inequivalent sites. Do not change the formula or signs to make U positive. If a diagnostic is needed, propose one physically motivated isolated sensitivity check and obtain authorization before running it.

## Reporting and completion

Do not perform DFT+U validation unless explicitly requested. When all authorized stages pass, create one root-level final report containing:

- system formula, target channel, representative atom and site-selection rationale;
- magnetic, SOC/noncollinear, symmetry, smearing, cutoff, and k-point assumptions;
- PAW metadata, group order, and consistency record;
- stage directories, job IDs, scheduler evidence, and acceptance gates;
- Reference, screened, and bare occupation data with exact `OUTCAR` anchors;
- fit parameters, uncertainties, R², residuals, exclusions, and units;
- the formula `U = 1/chi - 1/chi0`, the resulting U, and uncertainty or sensitivity information;
- retries, if any, with separate paths and diagnoses;
- limitations, especially the scalar single-site and zero-off-diagonal assumptions.

Keep required VASP evidence in its stage directories. In the final response state the current status, server, system, representative site, accepted and excluded jobs, `chi`, `chi0`, U with units, evidence path, and unresolved limitations.

## Extension contract

Future modes should preserve the same immutable input contract, stage gates, evidence rules, and observable definitions. Add a mode only when its perturbation scheme, restart semantics, response extraction, fitting model, and report format are explicitly defined:

- `multi-representative-site`: independent scalar calculations for chemically or symmetrically distinct sites;
- `response-matrix`: coupled site perturbations with explicit off-diagonal response extraction and matrix inversion;
- `method-comparison`: controlled comparison of PAW families or perturbation grids, never a silent method change.
