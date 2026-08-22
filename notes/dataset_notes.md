# Dataset Notes: Interpreting Activity Labels

Personal reference notes on how activity/inactivity is encoded in `data/raw/` for this repo.

## 1. General structure

Each dataset in `data/raw/` has one row per molecule (identified by `smiles`, and `mol_id` for MUV/Tox21), and one column per **assay/target**. A molecule's activity is always **per target**, not a single global label — the same molecule can be active on one target and inactive/untested on another.

Value meanings (all three datasets):
- `1` — active
- `0` — inactive
- blank/NaN — no label for that (molecule, target) pair

## 2. Tox21 (`data/raw/tox21.csv`)

12 assay columns, `1`/`0`/blank per target. Blanks are common — not every compound was screened against every target.

Columns are two panels of toxicology-relevant biological pathways:

**Nuclear Receptor (NR) panel** — hormone receptors that, if disrupted, can cause endocrine/toxic effects:
- `NR-AR` — Androgen Receptor
- `NR-AR-LBD` — Androgen Receptor (ligand-binding domain only)
- `NR-AhR` — Aryl hydrocarbon Receptor
- `NR-Aromatase` — Aromatase enzyme
- `NR-ER` — Estrogen Receptor
- `NR-ER-LBD` — Estrogen Receptor (ligand-binding domain only)
- `NR-PPAR-gamma` — receptor involved in fat/glucose metabolism

**Stress Response (SR) panel** — cellular stress pathways activated by chemical damage:
- `SR-ARE` — antioxidant stress response
- `SR-ATAD5` — DNA damage response
- `SR-HSE` — heat shock response
- `SR-MMP` — mitochondrial membrane damage
- `SR-p53` — p53 activation (DNA damage/cancer-risk pathway)

## 3. MUV (`data/raw/muv.csv`)

17 assay columns named `MUV-XXX`. Same `1`/`0`/blank semantics as Tox21, and blanks dominate (most molecules were only tested on a handful of the 17 targets).

Unlike Tox21, `MUV-XXX` names are **not** mnemonic — each number is a **PubChem BioAssay ID (AID)**. E.g. `MUV-712` = PubChem AID 712. The target's biological identity isn't spelled out in the CSV; you'd need to look up the AID on PubChem to find out what protein/pathway it represents. It's still a specific, known target — just referenced indirectly.

### Worked example

Header: `MUV-466,MUV-548,MUV-600,MUV-644,MUV-652,MUV-689,MUV-692,MUV-712,MUV-713,MUV-733,MUV-737,MUV-810,MUV-832,MUV-846,MUV-852,MUV-858,MUV-859,mol_id,smiles`

**CID2999678** — `Cc1cccc(N2CCN(C(=O)C34CC5CC(CC(C5)C3)C4)CC2)c1C`
```
,,,,,,,0,,,,0,,,,,,CID2999678,...
```
- `MUV-712 = 0` (inactive), `MUV-810 = 0` (inactive)
- all other 15 targets: blank (not tested)

**CID3240391** — `COc1ccc(OC)c(-n2c(C)nc3cc(C(=O)O)ccc32)c1`
```
0,,0,,,0,,1,,,,,,,,,,CID3240391,...
```
- `MUV-466 = 0`, `MUV-600 = 0`, `MUV-689 = 0` (inactive)
- `MUV-712 = 1` (**active**)
- all other 13 targets: blank (not tested)

Note both molecules were tested on `MUV-712`, but with opposite outcomes (CID2999678 inactive, CID3240391 active) — illustrating that the label is specific to the (molecule, target) pair, read from the intersection of a row and a target column.

## 4. DUD-E GPCR (`data/raw/dude-gpcr.csv`)

5 target columns: `aa2ar`, `adrb2`, `drd3`, `cxcr4`, `adrb1` — standard shorthand names for specific **GPCRs** (G protein-coupled receptors, a major drug-target family):
- `aa2ar` — Adenosine A2a receptor
- `adrb1` / `adrb2` — Beta-1 / Beta-2 adrenergic receptors
- `drd3` — Dopamine D3 receptor
- `cxcr4` — a chemokine receptor (also relevant in cancer/HIV research)

Built by `dude_create_csv.py`: for each target, molecules from that target's `actives_final.ism` get `1`, molecules from `decoys_final.ism` get `0`, then all targets are outer-merged on `smiles`. So here:
- `1.0` — active
- `0.0` — decoy (property-matched, presumed-inactive compound per DUD-E's methodology — not necessarily experimentally confirmed inactive)
- blank — the molecule isn't part of that target's active/decoy set at all (i.e., it belongs to a different target's data)

Verified counts confirm real `0.0` values exist (not just blanks), e.g.:
```
aa2ar: 1.0=482  0.0=31550  blank=67750
adrb2: 1.0=231  0.0=15000  blank=84551
drd3:  1.0=480  0.0=34050  blank=65252
cxcr4: 1.0=40   0.0=3406   blank=96336
adrb1: 1.0=247  0.0=15850  blank=83685
```

## 5. What is an "assay"?

An assay is a standardized lab experiment testing whether a molecule produces a specific biological effect against a specific target (e.g. binding, blocking, activating). The molecule is mixed with the target (in cells or a purified biochemical setup), a readout is measured (binding, fluorescence, enzyme activity, cell viability, etc.), and a threshold on that readout determines the `1`/`0` (active/inactive) label. Each column in these datasets = one assay = one fixed target.

Blanks are common because running an assay costs time/money — not every molecule was screened against every target. This sparsity is exactly the "low-data" problem the few-shot models in this repo (Siamese/Matching/Prototypical/Relation Nets) are designed to address: learning to predict activity on a target from only a handful of labeled examples.

## 6. Targets vs. molecules

Both are "molecules" in the broadest chemical sense, but very different in kind:
- **The molecules in the `smiles` column** — small, drug-like compounds (tens to a couple hundred atoms). These are what's being tested/screened. SMILES is a compact text notation for this kind of small molecule.
- **The targets** (`NR-AR`, `drd3`, `MUV-712`, etc.) — proteins: much larger biomolecules (thousands of atoms, folded amino acid chains) that normally perform some biological function (hormone receptor, enzyme, etc.). They are not represented structurally anywhere in these CSVs — only referenced by name/ID as "which experiment this label came from."

Relationship being measured: **small molecule (drug candidate) interacts with a protein (target)**. The model never sees the target's structure — each target instead defines a separate "task" (a separate column), and few-shot learning is used to generalize to new targets from a small number of labeled molecule examples per target.
