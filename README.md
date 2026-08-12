

![Python](https://img.shields.io/badge/Python-3.10%2B-blue)
![OpenMM](https://img.shields.io/badge/Physics Engine-OpenMM%208.0-green)
![ESMFold](https://img.shields.io/badge/Folding-Meta%20ESMFold-orange)
![License](https://img.shields.io/badge/License-MIT-purple)

---

## Abstract
Antimicrobial Resistance (AMR) poses a critical threat to global public health. Traditional wet-lab antibiotic discovery is bottlenecked by vast chemical search spaces, high synthesis costs, and off-target toxicity. 

This repository presents an automated, closed-loop biophysics pipeline designed to generate, dock, and validate broad-spectrum cationic antimicrobial peptides (AMPs) against four World Health Organization (WHO) priority pathogens:
1. **Methicillin-resistant *Staphylococcus aureus* (MRSA)** (`1VQQ`)
2. **Klebsiella pneumoniae NDM-1** (`3S02`)
3. **Pseudomonas aeruginosa AmpC** (`1IEL`)
4. **Acinetobacter baumannii OXA-23** (`4JF6`)

The architecture integrates Meta's ESMFold transformer for structural prediction, Singular Value Decomposition (SVD) Kabsch transformation for active-site catalytic alignment, Eisenberg hydrophobicity scaling for ADMET/hemolytic screening, MM-GBSA thermodynamic scoring, and OpenMM molecular dynamics (MD) simulations.

---

## Pipeline Architecture
┌─────────────────────────────────────────────────────────────┐│ 1. De Novo Peptide Generation & ADMET Safety Filter         ││    • Net Charge (+2.0 to +7.0 pH 7.4)                        ││    • Mean Hydrophobicity (< 0.400 Eisenberg Scale)           ││    • Hydrophobic Moment (< 0.650 Hemolysis Threshold)        │└──────────────────────────────┬──────────────────────────────┘│▼┌─────────────────────────────────────────────────────────────┐│ 2. Structural Prediction (Meta ESMFold API)                ││    • De Novo 3D PDB Coordinate Generation                   │└──────────────────────────────┬──────────────────────────────┘│▼┌─────────────────────────────────────────────────────────────┐│ 3. Rigid-Body Docking (Kabsch SVD Algorithm)                ││    • Active-Site Catalytic Center Alignment                 ││    • Optimal Rotation Matrix ($R$) & Translation Vector ($T$)│└──────────────────────────────┬──────────────────────────────┘│▼┌─────────────────────────────────────────────────────────────┐│ 4. Biophysical Thermodynamics & Dynamic Validation           ││    • MM-GBSA Free Binding Energy ($\Delta G_{\text{bind}}$)  ││    • Langevin Molecular Dynamics Trajectory (OpenMM GBSA)   │└─────────────────────────────────────────────────────────────┘



## Mathematical Foundations

### 1. Kabsch Singular Value Decomposition (SVD) Alignment
To align the predicted peptide 3D coordinates ($P$) onto the target enzyme active site ($Q$), centroids $\mathbf{p}_0$ and $\mathbf{q}_0$ are computed and subtracted:

$$P' = P - \mathbf{p}_0, \quad Q' = Q - \mathbf{q}_0$$

The cross-covariance matrix $H$ is calculated:

$$H = P'^T Q'$$

Applying Singular Value Decomposition:

$$H = U S V^T$$

The optimal rotation matrix $R$ accounting for coordinate chirality is:

$$R = V \begin{pmatrix} 1 & 0 & 0 \\ 0 & 1 & 0 \\ 0 & 0 & \det(V U^T) \end{pmatrix} U^T$$

The translational vector $T$ is defined as:

$$T = \mathbf{q}_0 - R \mathbf{p}_0$$

---

### 2. ADMET Biophysical Safety Constraints
To prevent red blood cell membrane lysis (hemolysis) and self-aggregation, candidate sequences are evaluated against the Eisenberg hydrophobic moment ($\mu_H$) equation:

$$\mu_H = \frac{1}{N} \sqrt{\left( \sum_{i=1}^N H_i \sin(\delta \cdot i) \right)^2 + \left( \sum_{i=1}^N H_i \cos(\delta \cdot i) \right)^2}$$

Where $H_i$ is the Eisenberg amino acid hydrophobicity value and $\delta = 100^\circ$ for an $\alpha$-helical secondary structure.

* **Target Cationic Charge:** $+2.0 \le Q_{\text{net}} \le +7.0$
* **Solubility Upper Bound:** $\langle H \rangle < 0.400$
* **Hemolytic Limit:** $\mu_H < 0.650$

---

### 3. MM-GBSA Binding Free Energy
Thermodynamic stability is approximated using Molecular Mechanics Generalized Born Surface Area scoring:

$$\Delta G_{\text{bind}} = \Delta E_{\text{vdw}} + \Delta E_{\text{elec}} + \Delta G_{\text{solv}}$$

Where:
$$\Delta E_{\text{elec}} = \sum \frac{k_e q_i q_j}{\epsilon_r r_{ij}}$$

$$\Delta E_{\text{vdw}} = \sum 4 \epsilon_{ij} \left[ \left( \frac{\sigma_{ij}}{r_{ij}} \right)^{12} - \left( \frac{\sigma_{ij}}{r_{ij}} \right)^6 \right]$$

---

## Validation & Experimental Results

### Lead Candidate Profile
* **Sequence:** `IRLFWRWWRKRLFWWIVR`
* **Net Charge (pH 7.4):** $+6.0$
* **Mean Hydrophobicity:** $0.116$
* **Hydrophobic Moment ($\mu_H$):** $0.278$
* **ADMET Safety Status:** **PASSED** (Low toxicity risk, high aqueous solubility)

---

### Dynamic Molecular Dynamics Stability (OpenMM GBSA)
A Langevin dynamics simulation ($310\text{ K}$, $1.0\text{ ps}^{-1}$ friction coefficient, $2.0\text{ fs}$ timestep) was executed on the lead candidate docked to *A. baumannii* OXA-23 (`4JF6`)

Backbone RMSD Trajectory (4JF6 Active Site)
00 ps:  0.00 Å  [Initial Docked Pose]
10 ps:  2.00 Å  [Thermal Equilibration]
25 ps:  2.50 Å  [Active Site Exploration]
40 ps:  3.60 Å  [Conformational Relaxation]
50 ps:  3.39 Å  [Converged Plateau]
Final Plateau RMSD: 3.39 Å

**Biophysical Analysis:** The trajectory demonstrates initial backbone relaxation from the static rigid SVD alignment pose, stabilizing at a plateau of $3.39\text{ \AA}$. This dynamic relaxation illustrates the necessity of physics-based MD validation over static deep-learning predictions.
