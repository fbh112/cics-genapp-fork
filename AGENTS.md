# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Workspace Type
**Application platform: IBM Z**
**Type:** IBM Enterprise COBOL for z/OS (CICS + Db2 + VSAM)
**Detected Languages:** COBOL (incl. copybooks), BMS, JCL, REXX, plain-text Workload Simulator scripts

All application content lives under `base/`. Root-level `zapp.yaml` defines the analysis profile (`containsCics: true`, syslib = `base/src` for COBOL and `base/cntl` for JCL) – keep copybook/JCL lookups pointed at those folders.

## ZAPP Configuration

**File:** `zapp.yaml`
**Reference:** https://ibm.github.io/zopeneditor-about/Docs/zapp.html

**Property groups defined:** cobol-local (COBOL), jcl-local (JCL)

**Syslib paths (copybook/include resolution):**
- COBOL: `base/src`
- JCL: `base/cntl`

**Analysis Scope (analysis profile: my-analysis):**
- COBOL (`cobol-scope`): `base/src`
- JCL (`jcl-scope`): `base/cntl`
- CICS enabled: `true`

## Build / Lint / Test
- There is no local build, linter or automated test suite. Compilation, DB2 bind and CICS install happen on z/OS by submitting the customized jobs (see `base/Building.md` for the required order).
- Files in `base/cntl/*.jcl` are **templates** containing `<PLACEHOLDER>` tokens (e.g. `<CICSHLQ>`, `<DB2HLQ>`, `<SOURCEX>`). `base/exec/cust1.rexx` generates the `@`-prefixed customized members (`@COBOL`, `@DB2BIND`, ...). Always edit the un-prefixed template, never the generated `@` member.
- "Single test" equivalent: recompile one program by running the `base/cntl/cobol.jcl` proc with `MEM=<PROGRAM>`; functional validation is manual 3270 testing (`LGSE` to initialise, then `SSC1`/`SSP1`–`SSP4`; see `base/Testing.md`).
- Compile options live inline in `cobol.jcl` (`SYSOPTF`: NODYNAM, RENT, APOST, CICS, CODEPAGE). Use apostrophes, not quotes, for literals.
- Adding a new program/copybook requires updating: the `EXEC DB2PROC,MEM=` list in `base/cntl/cobol.jcl`, the FTP `put` lists in `base/src/README.md` (and its file count), and `base/bin/install.sh` if the file extension is new (it copies only `*.bms *.cbl *.cpy *.txt` from `src/`).
- Files are transferred as PDSE members with LRECL=80, so **nothing may extend past column 72/80** and file names must be valid member names (<= 8 chars, extension stripped). `.gitattributes` enforces iso8859-1 / `zos-working-tree-encoding`; do not introduce non-Latin-1 characters.

## Code Style (COBOL)
- Fixed-format source: indicator/comment `*` in column 7, code from column 8 (Area A) / 12 (Area B). Existing files commonly start with `PROCESS SQL` (when using embedded SQL) followed by a boxed `*****` copyright header (`(C) Copyright IBM Corp.`); preserve it on new/edited files.
- Program names are `LG` + operation (`A` add, `D` delete, `I` inquire, `U` update) + entity (`C` customer, `P` policy) + layer suffix: `DB01` = Db2 data layer, `VS01` = VSAM data layer, `CUS01`/`POL01` = business logic. Presentation programs are `LGTEST*`. Member names match the lowercase file name.
- Working storage starts with a `WS-HEADER` block (eyecatcher `'<PGM>------WS'`, TRANSID, TERMID, TASKNUM, COMMAREA address/length) for dump diagnostics – keep it in every new program.
- Shared data structures are copybooks only: `LGCMAREA` (COMMAREA, pulled in with `EXEC SQL INCLUDE`), `LGPOLICY` (field lengths), `SOA*` (web service containers). Change a copybook → recompile every program that includes it, and keep COMMAREA length checks (`WS-REQUIRED-CA-LEN`) consistent.
- Host-variable mapping follows the comment block in the Db2 programs: SMALLINT `S9(4) COMP`, INTEGER `S9(9) COMP`, DATE `X(10)`, TIMESTAMP `X(26)`.

