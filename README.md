# How to build a GROMACS MD tutorial?

MD (Molecular Dynamics) is a crucial step in Drug Discovery, providing the insight interaction between atoms in real-time. GROMACS is one of the most efficient tool for simulate MD processes.  
The GROMACS MD-tutorial includes several steps.

---

## Environment preparation

Download Ubuntu or other Linux platform to do the experiment.

### 🔹 Step 1: Update your system
```bash
sudo apt update && sudo apt upgrade -y
```

---

### 🔹 Step 2: Install dependencies for GROMACS
```bash
sudo apt install build-essential cmake git libfftw3-dev libgsl-dev \
libboost-all-dev libeigen3-dev libxml2-dev libexpat1-dev \
libncurses5-dev libopenmpi-dev openmpi-bin -y
```

---

### 🔹 Step 3: Download and compile GROMACS
```bash
cd ~
wget ftp://ftp.gromacs.org/pub/gromacs/gromacs-2023.3.tar.gz
tar -xvzf gromacs-2023.3.tar.gz
cd gromacs-2023.3
mkdir build && cd build
cmake .. -DGMX_BUILD_OWN_FFTW=ON -DREGRESSIONTEST_DOWNLOAD=ON -DCMAKE_INSTALL_PREFIX=/usr/local/gromacs
make -j$(nproc)
sudo make install
```

---

### 🔹 Step 4: Source GROMACS in every session
```bash
echo 'source /usr/local/gromacs/bin/GMXRC' >> ~/.bashrc
source ~/.bashrc
```

---

### 🔹 Step 5: Install system dependencies for building Python
```bash
sudo apt update
sudo apt install build-essential libssl-dev zlib1g-dev \
libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm \
libncursesw5-dev xz-utils tk-dev libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev -y
```

---

### 🔹 Step 6: Install pyenv (Python version manager)
```bash
curl https://pyenv.run | bash
```

Add to `~/.bashrc` or `~/.zshrc`:
```bash
export PATH="$HOME/.pyenv/bin:$PATH"
eval "$(pyenv init --path)"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```

Install and set Python 3.7.17:
```bash
pyenv install 3.7.17
pyenv global 3.7.17
```

Create a virtual environment:
```bash
python -m venv gromacs_env
```

Activate environment:
```bash
source gromacs_env/bin/activate
```

---

### 🔹 Step 7: Install required Python packages
```bash
python3.7 -m pip install --upgrade pip
python3.7 -m pip install numpy
python3.7 -m pip install networkx==1.11
```

---

## ✅ Verification Commands

### Test GROMACS:
```bash
gmx --version
```

### Test Python packages:
```bash
python3.7 -c "import numpy; print(numpy.__version__)"
python3.7 -c "import networkx as nx; print(nx.__version__)"
```

---

## MD simulation

---

### 🔹 Step 1: Data cleaning

Download `.pdb` file from https://www.rcsb.org/  
Download Discovery Studio  

Clean:
- HOH, WAT (retain in case of water participates in active site)
  - scripts -> selection -> water  
- HETATM (Na+, Cl-, Cu2+, Mg2+)
  - click atoms -> delete  
- Alcohol (MET, EOH), Glucose (GLC)
  - right click -> delete  

Save as: `complex.pdb`

---

### 🔹 Step 2: Data Preparation

Create gmx folder:
```bash
mkdir C:\Users\ADMIN\Downloads\gmx
```

Open `complex.pdb`:
- scripts -> selection -> ligand -> delete -> save as `protein.pdb`
- scripts -> selection -> protein -> delete -> save as `ligand.mol2`

Run:
```bash
ls
```

---

### 🔹 Step 3: Environment Setup
```bash
source ~/gromacs_env/bin/activate
source /usr/local/gromacs/bin/GMXRC
cd /mnt/c/Users/ADMIN/Downloads/gmx
```

---

### 🔹 Step 4: Protein convert

Download force field:
https://mackerell.umaryland.edu/charmm_ff.shtml#gromacs  

```bash
tar -zxvf charmm36-jul2022.ff.tgz
```

```bash
gmx pdb2gmx -f protein.pdb -o protein_processed.gro -ter
gmx pdb2gmx -f protein.pdb -o protein_processed.gro -ignh
```

Select:
- CHARMM FF: 1  
- TIP3P: 1  
- NH3+: 0  
- COO-: 0  

---

### 🔹 Step 5: Ligand Topology Preparation

Add hydrogen using Avogadro.

Fix ligand:
```bash
perl sort_mol2_bonds.pl ligand.mol2 ligand_fix.mol2
```

Generate topology:
```bash
python3.7 cgenff_charmm2gmx_py3_nx1.py ligand ligand_fix.mol2 ligand_fix.str charmm36-jul2022.ff
```

---

### 🔹 Step 6: Build the Complex

```bash
gmx editconf -f ligand_ini.pdb -o ligand.gro
```

Create `complex.gro`:
- merge protein + ligand manually
- update atom count

---

Edit `topol.top`:
```text
#include "lig.itp"
#include "lig.prm"
```

Add:
```text
ligand     1
```

---

### 🔹 Step 7: Solvation
```bash
gmx editconf -f complex.gro -o newbox.gro -bt dodecahedron -d 1.0
gmx solvate -cp newbox.gro -cs spc216.gro -p topol.top -o solv.gro
```

---

### 🔹 Step 8: Adding Ions
```bash
gmx grompp -f ions.mdp -c solv.gro -p topol.top -o ions.tpr
```

```bash
gmx genion -s ions.tpr -o solv_ions.gro -p topol.top -pname NA -nname CL -neutral
```

---

### 🔹 Step 9: Energy Minimization
```bash
gmx grompp -f em.mdp -c solv_ions.gro -p topol.top -o em.tpr
gmx mdrun -v -deffnm em
```

---

### 🔹 Step 10: Equilibration - NVT
```bash
gmx make_ndx -f ligand.gro -o index_ligand.ndx
```

```text
0 & ! a H*
q
```

```bash
gmx genrestr -f ligand.gro -n index_ligand.ndx -o posre_ligand.itp -fc 1000 1000 1000
```

---

### 🔹 Step 11: Equilibration - NPT
```bash
gmx grompp -f npt.mdp -c nvt.gro -t nvt.cpt -r nvt.gro -p topol.top -n index.ndx -o npt.tpr
gmx mdrun -deffnm npt -v
```

---

### 🔹 Step 12: Production MD
```bash
gmx grompp -f md.mdp -c npt.gro -t npt.cpt -p topol.top -n index.ndx -o md_ligand.tpr
gmx mdrun -deffnm md_ligand -v
```

---

### 🎉 Output files:
- md_ligand.edr  
- md_ligand.log  
- md_ligand.xtc  
- md_ligand.tpr  

---

✅ The simulation is successfully completed.
