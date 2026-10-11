# Vina docking pipeline

One Python script that takes a folder with proteins and ligands, docks every ligand into every protein with AutoDock Vina, and shows the results in ChimeraX. I wrote it because preparing files and running Vina by hand for each pair got old fast

Everything lives in `docking_pipeline.py`. Drop it into a folder with your input files and run it:

```
python docking_pipeline.py
```

## What it does

1. Looks at the files next to the script. A `.pdb` / `.ent` with 20 or more amino acids counts as a protein. Everything else (`.sdf`, `.mol`, `.mol2`, small `.pdb`) counts as a ligand. An SDF with several molecules gets split into separate ligands.
2. Prepares the files. Proteins: water and other non-protein stuff (crystal ligands, cofactors, ions) are removed, hydrogens are added, PDBQT is written with Open Babel. Ligands: RDKit builds a 3D structure if there isn't one, Meeko writes the PDBQT.
3. Picks a docking box (see below) and runs Vina for every protein/ligand pair.
4. Saves poses, score tables and summaries into `results/`.
5. Opens ChimeraX, saves a PNG of every pose, and leaves a small viewer open so you can flip through proteins and poses

## Requirements

- Python 3 (I use 3.14)
- `vina.exe` next to the script, or `vina` in your PATH. Download it from the AutoDock Vina releases page. I don't ship it because it has its own license.
- ChimeraX, only for the last step. The script looks for it in PATH and the usual install folders and takes the newest version it finds. If that fails, set `CHIMERAX_PATH` at the top of the script.
- Python packages: `rdkit`, `meeko`, `openbabel-wheel`, `pyKVFinder`. If something is missing the script tries to `pip install` it

I run this on Windows. It probably works elsewhere, but I haven't tried

By default everything that isn't protein is thrown out of the receptor, including the ligand the structure was crystallized with. Otherwise it would sit in the real binding pocket and block it. Modified amino acids like MSE stay, and so do common metal ions (Zn, Fe, Mn, Cu, Co, Ni, Mg). If you want to keep all of it, set `REMOVE_HETATM = False`

## How the box is chosen

Three options, tried in this order:

**1. `context.txt`.** Put numbers in it: center x y z, then size x y z, so six numbers per protein. Several proteins means several groups of six, in the order of the file names:

```
12.5 8.3 -4.1 20 20 20
30.0 15.2 7.7 20 20 20
```

If there is only one group, it is used for all proteins. If there are fewer groups than proteins, the script stops. Remember that every PDB has its own coordinate system

**2. Automatic cavity search.** If there is no `context.txt`, pyKVFinder looks for cavities in each protein. The script takes the largest ones (`MAX_CAVITIES`, 2 by default, and only those bigger than `MIN_CAVITY_VOLUME`) and puts a 20 A box on each. Every ligand is docked into each of those boxes and the best score is kept. The boxes are saved in `results/<protein>/box.txt`

The largest cavity is not always the real binding site, and Vina tends to like big buried pockets. Treat it as a guess and look at the pictures

**3. Blind docking.** If no cavity was found, the box covers the whole protein. Slow and not very accurate, but it never misses

### Comparing ligands

With `COMPARE_LIGANDS = False` (default) each ligand picks its own best cavity. That shows where it likes to bind, but scores from different cavities aren't really comparable

With `COMPARE_LIGANDS = True` the script picks one cavity per protein, the one with the best mean score over all ligands, and takes everything from there. Use this when you want to rank ligands against each other

## Settings

All of them are at the top of the script with comments. The ones you will touch most:

| Setting | What it does |
|---|---|
| `CHIMERAX_PATH` | empty means auto-detect, or the full path to ChimeraX |
| `REMOVE_HETATM` | remove crystal ligands, cofactors and ions from the protein |
| `KEEP_METAL_IONS` | keep metals even when `REMOVE_HETATM` is on |
| `EXHAUSTIVENESS` | Vina search effort, higher is slower |
| `MAX_CAVITIES` | how many cavities to dock into |
| `MIN_CAVITY_VOLUME` | skip cavities smaller than this (A^3) |
| `AUTO_BOX_SIZE` | box size around each cavity |
| `COMPARE_LIGANDS` | see above |
| `MAX_POSES` | poses per ligand to render |
| `SHORT_SNAPSHOT_NAMES` | shorten ligand names in snapshot file names |

## Output

```
results/
  README.txt          short explanation of the files
  summary_all.txt     best score of every ligand for every protein (also .csv)
  <protein>/
    summary.txt       same for one protein (also .csv)
    box.txt           boxes that were used
    docking/          <ligand>_docked.pdbqt, <ligand>_scores.txt
    snapshots/        <protein>_<ligand>_pose<N>.png, e.g. 1HSG_lig14_pose1.png
```

The summary tables are sorted by score, best first. The scores are in kcal/mol, lower (more negative) is better. Vina scores are rough estimates. Compare ligands with each other, don't trust the absolute number

## ChimeraX viewer

When the script finishes, ChimeraX opens with all proteins loaded, one shown at a time. The cavities are the green dots. The frames of a protein are the poses of its first ligand, then the poses of the second, and so on

Commands:

- `dprot 2` shows protein number 2
- `dframe 7` shows frame 7 of the current protein
- `dnext`, `dprev` go through the frames

The Log panel also prints clickable links for the same things, plus the ligand name and score of the current frame

ChimeraX can't open PDBQT, so the script converts everything to plain PDB first. The converted files are in `temp_poses/`

## Good to know

- Every run deletes the old `.pdbqt` files in `protein_output/` and `ligand_output/`.
- Loading all poses at the end takes a while if you have many proteins and ligands.
- Cavity detection is a heuristic. If the picture looks wrong, set the box yourself with `context.txt`.
-Copyright (c) 2026 rauanu-cas9. All rights reserved.