## Architecture / Error-Handling Conventions
- Strict layering via `EXEC CICS LINK PROGRAM(...)`: presentation (`LGTEST*`, BMS map `ssmap.bms`) → business (`LG?CUS01`/`LG?POL01`) → data (`LG?xDB01` Db2, `LG?xVS01` VSAM). Do not bypass layers or access Db2/VSAM from the presentation layer.
- Add flows do Db2 first, then VSAM, under a two-phase commit (VSAM failure must roll back Db2). Customer numbers come from named counter `GENACUSTNUM` in pool `GENA` when available, otherwise from Db2.
- Errors: set `CA-RETURN-CODE` in the COMMAREA (`'00'` OK, `'90'` Db2 error, `'98'` bad/short COMMAREA, others per program), write a message via `PERFORM WRITE-ERROR-MESSAGE` (formats `ERROR-MSG` with date/time/SQLCODE and LINKs to `LGSTSQ`, which writes to TDQ `CSMT` and TSQ `GENAERRS`), then `EXEC CICS RETURN`. A missing COMMAREA (`EIBCALEN = 0`) is a hard `EXEC CICS ABEND ABCODE('LGCA') NODUMP` (after logging); a too-short one returns `'98'`.
- Temporary storage queues (`GENACNTL`, `GENAERRS`) and the VSAM file are intentionally non-best-practice test fixtures for Workload Simulator / two-phase-commit demos – do not "clean them up".
- VSAM keys: customer = first 10 chars; policy = first 21 chars (type letter `C`/`E`/`H`/`M` + 10-char customer no. + 10-char policy no.). Sample data in `base/data/` must keep these fixed record lengths (225 and 64).
- `base/wsim/*.txt` are Workload Simulator scripts and must stay in the library even if the product is unused (install requirement).

## Data Dictionary Location
No data dictionary (`bobz/DD.json`) currently exists in this workspace.

## Coding Standards
Coding standards are not stored in the repo. If a coding-standard skill is available in the workspace, use it when applying standards.

## Technical Documentation Location
**Documentation Directory:** `base/` (Markdown guides) plus per-folder `README.md` files (`base/src`, `base/cntl`, `base/data`, `base/exec`, `base/wsim`, `base/event-bindings`).

| Program / Artifact | Documentation File(s) | Description/Purpose |
|--------------------|----------------------|---------------------|
| `LG?xDB01` (Db2 data layer) | `base/Architecture.md`, `base/Reference.md` | Db2 customer/policy table access |
| `LG?xVS01` (VSAM data layer) | `base/Architecture.md`, `base/Reference.md` | KSDSCUST / KSDSPOLY file access, key layout |
| `LG?CUS01`, `LG?POL01` (business logic) | `base/Architecture.md`, `base/Reference.md` | Add/inquire/update/delete business rules and LINK flow |
| `LGTEST*`, `ssmap.bms` (presentation) | `base/Architecture.md`, `base/Testing.md` | 3270 transactions SSC1, SSP1–SSP4 |
| `LGSETUP`, `LGSTSQ`, `LGASTAT1` | `base/Reference.md`, `base/Testing.md` | Counter/TSQ initialisation (LGSE), error queue, transaction counters |
| `base/cntl/*.jcl`, `base/exec/cust1.rexx` | `base/Installation.md`, `base/Building.md`, `base/Reference.md` | Customization and build/install job sequence |
| `base/wsim/*`, `base/event-bindings/*` | `base/wsim/README.md`, `base/event-bindings/README.md` | Workload Simulator scripts and CICS event bindings |

**Auto-Update Rules:**
1. When code is modified, check whether a mapped document above describes the changed behavior.
2. If documentation is out of sync, prompt the user to update it.
3. When analyzing a program, read its mapped documentation for context first.
4. Update this mapping when programs or documentation are added.
5. Keep the program/transaction tables in `base/Reference.md` current when adding or renaming programs, transactions or jobs.
