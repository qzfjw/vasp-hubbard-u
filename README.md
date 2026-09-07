# vasp-hubbard-u

`vasp-hubbard-u` is a Codex skill for VASP linear-response Hubbard-U calculations.
It focuses on a single representative correlated site and keeps every perturbation
point isolated from the others so the final `U` remains auditable.

## Algorithm

The default workflow follows the VASP `LDAUTYPE=3` linear-response tutorial:

1. Relax or validate the structure with plain PBE.
2. Split one representative correlated atom into its own species group.
3. Run a static plain-PBE Reference with `LDAU = .FALSE.` and `LDAUPRINT = 2`.
4. Apply a symmetric local potential grid to the representative group only.
5. Run screened response points with `ICHARG = 1`.
6. Run bare response points with `ICHARG = 11`.
7. Extract the final projected `d` or `f` occupation from `OUTCAR`.
8. Fit the response slopes and compute:

```text
U = 1/chi - 1/chi0
```

The skill treats the projected occupation as a PAW-projected observable, not an
absolute orbital population. It also keeps the Reference, screened, and bare
branches on the same input contract for fair comparison.

## Installation

### From a local copy

Copy the folder into your Codex skills directory:

```text
C:\Users\Leo\.codex\skills\vasp-hubbard-u
```

### From GitHub

If you already have the repository cloned, place the folder contents into
`$CODEX_HOME/skills/vasp-hubbard-u` and restart Codex so the skill is discovered.

## Usage

Invoke it explicitly:

```text
Use $vasp-hubbard-u to prepare and analyze a VASP linear-response Hubbard-U calculation.
```

The skill is designed for single-representative-site calculations. If you need
multiple inequivalent sites or a full response matrix, treat those as separate
workflow extensions rather than assuming the scalar result covers them.
