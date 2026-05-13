# Molecular_Dynamic_-_Simulation
## MD Simulation - http://www.mdtutorials.com/gmx/lysozyme/01_pdb2gmx.html

### Gromacs File Formats
```
.pdb file ---- Protein database (Coordinate File)
.gro file ---- Gromacs file (Coordinate File)
.top file ---- Description File(topology file) --- Whole system
.itp file ---- Description file(include topology) ---- Subsystem
.ndx file ---- index file
.mdp file ---- molecular dynamics Parameter file
.tpr file ---- portable run Input file (cordinate + topology + parameter)
.log file ---- log file
.der file ---- Energy file
.trr file ---- Trajectory file
.xtc file ---- Extended trajectory file (compressed)
.cpt file ---- checkpoint file
```


## MD Simulation Steps for Lysozyme Tutorial

### Create Initial state
1. Generate Topology for Protein
2. Add box and solvation to the System
3. Add Ions to the solved system

### Introduction to interaction potentials
4. Energy Minimization

Predict how the particle moves
5. Equilibration of System
6. MD Production run

##' Download the 1AKI.pdb

### Clean the molecule

```
grep -v HOH 1aki.pdb > 1aki_clean.pdb 
```
```
gmx pdb2gmx -f 1AKI_clean.pdb -o 1AKI_processed.gro -water tip3p 
```
You will get the 15 force-filled ---> take 15 OPLS

### Add the cubic box

```

gmx editconf -f 1aki_processed.gro -o 1aki_nexbox.gro -c -d 1.0 -bt cubic
```

### Solvation
```

gmx solvate -cp 1aki_nexbox.gro -cs spc216.gro -o 1aki_solv.gro -p topol.top
```

### Adding ions

create ions.mdp file --> nano ion.mdp then add the following to the file
````
nano ions.mdp
````
```
----------------------------------------------------------------------------------------------------
; ions.mdp - used as input into grompp to generate ions.tpr
; Parameters describing what to do, when to stop and what to save
integrator  = steep         ; Algorithm (steep = steepest descent minimization)
emtol       = 1000.0        ; Stop minimization when the maximum force < 1000.0 kJ/mol/nm
emstep      = 0.01          ; Minimization step size
nsteps      = 50000         ; Maximum number of (minimization) steps to perform

; Parameters describing how to find the neighbors of each atom and how to calculate the interactions
nstlist         = 1         ; Frequency to update the neighbor list and long range forces
cutoff-scheme	= Verlet    ; Buffered neighbor searching
ns_type         = grid      ; Method to determine neighbor list (simple, grid)
coulombtype     = cutoff    ; Treatment of long range electrostatic interactions
rcoulomb        = 1.0       ; Short-range electrostatic cut-off
rvdw            = 1.0       ; Short-range Van der Waals cut-off
pbc             = xyz       ; Periodic Boundary Conditions in all 3 dimensions
------------------------------------------------------------------------------------------------------
```

### To Assemble your .tpr file with the following:
```
gmx grompp -f inputs/ions.mdp -c 1AKI_solv.gro -p topol.top -o ions.tpr
gmx grompp -f inputs/ions.mdp -c 1aki_solv.gro -p topol.top -o ions.tpr

```

### Now we have an atomic-level description of our system in the binary file ions.tpr. We will pass this file to genion:
```

gmx genion -s ions.tpr -o 1aki_solv_ions.gro -p topol.top -pname NA -nname CL -neutral
```

### Will be asked to add continues Group - add Group 13 (SOL file)

## Next Level - Energy Minimization

create nano minim.mdp file using nano minim.mdp then add following
````
nano minim.mdp
````
````
--------------------------------------------------------------------------------------------
; minim.mdp - used as input into grompp to generate em.tpr
; Parameters describing what to do, when to stop and what to save
integrator  = steep         ; Algorithm (steep = steepest descent minimization)
emtol       = 1000.0        ; Stop minimization when the maximum force < 1000.0 kJ/mol/nm
emstep      = 0.01          ; Minimization step size
nsteps      = 50000         ; Maximum number of (minimization) steps to perform

; Parameters describing how to find the neighbors of each atom and how to calculate the interactions
nstlist         = 1         ; Frequency to update the neighbor list and long range forces
cutoff-scheme   = Verlet    ; Buffered neighbor searching
ns_type         = grid      ; Method to determine neighbor list (simple, grid)
coulombtype     = PME       ; Treatment of long range electrostatic interactions
rcoulomb        = 1.0       ; Short-range electrostatic cut-off
rvdw            = 1.0       ; Short-range Van der Waals cut-off
pbc             = xyz       ; Periodic Boundary Conditions in all 3 dimensions
--------------------------------------------------------------------------------------------
````
### Execute the following command 
```

gmx grompp -f inputs/minim.mdp -c 1aki_solv_ions.gro -p topol.top -o em.tpr
```

