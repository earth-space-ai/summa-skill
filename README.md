# SUMMA Skill

A progressive-disclosure skill for [SUMMA](https://github.com/NCAR/summa)
(Structure for Unifying Multiple Modeling Alternatives), the process-based
hydrologic modeling framework from NCAR / Univ. of Saskatchewan / Univ. of
Washington.

> **Skill author:** Koutian Wu (ktwu01@gmail.com)
> **Skill version:** 0.1.0

> ⚠️ **Disclaimer — please read before using this skill.**
> This skill is **not a gold-standard reference**. It is a helper that lowers
> the barrier for new users to **get their hands dirty** with the model. AI
> agents (and the humans drafting this material) make mistakes; commands, file
> paths, namelist options, and physics explanations here can be wrong,
> incomplete, or out of date. **Always cross-check with the official model
> documentation, the source code, and a human expert before trusting any
> output for research, publication, or operational use.**

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

## Acknowledgments

**Gold-standard references for SUMMA** (use these to cross-check anything in this skill):
- SUMMA online docs: https://summa.readthedocs.io
- NCAR/summa repository (master/develop): https://github.com/NCAR/summa
- CH-Earth/summa active fork (SUNDIALS branch): https://github.com/CH-Earth/summa
- UW-Hydro/pysumma Python wrapper: https://github.com/UW-Hydro/pysumma
- Clark et al. foundational papers: 2015a, 2015b, 2015c, 2021

This skill exists only because of the work of other people, and any value it
has is borrowed from theirs.

- **Martyn Clark** and the **NCAR SUMMA team** for building and maintaining
  [NCAR/summa](https://github.com/NCAR/summa), and for the foundational
  Clark et al. (2015a/b/c, 2021) papers that introduce the unified flux /
  unified treatment framework this skill is built around.
- The **CH-Earth/summa** fork and the **SUNDIALS-branch** developers for the
  active engineering work the skill points users toward when they need
  recent solver behavior.
- The **UW-Hydro/pysumma** maintainers for the Python wrapper that makes
  scripting SUMMA workflows tractable for new users.
- The contributors who wrote the SUMMA Sphinx documentation at
  https://summa.readthedocs.io.
- **Zesen Huang** for [laps-skill](https://github.com/huangzesen/laps-skill),
  the progressive-disclosure layout this repo borrows.
- Sibling skills `noahmp-skill`, `parflow-skill`, and `vic-skill` for shared
  structure and cross-references across hydrologic and land-surface models.

Any errors, oversimplifications, or out-of-date claims in this skill are the
skill author's responsibility, not the upstream community's.

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
