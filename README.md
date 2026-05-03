# How to build a gromacs MD tutorial?

MD (Molecular Dynamics) is a crucial step in Drug Discovery, providing the insight interaction between atoms in real-time. Gromacs is one of the most efficient tool for simulate MD processes.  
The Gromacs MD-tutorial includes serveral steps

---

## Environment preparation

Download Ubuntu or other Linux platform to do the experiment.

---

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

### 🔹 Step 4: Source gromacs in every session
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

---

#### Add to ~/.bashrc or ~/.zshrc
```bash
export PATH="$HOME/.pyenv/bin:$PATH"
eval "$(pyenv init --path)"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```

---

#### Install and set Python 3.7.17
```bash
pyenv install 3.7.17
pyenv global 3.7.17
```

---

#### Create a virtual environment
```bash
python -m venv gromacs_env
```

---

#### Activate the environment
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

### Verification Commands

#### To test GROMACS:
```bash
gmx --version
```

#### To test Python packages:
```bash
python3.7 -c "import numpy; print(numpy.__version__)"
python3.7 -c "import networkx as nx; print(nx.__version__)"
```

---

## MD simulation

---

### 🔹 Step 1: Data cleaning

download .pdb file from https://www.rcsb.org/  
download Discovery Studio  

clean HOH, WAT (retain in case of Water participates in active site)
    scripts -> selection -> water  

clean HETATM (Hetero atom Na+, Cl-, Cu2+,Mg2+)
    click on atoms laid on protein (or separately) -> delete
    on PDB file: start with HETATM -> delete (optional)

clean alcol (MET,EOH), Glucose (GLC)
    right click and tow to select object -> delete  

Save and name: complex.pdb

---

### 🔹 Step 2: Data Preparation

create gmx folder - contains all of data for a MD process:
```bash
mkdir C:\Users\ADMIN\Downloads\gmx
```

Open complex.pdb  
    scripts -> selection -> ligand -> delete -> save as protein.pdb  
    scripts -> selection -> protein -> delete -> save as ligand.mol2  

run ls -> complex.pdb  protein.pdb  ligand.mol2

---

### 🔹 Step 3: Environment Setup
```bash
source ~/gromacs_env/bin/activate (Depend on where you set up the gromacs_env)
source /usr/local/gromacs/bin/GMXRC
source /usr/local/gromacs-gpu/bin/GMXRC (if you have cuda)
cd /mnt/c/Users/ADMIN/Downloads/gmx
```

---

### 🔹 Step 4: Protein convert

download force field environment on https://mackerell.umaryland.edu/charmm_ff.shtml#gromacs  
tar -zxvf charmm36-jul2022.ff.tgz  

```bash
gmx pdb2gmx -f protein.pdb -o protein_processed.gro -ter 
gmx pdb2gmx -f protein.pdb -o protein_processed.gro -ignh (In case of ignore hydrogens to avoid error)
```

Select 1 for charmmFF, Select 1 for Tip3P, Select 0 for NH3+, Select 0 for COO-

---

### 🔹 Step 5: Ligand Topology Preparation

**Important**: Add Hidro to Ligand and Fix Ligand file  

Download Avogadro: open ligand.mol2 -> Build -> add Hidrogens-> Save  

Open ligand.mol2 with Notepad, and fix something below  

<img width="1293" height="1027" alt="image" src="https://github.com/user-attachments/assets/9a079aae-03e4-4763-ae2a-c9fcd8362b26" />

Download **sort_mol2_bonds.pl** above and put in gmx folder  
```bash
perl sort_mol2_bonds.pl ligand.mol2 ligand_fix.mol2
```

Generate ligand topology with CGenFF:
Choose ligand_fix.mol2 to upload, version 4.6 and process -> Download results -> Copy all file -> Paste to gmx folder (Allow replace files)

Download cgenff_charmm2gmx_py3_nx1.py and fix the ligand_name in line resn = self.G.node[atomi].get('resname', 'ligand_name') (def write-pdb) to avoid ERROR

```bash
python3.7 cgenff_charmm2gmx_py3_nx1.py ligand ligand_fix.mol2 ligand_fix.str charmm36-jul2022.ff
```

---

### 🔹 Step 6: Build the Complex

```bash
gmx editconf -f ligand_ini.pdb -o ligand.gro
```

Make complex.gro:
Copy protein_processed.gro -> change name to complex.gro and open with notepad  
Open ligand.gro with notepad -> copy the content of ligand file to complex file  
Change the number of atoms on the top of complex.gro file by atoms = protein atoms + ligand atoms (blue square)

<img width="810" height="938" alt="image" src="https://github.com/user-attachments/assets/263b87bc-e817-4062-8aea-e5d7eb5e8633" />

Open **topol.top** and search:

```
; Include Position restraint file
#ifdef POSRES
#include "posre.itp"
#endif
```

