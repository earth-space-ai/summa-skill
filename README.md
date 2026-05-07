# SUMMA Skill

A progressive-disclosure skill for [SUMMA](https://github.com/NCAR/summa)
(Structure for Unifying Multiple Modeling Alternatives), the process-based
hydrologic modeling framework from NCAR / Univ. of Saskatchewan / Univ. of
Washington.

> **Skill author:** Koutian Wu (ktwu01@gmail.com)
> **Skill version:** 0.1.0

## What this is

A self-contained knowledge package that teaches AI agents (and humans) how to
**install, compile, run, configure, post-process, debug, and contribute to**
SUMMA. It captures the procedural knowledge that is normally only transmitted
by working alongside an experienced SUMMA developer: which build path to
take on macOS vs HPC, what each of the 40 model decisions actually controls,
how to slice the layer-resolved NetCDF output, and when to switch from the
NCAR upstream to the CH-Earth SUNDIALS fork.

**Progressive disclosure:**
- `SKILL.md` is the routing hub: decision tree, repo layout, quick start, critical rules.
- `reference/*.md` are deep-dive docs loaded on demand.

## Contents

| Document | What's inside |
|----------|---------------|
| `SKILL.md` | Entry point: decision tree, repo layout, quick start, critical rules |
| `reference/getting-started.md` | Repo, dependencies (NetCDF, LAPACK, Sundials), Makefile vs CMake vs Docker, build verification |
| `reference/architecture.md` | `build/source` directory map, derived data types, the `coupled_em` call chain, mDecisions registry |
| `reference/running-summa.md` | File manager, attributes, parameters, forcing, output control, the `summa.exe` CLI |
| `reference/model-decisions.md` | All 40 numbered decisions with options, defaults, and how they map to Noah-MP, CLM, VIC mimics |
| `reference/output-and-postprocess.md` | NetCDF dimensions, layer indexing, restart vs history, pysumma helpers |
| `reference/case-studies.md` | Reynolds Mountain East and the canonical Clark et al. 2015b experiments |
| `reference/sundials-and-solvers.md` | CH-Earth fork, IDA, KINSOL, Jacobian and band-matrix options |
| `reference/debugging.md` | Common compile, runtime, and balance failures |
| `reference/contributing-pr.md` | Fork choice, develop branch, coding conventions, PR review |

## Sources

This skill is grounded in:

1. **NCAR/summa** repository (master and develop branches) and the in-tree `docs/` mkdocs source.
2. **CH-Earth/summa** fork (active development, SUNDIALS branch).
3. **Clark et al.** foundational papers (2015a/b/c, 2021).
4. **pysumma** documentation (UW-Hydro/pysumma).
5. **SUMMA online docs**: https://summa.readthedocs.io.

## Install

This skill follows the same layout as the noahmp-skill and the
[laps-skill](https://github.com/huangzesen/laps-skill) family:

```
summa-skill/
|-- SKILL.md       <- routing hub (read first)
|-- README.md      <- this file
|-- LICENSE
|-- .gitignore
`-- reference/     <- deep-dive docs
```

To use with a Claude Code or similar agent, drop the directory into your
skills library and refresh.

## License

MIT (this skill). SUMMA itself is distributed under GPLv3, see
https://github.com/NCAR/summa/blob/master/COPYING.