Make sure you have been updating your topol.top file when running genbox and genion, or else you will get lots of nasty error messages ("number of coordinates in coordinate file does not match topology," etc).

We are now ready to invoke mdrun to carry out the EM:
```
gmx mdrun -v -deffnm em
```

Let's do a bit of analysis. The em.edr file contains all of the energy terms that GROMACS collects during EM. You can analyze any .edr file using the GROMACS energy module:
```

gmx energy -f em.edr -o potential.xvg
```

Then select 10 by using the 10 0 command

to visualize  the Xmgrace potential.xvg

Predict how the particle Moves
Equilibration of the System

### Create nvt.mdp file
```
nano nvt.mdp file
```
````
----------------------------------------------------------------------------------------------------------
title                   = OPLS Lysozyme NVT equilibration 
define                  = -DPOSRES  ; position restrain the protein
; Run parameters
integrator              = md        ; leap-frog integrator
nsteps                  = 50000     ; 2 * 50000 = 100 ps
dt                      = 0.002     ; 2 fs
; Output control
nstxout                 = 2500      ; save coordinates every 5.0 ps
nstvout                 = 2500      ; save velocities every 5.0 ps
nstenergy               = 2500      ; save energies every 5.0 ps
nstlog                  = 2500      ; update log file every 5.0 ps
; Bond parameters
continuation            = no        ; first dynamics run
constraint_algorithm    = lincs     ; holonomic constraints 
constraints             = h-bonds   ; bonds involving H are constrained
lincs_iter              = 1         ; accuracy of LINCS
lincs_order             = 4         ; also related to accuracy
; Nonbonded settings 
cutoff-scheme           = Verlet    ; Buffered neighbor searching
ns_type                 = grid      ; search neighboring grid cells
nstlist                 = 10        ; 20 fs, largely irrelevant with Verlet
; vdW
rvdw                    = 1.2       ; short-range van der Waals cutoff (in nm)
rvdw-switch             = 1.0
vdw-modifier            = force-switch
DispCorr                = No        ; per CHARMM FF convention 
; Electrostatics
rcoulomb                = 1.2       ; short-range electrostatic cutoff (in nm)
coulombtype             = PME       ; Particle Mesh Ewald for long-range electrostatics
pme_order               = 4         ; cubic interpolation
fourierspacing          = 0.16      ; grid spacing for FFT
; Temperature coupling is on
tcoupl                  = V-rescale ; stochastic Bussi thermostat 
tc-grps                 = System 
tau_t                   = 1.0       ; value of tau (ps)
ref_t                   = 298       ; temperature (K) 
; Pressure coupling is off
pcoupl                  = no        ; no pressure coupling in NVT
; Periodic boundary conditions
pbc                     = xyz       ; 3-D PBC
; Velocity generation
gen_vel                 = yes       ; assign velocities from Maxwell distribution
gen_temp                = 298       ; temperature for Maxwell distribution
gen_seed                = -1        ; generate a random seed
----------------------------------------------------------------------------------------------------------
```
````
The first phase is conducted under an NVT ensemble (constant Number of particles, Volume, and Temperature).

We will call grompp and mdrun just as we did at the EM step:
````
gmx grompp -f inputs/nvt.mdp -c em.gro -r em.gro -p topol.top -o nvt.tpr

gmx mdrun -deffnm nvt
````

Let's analyze the temperature progression, again using energy:

````
gmx energy -f nvt.edr -o temperature.xvg
````

to check the temperature file xmgrace temperature.xvg


Create npt.mdp file
````
nano npt.mdp
````
Add the following
```

-------------------------------------------------------------------------------------------------------

title                   = OPLS Lysozyme NPT equilibration
define                  = -DPOSRES  ; position restrain the protein
; Run parameters
integrator              = md        ; leap-frog integrator
nsteps                  = 250000    ; 2 * 250000 = 500 ps
dt                      = 0.002     ; 2 fs
; Output control
nstxout                 = 500       ; save coordinates every 1.0 ps
nstvout                 = 500       ; save velocities every 1.0 ps
nstenergy               = 500       ; save energies every 1.0 ps
nstlog                  = 500       ; update log file every 1.0 ps
; Bond parameters
continuation            = yes       ; Restarting after NVT
constraint_algorithm    = lincs     ; holonomic constraints
constraints             = h-bonds   ; bonds involving H are constrained
lincs_iter              = 1         ; accuracy of LINCS
lincs_order             = 4         ; also related to accuracy
; Nonbonded settings
cutoff-scheme           = Verlet    ; Buffered neighbor searching
ns_type                 = grid      ; search neighboring grid cells
nstlist                 = 10        ; 20 fs, largely irrelevant with Verlet scheme
; vdW
rvdw                    = 1.2       ; short-range van der Waals cutoff (in nm)
rvdw-switch             = 1.0
vdw-modifier            = force-switch
DispCorr                = No
; Electrostatics
rcoulomb                = 1.2       ; short-range electrostatic cutoff (in nm)
coulombtype             = PME       ; Particle Mesh Ewald for long-range electrostatics
pme_order               = 4         ; cubic interpolation
fourierspacing          = 0.16      ; grid spacing for FFT
; Temperature coupling is on
tcoupl                  = V-rescale ; stochastic Bussi thermostat
tc-grps                 = System
tau_t                   = 1.0
ref_t                   = 298
; Pressure coupling is on
pcoupl                  = C-rescale
pcoupltype              = isotropic             ; uniform scaling of box vectors
tau_p                   = 5.0                   ; time constant, in ps
ref_p                   = 1.0                   ; reference pressure, in bar
compressibility         = 4.5e-5                ; isothermal compressibility of water, bar^-1
refcoord_scaling        = com
; Periodic boundary conditions
pbc                     = xyz       ; 3-D PBC
; Velocity generation
gen_vel                 = no        ; Velocity generation is off

