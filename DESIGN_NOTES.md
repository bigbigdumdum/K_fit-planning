# Design notes: review of CLAUDE.md

Decisions log from the first review of CLAUDE.md (2026-09-30).
Each item has Claude's comment, your response, and the resulting decision.
CLAUDE.md remains the source of truth. This file only records the discussion
and the defaults chosen where CLAUDE.md is silent.

---

## 1. Resolved decisions

### 1.1 Parser
- **Comment:** Avoiding dependencies (hand-written parser) conflicts with robust mmCIF parsing (gemmi or Biopython).
- **Response:** Use Biopython to parse.
- **Decision:** Parse with `Bio.PDB.PDBParser` and `Bio.PDB.MMCIFParser`. Dependencies: `biopython`, `numpy`, `kearsley-numba`.
- **Implementation notes:**
  - Model number comes from `model.serial_num`, the number in the file (1-based, as in the `MODEL` record or `pdbx_PDB_model_num`). It does **not** come from Biopython's 0-based `model.id`.
  - Atom id comes from `atom.get_serial_number()`: the `ATOM`/`HETATM` serial for PDB, `_atom_site.id` for mmCIF.
  - Alternate locations are read from `DisorderedAtom.disordered_get_list()` so that every altloc is visible. Each altloc has its own serial, so every altloc gets its own `matomid`.
  - Biopython reads chain and residue numbers from the `auth_*` columns for mmCIF by default. The dropdowns will therefore show author numbering, the same as PDB files.

### 1.2 Unique atom identifier (`matomid`)
- **Comment:** PDB serials are not unique across models (NMR) and overflow above 99,999.
- **Response:** Pair the atom id with the model number. New section added to CLAUDE.md.
- **Decision:** `matomid = f"{model_number}-{atom_id}"`, for example `1-245`. The parser checks every `matomid` for uniqueness and raises an error on duplicates. The manual-entry box and the pairs CSV both use `matomid`.

### 1.3 Cutoff failure
- **Comment:** Should failed fits (RMSD > RMSDc or SO < 100 %) still produce output?
- **Response:** The output should say "no fit" in the `.txt` report, along with the RMSD and structure overlap that failed.
- **Decision:** The report entry for that pair reads `NO FIT`, followed by the RMSD, SO, and the cutoff(s) that were breached. See open question 3.1 for which files are still written.
- **Note:** `kearsley_numba.get_so` returns a percentage and uses a strict `<`. "All atoms within SOdc" therefore means `SO == 100.0`.

### 1.4 Multi-model files (NMR)
- **Response:** By default, apply the transformation only to the selected model. Add an optional checkbox to superimpose all models.
- **Update (CLAUDE.md "Fitallmodels"):** when ticked, *each* model is fitted separately using the same atom selection. With a single input, the model the atoms were selected from is the reference.
- **Implemented:**
  - Default: only the model used for atom selection is fitted and moved; other models are written unchanged.
  - `fit_all_models=True`: every model of each target gets its own fit. Atoms are matched across models by chain, residue, atom name and altloc (PDB atom serials restart in each model). A model missing a selected atom is NO FIT.
  - Single input: `pipeline.self_fit_job` fits every model of a copy (`name_1`) onto the selected model.

### 1.5 Alternate locations in the dropdown
- **Response:** Show alternate locations, with occupancy in brackets.
- **Decision:** Atom dropdown label format: `CA  altloc B (occ 0.40)  [1-245]`. Atoms without an altloc show only `CA  [1-245]`.

### 1.6 Pairs and matrices output
- **Response:** Separate files are fine.
- **Decision:**
  - `<target>__pairs.csv`: one row per atom pair (reference `matomid`, name, element, residue, chain; the same fields for the target; distance after fit).
  - `transforms.csv`: one row per target (rotation R11 to R33, translation t1 to t3, RMSD, SO, atom count, status).

### 1.7 Repositories
- **Response:** Two repos.
- **Decision:** Two separate git repos (names in section 2). `git init` and every commit require your permission first.

