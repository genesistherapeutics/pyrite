# Pyrite: A Benchmark for Protein-Ligand Pose Ranking Agents

Pyrite is a **pairwise pose-ranking** benchmark. Each item is a pair of docked poses — Pose A
and Pose B — of the *same* protein–ligand complex, and the task is to pick the pose closer to
the experimental (crystallographic) binding mode. The ground-truth label is the closer pose.

- **750 pose pairs** over **177 crystal structures** / **120 proteins**, all from public RCSB
  PDB entries (177 unique PDB IDs).
- Each pose carries its **3D ligand coordinates**, a protein–ligand **interaction
  fingerprint**, and physics/geometry **descriptors** (MM-GBSA energy, Mogul torsion validity,
  torsion strain, PoseBusters validity, pose-shape descriptors).
- Each pair is annotated for **temporal split** (`released_after_af3_cutoff`), **difficulty**
  (`pair_kind`), **UniProt family**, ligand metadata, and water/ion context.
- A reasoning-LLM baseline (**Qwen3.8-27B-GRPO**) ships with its per-pair predictions.

Scoring convention used in the accompanying work: **mean per-structure accuracy** — per-pair
correctness averaged within each `structure_id`, then averaged across structures (equal weight
per structure), reported overall and split by `released_after_af3_cutoff`.

> **Label leakage:** columns computed against the crystal structure (`metric_rmsd_*`,
> `pose_rmsds_*`, `metric_rmsd_lt*`, `metric_lddt_pli_*`, `metric_gt_interaction_*`,
> `gt_frac`/`a_frac`/`b_frac`, and `correct`/`MM_correct`) encode the answer. Use them as
> labels/analysis only — never as model inputs.

## Files

- `data/pyrite_v6.1_curated.csv` / `.parquet` — the benchmark: 750 pairs × 113 columns.
- `data/pyrite_grpo_predictions.csv` / `.parquet` — Qwen3.8-27B-GRPO predictions per pair.
- `viewer.html` — self-contained interactive viewer (see below).

Columns suffixed `_A` / `_B` give the value for Pose A / Pose B respectively.

## Interactive viewer

`viewer.html` is a fully self-contained, offline browser viewer to aid visualization of the
benchmark — just open it in any modern browser (3Dmol.js and all data are embedded; no network
needed). For each of the
750 pairs it renders Pose A and Pose B side by side in 3D: the ligand pose as sticks inside a
**lite binding pocket** (only pocket residues within ~6 Å of the ligand, drawn as a transparent
cartoon so the poses stay in focus). It shows the **Qwen3.8-27B-GRPO** prediction and its full
reasoning trace for each pair.

The **ground truth is hidden by default** — use the *Show ground truth* toggle to reveal which
pose is crystal-closer (RMSDs to the crystal pose, which pane is correct, and whether GRPO's
pick was right). This lets you inspect the model's reasoning blind, then check the answer.

## `pyrite_v6.1_curated` columns

**Identifiers & label**
- `pair_id` — unique pose-pair identifier.
- `structure_id` — crystal-structure identifier (`<PDB>_<ligand code>`).
- `correct` — ground-truth label: which pose (`A`/`B`) is closer to the crystal pose.
- `pair_kind` — difficulty: `good_bad` (one clearly better pose) or `good_maybe` (both plausible).
- `A_pose_num`, `B_pose_num` — index of each pose within the generated pose set.

**Ligand & complex metadata**
- `pdb_id` — source RCSB PDB entry.
- `ligand_soi` — ligand of interest (`chain:resname:instance`).
- `num_ligands_in_structure` — number of ligands in the PDB entry.
- `canon_smiles` — canonical SMILES of the ligand.
- `is_drug_like` — boolean drug-likeness flag.
- `uniprot_id`, `uniprot_name`, `uniprot_description` — target UniProt accession, name, and description.
- `uniprot_family` — UniProt "belongs to …" family (or Pfam fallback); grouping key above protein.