--------------------------------------------------------------------------------------------------------------
```

A few other changes:
NVT equilibration phase

```

gmx grompp -f inputs/npt.mdp -c nvt.gro -r nvt.gro -t nvt.cpt -p topol.top -o npt.tpr
````
````

gmx mdrun -deffnm npt

````


Let's analyze the pressure progression, again using energy:
```

gmx energy -f npt.edr -o pressure.xvg
```


To check the pressure file xmgrace pressure.xvg



Let's take a look at density as well, this time using energy and entering "23 0" at the prompt.
```

gmx energy -f npt.edr -o density.xvg
```

Production MD Simulation

Create nano md.mdp and paste the following

````
nano md.mdp
````
```

----------------------------------------------------------------------------------------------------
title                   = OPLS Lysozyme MD run
; Run parameters
integrator              = md        ; leap-frog integrator
nsteps                  = 5000000   ; 2 * 2500000 = 10000 ps (10 ns)
dt                      = 0.002     ; 2 fs
; Output control
nstxout                 = 0         ; suppress bulky .trr file by specifying
nstvout                 = 0         ; 0 for output frequency of nstxout,
nstfout                 = 0         ; nstvout, and nstfout
nstenergy               = 5000      ; save energies every 10.0 ps
nstlog                  = 5000      ; update log file every 10.0 ps
nstxout-compressed      = 5000      ; save compressed coordinates every 10.0 ps
compressed-x-grps       = System    ; save the whole system
; Bond parameters
continuation            = yes       ; Restarting after NPT
constraint_algorithm    = lincs     ; holonomic constraints
constraints             = h-bonds   ; bonds involving H are constrained
lincs_iter              = 1         ; accuracy of LINCS
lincs_order             = 4         ; also related to accuracy
; Nonbonded settings
cutoff-scheme           = Verlet    ; Buffered neighbor searching
ns_type                 = grid      ; search neighboring grid cells
nstlist                 = 10        ; 20 fs, largely irrelevant with Verlet scheme
; vdW
rvdw                    = 1.2       ; short-range van der Waals cutoff (in nm)
rvdw-switch             = 1.0
vdw-modifier            = force-switch
DispCorr                = No
; Electrostatics
rcoulomb                = 1.2       ; short-range electrostatic cutoff (in nm)
coulombtype             = PME       ; Particle Mesh Ewald for long-range electrostatics
pme_order               = 4         ; cubic interpolation
fourierspacing          = 0.16      ; grid spacing for FFT
; Temperature coupling is on
tcoupl                  = V-rescale             ; modified Berendsen thermostat
tc-grps                 = System
tau_t                   = 1.0
ref_t                   = 298
; Pressure coupling is on
pcoupl                  = C-rescale
pcoupltype              = isotropic             ; uniform scaling of box vectors
tau_p                   = 5.0                   ; time constant, in ps
ref_p                   = 1.0                   ; reference pressure, in bar
compressibility         = 4.5e-5                ; isothermal compressibility of water, bar^-1
; Periodic boundary conditions
pbc                     = xyz       ; 3-D PBC
; Velocity generation
gen_vel                 = no        ; Velocity generation is off

----------------------------------------------------------------------------------------------------------------
```

We will run a 10-ns MD simulation, the script for which can be found here.

```

gmx grompp -f inputs/md.mdp -c npt.gro -t npt.cpt -p topol.top -o md_0_10.tpr

```

Now, execute mdrun:
```

gmx mdrun -deffnm md_0_10

```

### Protein_Ligand MD Simulation

Create Initial state
1.Generate Topology for Protein
2.Generate Topology for Ligand
3.Build a complex

Add box and Solvation to the System

Introduction to interaction Potentials
4.Energy Minimization

Predict how the particle Moves
5.Equilibration of System
6.MD Product run