### 1.8 Translation convention of `kearsley_numba`
- **Comment:** The library moves coordinates as `R · (x − t)`, not the usual `x' = R·x + t`.
- **Response:** Handle this only in the output file.
- **Decision:** The fitter uses `kearsley_numba`'s own `(R, t)` unchanged and moves coordinates with `kearsley_numba.transform`. Only the writer converts the values: `transforms.csv` and the report give `R` and `t_std = −R·t`, and state the convention, so the matrices work directly in PyMOL or ChimeraX.

---

## 2. Names

- **Response:** The playful names in the first proposal were rejected. Use standard, informative names.
- **Decision:** Names describe what the code does. No themed names.

**Repositories**

| Role | Name |
|---|---|
| Engine (repo 1) | `K_fit` (Python package `K_fit`) |
| Colab notebook (repo 2) | `K_fit-colab` |

**Engine modules and main members**

| Module (CLAUDE.md role) | File | Main classes / functions |
|---|---|---|
| Parser | `K_fit/parser.py` | `load_structure()`, class `Structure`, class `AtomRecord` |
| PDB download | `K_fit/fetch.py` | `fetch_pdb_entry()` |
| Input checker | `K_fit/checker.py` | `check_atom_pairs()`, `read_pairs_csv()`, class `PairCheckResult` |
| Fitter | `K_fit/fitter.py` | `fit_pair()`, `apply_transform()`, class `FitResult` |
| Output | `K_fit/writer.py` | `write_structure()`, `write_report()`, `write_pairs_csv()`, `write_transforms_csv()`, `to_standard_transform()` |
| Packaging | `K_fit/archive.py` | `zip_outputs()` |

**Notebook widget helpers** (in `K_fit-colab`): `build_atom_selector()` (model/chain, residue and atom dropdowns), `build_matomid_entry()` (manual `matomid` entry), `build_pairs_csv_upload()` (pairs CSV upload).

---

## 3. Open questions / still to confirm

### 3.1 What is written when a fit fails
Implemented as proposed (not yet confirmed by you): no fitted structure file is written unless at least one fit of that target passed. The pairs CSV, the `transforms.csv` row (status `NO FIT`) and the report entry are always written.

### 3.2 CLAUDE.md not yet updated for
- The "superimpose all models" checkbox (1.4).
- The altloc dropdown with occupancy (1.5).
- There are still two "Stage 3" headings (Input checks and Atom specification).

### 3.3 Writing output "in the same format" with Biopython
Biopython's `PDBIO` and `MMCIFIO` rebuild files from the structure object. That drops PDB header records (REMARK, SEQRES, CONECT and so on) and can reorder mmCIF categories. Proposal:
- **PDB:** keep the original lines and replace only columns 31 to 54 (x, y, z) of the moved `ATOM`/`HETATM` records, matched by `matomid`. Everything else stays byte-identical.
- **mmCIF:** load the file with `MMCIF2Dict`, replace `_atom_site.Cartn_x/y/z` for the moved atoms, and write it back with `MMCIFIO.set_dict()`. All categories are kept; whitespace and layout may change.

Both approaches use only Biopython and the standard library. Implemented as proposed; PDB `ANISOU` and mmCIF `_atom_site_anisotrop` tensors are also rotated (R U Rᵀ).

### 3.4 Defaults assumed unless you say otherwise
- Checksum: **SHA-256** of the input file bytes (of the downloaded file for PDB IDs).
- CSV column header for a PDB ID input: `<pdbid>.cif`. If two inputs share a file name, the duplicates get `_2`, `_3` and so on, both in the headers and in the output file names.
- Hetero listing in Stage 3 reports waters (HOH) separately from other hetero residues, so ligands aren't buried under hundreds of waters.
- The reference structure is never moved or rewritten, only listed in the report.

### 3.5 Minor issues in `kearsley_numbajit` (not blocking)
- `scipy` is listed as a dependency but never imported.
- `centroid()` would crash if called (`.append` on an ndarray). It is unused.