**Temporal split & provenance**
- `deposition_date`, `release_date` — PDB deposition / release dates.
- `released_after_af3_cutoff` — boolean; `True` = released after the AlphaFold3 training cutoff (held-out / novel-complex split).

**3D structure & interactions**
- `pose_A`, `pose_B` — 3D ligand coordinates (atoms, elements, and x/y/z), compact text format.
- `pose_interactions_A`, `pose_interactions_B` — protein–ligand interaction fingerprint, `;`-separated `RESIDUE-{BB,SC}-<type>` tokens (HBD/HBA, SALT, PI_STACK, T_STACK, …).

**MM-GBSA energy**
- `mm_step1_binding_energy_A`, `mm_step1_binding_energy_B` — MM-GBSA binding free energy (lower = more favorable).
- `mm_gap` — absolute MM-GBSA energy difference between the two poses, |A − B|.
- `MM_correct` — 1 if the lower-MM-GBSA-energy pose is the crystal-closer one (label-derived convenience flag).

**Pose accuracy vs crystal (LABEL-LEAKING — analysis only)**
- `metric_rmsd_A`, `metric_rmsd_B` — ligand RMSD to the crystal pose (Å).
- `pose_rmsds_A`, `pose_rmsds_B` — ligand RMSD to the crystal pose (Å; alternate computation).
- `metric_rmsd_lt2_A/B`, `metric_rmsd_lt1_A/B` — indicator: pose RMSD < 2 Å / < 1 Å.
- `metric_pb_valid_A`, `metric_pb_valid_B` — PoseBusters validity (1 = passes all physical checks).
- `metric_rmsd_lt2_pb_valid_A/B`, `metric_rmsd_lt1_pb_valid_A/B` — pose is < 2 Å / < 1 Å **and** PoseBusters-valid.
- `metric_lddt_pli_A`, `metric_lddt_pli_B` — lDDT-PLI (protein–ligand interaction lDDT) vs crystal.
- `metric_gt_interaction_recall_A/B`, `metric_gt_interaction_iou_A/B` — recall / IoU of the pose's interactions vs the crystal interaction set.
- `metric_gt_interaction_conserved_count_A/B`, `metric_gt_interaction_missing_count_A/B`, `metric_gt_interaction_extra_count_A/B`, `metric_gt_interaction_count_A/B` — conserved / missing / extra / total interaction counts vs crystal.

**Electron-density support — 2Fo-Fc map (LABEL-LEAKING — analysis only)**

Fractions of a ligand's heavy atoms that sit in strong experimental electron density, scored
against the entry's 2Fo-Fc map (map value z-scored over the whole map, threshold ≥ 1.0 σ).
The better-density-supported pose is the crystal-closer one, so these leak the label.
- `gt_frac` — density-support fraction of the **deposited crystal ligand** (reference-quality check).
- `a_frac`, `b_frac` — density-support fraction of **Pose A** / **Pose B**.
- `map_available` — whether an experimental 2Fo-Fc density map (`<pdb>.ccp4`) was retrievable (`yes`/`no`/`error`). Constant `yes` here — the curated set keeps only entries with a usable map.
- `C1`, `C2`, `C3` — density-based exclusion flags used to build the set (all `False` here; every pair passes). `C1` = `gt_frac < 0.75` (deposited ligand poorly supported by its own density); `C2` = correct-pose density fraction `< 0.50`; `C3` = `|correct − wrong density fraction| < 0.10` (density cannot discriminate the two poses).