After these lines add following lines

```
; Include ligand topology
#include "lig.itp"
```

Open topol.top and search:

```
#include "./charmm36-jul2022.ff/forcefield.itp"
```

After this line insert following lines:

```
; Include ligand parameters
#include "lig.prm"
```

Go to the end of topol.top, add the line:
```
ligand     1 (change the ligand to the corresponding name)
```

---

### 🔹 Step 7: Solvation
```bash
gmx editconf -f complex.gro -o newbox.gro -bt dodecahedron -d 1.0
gmx solvate -cp newbox.gro -cs spc216.gro -p topol.top -o solv.gro
```

---

### 🔹 Step 8: Adding Ions

download ions.mdp file (or build your own)

```bash
gmx grompp -f ions.mdp -c solv.gro -p topol.top -o ions.tpr
```

If occur error: number of coordinates in coordinate file (solv.gro, 33518) does not match topology (topol.top, 2633)  
open topol.top, scroll to the bottom of the file -> fix line break  

#### If warnings occur:
```bash
gmx grompp -f ions.mdp -c solv.gro -p topol.top -o ions.tpr -maxwarn 1
```

```bash
gmx genion -s ions.tpr -o solv_ions.gro -p topol.top -pname NA -nname CL -neutral
```

-> select the group of SOL(water)  

✅ Verification: open topol.top, see at the bottom of the file, filled with Cl or Na

---

### 🔹 Step 9: Energy Minimization
```bash
gmx grompp -f em.mdp -c solv_ions.gro -p topol.top -o em.tpr
```

#### If warnings occur:
```bash
gmx grompp -f em.mdp -c solv_ions.gro -p topol.top -o em.tpr -maxwarn 1
```

-> start energy minimization

```bash
gmx mdrun -v -deffnm em
```

---

### 🔹 Step 10: Equilibration - NVT

```bash
gmx make_ndx -f ligand.gro -o index_ligand.ndx
```

Then type:
```
0 & ! a H*
q
```

```bash
gmx genrestr -f ligand.gro -n index_ligand.ndx -o posre_ligand.itp -fc 1000 1000 1000
```

-> select group of your ligand

Find ; Include water topology in topol.top Add following lines above it.

```
; Ligand position restraints
#ifdef POSRES
#include "posre_lig.itp"
#endif
```

```bash
gmx make_ndx -f em.gro -o index.ndx
```

Then type: it means select protein + ligand
```
> 1 | 13
> q
```

download nvt.mdp or custom your own

```bash
gmx grompp -f nvt.mdp -c em.gro -r em.gro -p topol.top -n index.ndx -o nvt.tpr
gmx mdrun -deffnm nvt -v
```
or (if you use GPU)
```bash
gmx mdrun -deffnm nvt -v -nb gpu -pme gpu -ntmpi 1 -ntomp 16 -gpu_id 0
```
---

### 🔹 Step 11: Equilibration - NPT early

```bash
gmx grompp -f npt.mdp -c nvt.gro -t nvt.cpt -r nvt.gro -p topol.top -n index.ndx -o npt.tpr
```

#### If warnings occur:
```bash
gmx grompp -f npt.mdp -c nvt.gro -t nvt.cpt -r nvt.gro -p topol.top -n index.ndx -o npt.tpr -maxwarn 1
```

```bash
gmx mdrun -deffnm npt -v
```
or if you use gpu 
```bash
gmx mdrun -deffnm npt -v -nb gpu -pme gpu -ntmpi 1 -ntomp 16 -gpu_id 0
```
---
### 🔹 Step 11b: Equilibration - NPT late
delete define in npt.mdp file (to release protein and ligand)
```bash
gmx grompp -f npt_release_complex.mdp -c npt.gro -t npt.cpt -p topol.top -n index.ndx -o npt_release.tpr
```
```bash
gmx mdrun -deffnm npt_release -v -nb gpu -pme gpu -ntmpi 1 -ntomp 16 -gpu_id 0
```
### 🔹 Step 12: Production MD

create md.mdp or custom your own (you can set up the number of steps-simulation time by changing nstep field)

```bash
gmx grompp -f md.mdp -c npt.gro -t npt.cpt -p topol.top -n index.ndx -o md_ligand.tpr
gmx mdrun -deffnm md_ligand -v
```
or if it is single GPU
```bash
gmx mdrun -deffnm md_ligand -v \
-nb gpu -pme gpu \
-ntmpi 1 -ntomp 16 \
-gpu_id 0
```

<img width="1464" height="460" alt="image" src="https://github.com/user-attachments/assets/39570b42-ecf6-4ea8-848e-47c085d1166a" />

The image above shown that you successfully complete the simulation process.  
The output will include the following file md_ligand.edr, md_ligand.log, md_ligand.xtc, md_ligand.tpr
```