**Pose-intrinsic validity & strain (usable as inputs)**
- `metric_pb_valid_A/B` — see above (PoseBusters validity is pose-intrinsic, not crystal-derived).
- `metric_mogul_num_unusual_torsions_A/B`, `metric_mogul_percent_unusual_torsions_A/B`, `metric_mogul_total_torsions_A/B` — Mogul torsion-outlier count, percent, and total torsions checked.
- `metric_mogul_torsion_penalty_A/B` — aggregate Mogul torsion-geometry penalty.
- `metric_mogul_num_unusual_rings_A/B` — Mogul ring-geometry outlier count.
- `metriclog_mogul_unusual_torsions_details_A/B` — per-torsion Mogul detail records (JSON).
- `metric_total_backbone_torsion_strain_A/B`, `metric_average_backbone_torsion_strain_A/B` — ligand backbone torsion strain (total / per-torsion).
- `metric_total_sidechain_torsion_strain_A/B`, `metric_average_sidechain_torsion_strain_A/B` — ligand sidechain torsion strain (total / per-torsion).
- `metric_total_torsion_strain_A/B`, `metric_average_total_torsion_strain_A/B` — total ligand torsion strain (total / per-torsion).

**Pose shape descriptors (usable as inputs)**
- `metric_geom_Rg_A/B` — radius of gyration.
- `metric_geom_asph_A/B` — asphericity.
- `metric_geom_extent_A/B` — spatial extent.
- `metric_geom_n_close_A/B` — intramolecular close-contact count.
- `metric_geom_n_atoms_A/B` — ligand heavy-atom count.
- `metric_geom_n_res_A/B`, `metric_geom_n_sc_res_A/B` — protein residues / sidechain residues in contact.

**Pose-pair comparison (A vs B)**
- `pose_overlap_shape`, `pose_overlap_color`, `pose_overlap_combo` — shape / pharmacophore(color) / combined overlap between Pose A and Pose B.
- `protein_rmsd_AB` — protein RMSD between the two poses' complexes (induced-fit / flexibility proxy).
- `ligand_rmsd_AB` — ligand RMSD between Pose A and Pose B.
- `ligand_max_atom_disp_AB` — maximum single-atom displacement between A and B (Å).
- `ligand_n_atoms_disp_ge2_AB` — number of ligand atoms displaced ≥ 2 Å between A and B.

**Water / ion context**
- `water_class` — `protein_ligand_water` (crystallographic water bridges protein–ligand) or `no_water`.
- `missing_ion_coordinating` — boolean: a deposited metal/halide ion within 3.0 Å (inner-sphere coordination) of the crystal ligand is missing/under-represented in the reference structure. `missing_ion_nearby` — same, within 6.0 Å.
- `missing_ion_elems` — element(s) of any such missing ion.
- `missing_ion_min_dist` — distance to the nearest missing ion (Å; −1 if none).
- `n_ions_deposited_near_lig`, `n_ions_reference` — ion counts near the ligand / in the reference structure.

## `pyrite_grpo_predictions` columns

- `pair_id` — matches `pyrite_v6.1_curated.pair_id`.
- `structure_id` — crystal-structure identifier.
- `correct` — ground-truth label (`A`/`B`).
- `released_after_af3_cutoff` — temporal-split flag (see above).
- `grpo_pred` — the model's predicted pose (`A`/`B`).
- `grpo_correct` — 1 if `grpo_pred == correct`.

## Reference model: Qwen3.8-27B-GRPO

A reasoning LLM (Qwen3.8-27B) RL-tuned with GRPO to pick the crystal-closer pose from the
tabular features and interaction fingerprints (no raw coordinates).

| metric | value |
|---|---|
| per-pair (pooled) accuracy | **0.669** |
| mean per-structure accuracy — overall | **0.611** |
| mean per-structure accuracy — pre-AF3 | 0.575 |
| mean per-structure accuracy — post-AF3 (held-out / novel complexes) | **0.687** |

## Provenance & license

All structures are public RCSB PDB entries (see `pdb_id`). Poses were generated by
cofolding/docking models; the metrics are computed descriptors. No proprietary or partner data
is included. Released under **CC BY 4.0** (see `LICENSE`); please also cite the relevant PDB
entries where appropriate.
