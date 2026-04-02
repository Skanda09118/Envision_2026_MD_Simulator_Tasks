# Theory Reference — Molecular Dynamics Simulation

Work through sections 1–6 before Task 3. Sections 7–10 before the radiation
damage tasks. Return to this document whenever something in a script or paper
does not make sense.

---

## 1. Statistical mechanics — the foundation

Before touching MD, you need to understand what it is trying to compute.

A macroscopic property — temperature, pressure, diffusivity — is an average over
an enormous number of microscopic configurations. Statistical mechanics connects
the two. The central object is the **partition function**, which encodes the
probability of every microstate the system can occupy.

In practice, computing the partition function analytically is impossible for any
realistic system. MD sidesteps this by generating a trajectory — a time sequence
of configurations — and computing time averages directly.

This works because of the **ergodic hypothesis**: given enough time, a system
will visit all accessible microstates. The time average of a property equals its
ensemble average.
```
<A> = lim(t→∞) (1/t) ∫ A(t) dt
```

This is the foundational assumption of MD. It means your simulation must run long
enough to sample the relevant configuration space — a 100 fs run tells you almost
nothing about equilibrium properties.

### Phase space
The complete state of an N-atom system is a single point in 6N-dimensional phase
space (3N positions + 3N momenta). A trajectory is a path through this space.
Different ensembles constrain which regions of phase space the trajectory can
explore.

---

## 2. The MD algorithm

### Newton's equations of motion
MD integrates Newton's second law for every atom at every timestep:
```
F_i = m_i * a_i = m_i * (d²r_i/dt²)
```

Forces come from the gradient of the potential energy surface:
```
F_i = -∇_i U(r_1, r_2, ..., r_N)
```

The algorithm at each step:
1. Compute U from current positions
2. Compute forces F_i = -∇U for all atoms
3. Integrate equations of motion to get new positions and velocities
4. Repeat

### The Verlet integrator
The standard integrator in MD is the velocity Verlet algorithm:
```
r(t + dt) = r(t) + v(t)·dt + (1/2)·a(t)·dt²
v(t + dt) = v(t) + (1/2)·[a(t) + a(t + dt)]·dt
```

Properties:
- Time-reversible — running the simulation backward gives the same trajectory
- Symplectic — conserves a shadow Hamiltonian close to the true one, so energy
  drift is bounded rather than accumulating indefinitely
- Second-order accurate in dt

### Timestep selection
The timestep must be small enough to resolve the fastest motion in the system —
typically the highest-frequency vibration. For most solids and liquids:

- Metals: 1–2 fs
- Covalent materials (Si, GaN): 0.5–1 fs
- Molecular systems with H bonds: 0.5–1 fs
- High-energy cascade simulations: 0.001–0.01 fs during the collision phase
  (LAMMPS can adapt this automatically with `fix dt/reset`)

If dt is too large, the integrator becomes unstable and energy diverges. A quick
check: in an NVE run, total energy should drift by less than 0.01% over the
production run. More than that, reduce dt.

---

## 3. Interatomic potentials

The potential energy function U(r) is the most consequential choice in any MD
simulation. It determines what physics the simulation can and cannot capture.

### Pair potentials
U depends only on distances between pairs of atoms.

**Lennard-Jones**
```
U(r) = 4ε [ (σ/r)^12 - (σ/r)^6 ]
```
- (σ/r)^12: Pauli repulsion at short range
- (σ/r)^6: London dispersion attraction at long range
- ε: well depth (energy scale), σ: collision diameter (length scale)
- Appropriate for: noble gases, simple liquids, coarse-grained models
- Not appropriate for: directional bonding, metals, anything covalent

**Morse potential**
```
U(r) = D_e [ (1 - e^(-a(r-r_e)))² - 1 ]
```
Better than LJ for diatomic molecules — has a finite dissociation energy D_e.
Still a pair potential, so no many-body effects.

### Many-body potentials
Real materials have interactions that depend on the local chemical environment,
not just pairwise distances. Many-body potentials capture this.

**Embedded Atom Method (EAM)**
Used for metals. Each atom is treated as embedded in the electron density
contributed by its neighbours:
```
U = Σ_i F_i(ρ_i) + (1/2) Σ_i Σ_j φ_ij(r_ij)
```

where ρ_i is the local electron density at atom i and F_i is the embedding
energy. EAM correctly captures how bond strength depends on coordination number
— a surface atom bonds differently from a bulk atom.

**Tersoff potential**
Used for covalent materials: Si, C, Ge, GaN, SiC. Bond order formalism — the
strength of a bond between atoms i and j is reduced by the presence of other
neighbours (bond competition):
```
U = Σ_i Σ_{j>i} f_c(r_ij) [ f_R(r_ij) + b_ij·f_A(r_ij) ]
```

where b_ij is the bond order parameter that encodes coordination. This captures:
- Directional covalent bonding
- Correct coordination preferences (Si prefers 4-fold, not 12-fold)
- Lattice stability for diamond cubic, wurtzite, zincblende structures

**REAX force field**
Reactive force field — bonds can form and break dynamically. Used when you need
to model chemistry. Computationally expensive.

**Machine learning potentials (MLPs)**
Trained on DFT data — Gaussian Approximation Potentials (GAP), Neural Network
Potentials (NNP), MACE. Near-DFT accuracy at a fraction of the cost. The current
frontier of MD methodology.

### The ZBL potential
Ziegler-Biersack-Littmark. Designed specifically for high-energy nuclear
collisions where two atoms approach far closer than their equilibrium separation.
At these distances, nuclear-nuclear Coulomb repulsion dominates and standard
potentials are not parameterised for this regime.
```
U_ZBL(r) = (Z_1·Z_2·e²)/(4πε_0·r) · φ(r/a_u)
```

where φ is the universal screening function and a_u is the screening length.

### Hybrid potentials
Production potentials for complex simulations combine multiple potential forms:
- **Tersoff-ZBL**: Tersoff at equilibrium distances, smoothly joined to ZBL at
  short range. Mandatory for radiation damage simulations.
- **LJ + Coulomb**: for ionic materials
- **EAM + LJ**: for metal-molecule interfaces

The smooth joining function (Fermi function) between potential regions is
critical — a discontinuity in U means a discontinuous force, which destroys
energy conservation.

### Potential cutoff
Most potentials are truncated at a cutoff radius r_c beyond which interactions
are set to zero. Consequences:
- Interactions beyond r_c are ignored — choose r_c large enough that U(r_c) ≈ 0
- Cutoff must be less than half the shortest box dimension (minimum image
  convention)
- Larger cutoff = more neighbours to compute = slower simulation

Long-range interactions (Coulomb, magnetic dipole) cannot be simply truncated —
they require Ewald summation or particle-mesh Ewald (PME).

---

## 4. Statistical ensembles

An ensemble is the set of all microstates consistent with fixed macroscopic
constraints. The ensemble you choose defines which thermodynamic variables are
controlled and which fluctuate.

### NVE — microcanonical ensemble
- **Fixed**: N (atoms), V (volume), E (total energy)
- **Fluctuates**: T, P
- No thermostat, no barostat. Pure Newton's laws.
- Total energy E = KE + PE is conserved (to within integrator precision)
- The natural ensemble for isolated systems

**When to use**: production runs where you need unperturbed dynamics —
diffusion coefficients, velocity autocorrelation functions, radiation cascades.
Any thermostat modifies the equations of motion and introduces artifacts into
dynamical quantities.

**Diagnostic**: monitor etotal over time. Drift > 0.01% suggests timestep is
too large or cutoff too short.

### NVT — canonical ensemble
- **Fixed**: N, V, T
- **Fluctuates**: E, P
- A thermostat couples the system to a heat bath at temperature T

**Nosé-Hoover thermostat** (default in LAMMPS NVT):
Extends the Hamiltonian with a fictitious degree of freedom s that acts as a
heat reservoir. Equations of motion become:
```
dr_i/dt = p_i/m_i
dp_i/dt = F_i - ξ·p_i
dξ/dt = (1/Q)·[Σ p_i²/m_i - 3Nk_BT]
```

ξ is the thermostat friction coefficient, Q is the thermostat mass (set by
the coupling constant τ_T in LAMMPS). The system is ergodic and generates the
correct canonical distribution.

**Coupling constant τ_T**: the time constant for thermostat response.
- Too small (e.g. 0.001 ps): thermostat is too aggressive, real dynamics are
  damped, temperature barely fluctuates (unphysical)
- Too large (e.g. 10 ps): thermostat responds too slowly, poor temperature
  control
- Typical range: 0.05–0.5 ps for most solids

**When to use**: equilibrating a structure at target temperature, any simulation
where you care about thermodynamic averages rather than exact dynamics.

### NPT — isothermal-isobaric ensemble
- **Fixed**: N, P, T
- **Fluctuates**: E, V
- Thermostat controls T, barostat controls P by rescaling box dimensions

**Parrinello-Rahman / Nosé-Hoover barostat** (used in LAMMPS NPT):
The simulation box vectors are extended degrees of freedom with their own
equations of motion. The box expands or contracts to maintain target pressure.

Barostat options in LAMMPS:
- `iso`: all three box dimensions scale equally (cubic symmetry)
- `aniso`: x, y, z scale independently (orthorhombic)
- `tri`: full triclinic box — all cell vectors can change (most general)

For a crystal with known symmetry, use the appropriate option. For hexagonal
GaN (wurtzite), use `aniso` with x=y coupled.

**Coupling constant τ_P**: analogous to τ_T for the barostat.
Typical range: 0.5–2.0 ps. Barostats are generally slower to respond than
thermostats.

**When to use**: finding equilibrium lattice constants, relaxing a structure
to zero stress before a production run, simulating at realistic pressures.

### NσE — constant stress ensemble
Box shape and size both fluctuate under applied stress tensor σ. Used for
mechanical property calculations. Less common but important for studying
deformation.

### μVT — grand canonical ensemble
Particle number fluctuates — atoms can be inserted or deleted. Used for
adsorption isotherms, chemical equilibria. Not standard MD (requires Monte
Carlo moves for insertion/deletion).

### The standard equilibration protocol
For any production simulation, regardless of material:
```
1. Build initial structure
2. Energy minimise (conjugate gradient or FIRE algorithm)
   → removes unrealistic forces from initial geometry
3. NPT at target T and P until volume and energy are stable
   → typically 10,000–50,000 steps
4. NVT at target T
   → typically 5,000–10,000 steps
5. NVE for production (or switch to the appropriate ensemble for your measurement)
```

Skipping steps 2–3 is the most common beginner mistake. An unrelaxed structure
has residual stress that will contaminate every subsequent measurement.

---

## 5. Temperature and pressure in MD

### Temperature
Computed from kinetic energy via the equipartition theorem:
```
(3/2) N k_B T = (1/2) Σ_i m_i v_i²
```

Degrees of freedom: for N atoms with no constraints, there are 3N translational
DOF (minus 3 for centre of mass momentum conservation in periodic systems,
giving 3N-3). The correct expression for MD is:
```
T = Σ_i m_i v_i² / (N_DOF · k_B)
```

Temperature fluctuations are physical and expected. In NVT, the variance in
temperature scales as 1/N — large systems have smaller fluctuations.

### Pressure
Computed via the virial theorem:
```
P·V = N·k_B·T + (1/3) Σ_i r_i · F_i
```

The second term is the virial — the contribution from interatomic forces. In
a liquid or gas, pressure fluctuates significantly. In a crystal at low
temperature, pressure fluctuations are smaller but nonzero.

Pressure is a tensor in general (the stress tensor). The scalar pressure
reported by LAMMPS is -1/3 * trace(σ). Off-diagonal components of σ are
shear stresses.

### Initial velocities
Assigned from a Maxwell-Boltzmann distribution at the target temperature:
```
f(v_i) ∝ exp(-m_i v_i² / 2 k_B T)
```

The `velocity all create T seed` command in LAMMPS does this. The seed
determines the random number sequence — same seed gives identical initial
velocities. Use different seeds to generate independent simulation runs for
statistical averaging.

After assigning velocities, the instantaneous temperature may not exactly
equal the target. The thermostat corrects this during equilibration.

---

## 6. Periodic boundary conditions

### The problem
A simulation box of 10,000 atoms has ~1,000 surface atoms — roughly 10% of the
total. Surface atoms have a fundamentally different environment from bulk atoms.
For bulk property measurements, this surface contamination is unacceptable.

### The solution
Make the box tile infinitely by wrapping atoms that exit one face back through
the opposite face. Every atom always sees itself surrounded by bulk material.

### Minimum image convention
With PBC, every atom has infinite periodic images. For efficiency, each atom
interacts only with the nearest image of each neighbour. This requires:
```
r_cutoff < L/2
```

where L is the shortest box dimension. Violating this means an atom could
interact with two images of the same neighbour — the force is then double-counted.

### When NOT to use full PBC
- Free surfaces: `boundary p p s` (s = shrink-wrapped, non-periodic in z)
  Used for thin films, nanorods, surfaces
- Isolated molecules: non-periodic in all directions
- Nanoparticles: non-periodic, surrounded by vacuum

### Box size effects
Even with PBC, finite box size introduces artifacts:
- A defect interacts with its own periodic image if the box is too small
- Long-wavelength phonons are suppressed if the box is smaller than their
  wavelength
- For radiation cascade simulations, the damage zone must not touch its periodic
  image — requires boxes of at least 15–20 nm per side for keV PKA energies

### Elastic constants and PBC
When computing elastic constants, the choice of boundary conditions and box
shape matters. Triclinic boxes (`boundary p p p` with `box tilt large`) allow
the cell to deform under applied strain.

---

## 7. Structural analysis

### Radial distribution function (RDF / g(r))
The probability of finding an atom at distance r from a reference atom,
normalised by the ideal gas density:
```
g(r) = V/(N²) · Σ_i Σ_{j≠i} δ(r - r_ij)
```

What it tells you:
- Sharp peaks: crystalline order, peak positions give lattice parameters
- Broad peaks: liquid-like short-range order
- g(r) → 1 at large r: bulk average density
- First peak position: nearest-neighbour distance
- Integral of first peak: coordination number

g(r) is directly comparable to X-ray or neutron diffraction data (via Fourier
transform). This is one of the primary validation tools for a new potential.

### Common Neighbour Analysis (CNA)
Classifies the local crystal structure around each atom by analysing the
topology of its neighbour network. Output labels:
- FCC: face-centred cubic
- BCC: body-centred cubic
- HCP: hexagonal close-packed
- Cubic diamond / hexagonal diamond
- ICO: icosahedral (found in metallic glasses)
- Unknown: amorphous, surface, or defect environment

CNA is extremely useful for identifying grain boundaries, stacking faults,
dislocations, and amorphised regions after damage events.

Available in OVITO as the Common Neighbour Analysis modifier.

### Polyhedral Template Matching (PTM)
More robust than CNA — also identifies crystal structure but handles distorted
lattices better. Also outputs local lattice orientation (useful for polycrystal
analysis). Preferred over CNA for heavily deformed systems.

### Bond angle distribution
Histogram of angles formed by atom-neighbour-neighbour triplets. Diagnostic for
local bonding geometry:
- Tetrahedral bonding (diamond, wurtzite): peak at ~109.5°
- Octahedral: peak at ~90°
- Amorphous: broad distribution

---

## 8. Defect analysis

### Point defects
The fundamental defects in a crystalline material:

**Vacancy**: a lattice site with no atom. Created when an atom is displaced
from its site with enough energy to not return.

**Interstitial**: an atom sitting between lattice sites. In a collision cascade,
displaced atoms that travel far enough become interstitials.

**Frenkel pair**: a vacancy-interstitial pair created together. The fundamental
unit of radiation damage. A 5 keV PKA in GaN produces O(100) Frenkel pairs
initially, of which a fraction recombine during self-healing.

**Antisite** (compound materials only): an atom sitting on the wrong sublattice.
In GaN, a Ga atom on an N site (Ga_N) or N atom on a Ga site (N_Ga). Antisites
have different formation energies and diffusion barriers than vacancies and
interstitials.

**Substitutional**: an impurity atom occupying a lattice site.

### Wigner-Seitz defect analysis
The standard method for counting defects in MD output. Algorithm:

1. Define a reference (perfect) lattice
2. For each atom in the current (damaged) configuration, find the nearest
   reference lattice site
3. If a site has zero atoms assigned: **vacancy**
4. If a site has two or more atoms assigned: **interstitial** (one of those
   atoms is displaced)

In OVITO: Wigner-Seitz Defect Analysis modifier. Outputs vacancy count,
interstitial count, and antisite count for compound materials.

The net number of Frenkel pairs = number of vacancies = number of interstitials
(they are created in pairs). After self-healing, surviving Frenkel pair count
is less than the initial count.

### Displacement per atom (dpa)
The standard metric for radiation damage dose. One dpa means every atom in
the material has been displaced from its lattice site once on average.

The Norgett-Robinson-Torrens (NRT) model gives the number of Frenkel pairs
produced by a PKA of energy T:
```
N_FP = 0.8 · T_dam / (2 · E_d)
```

where T_dam is the damage energy (PKA energy minus electronic losses) and
E_d is the threshold displacement energy (~20–45 eV depending on material and
direction).

The factor 0.8 accounts for recombination during the cascade. The actual
survival fraction in MD is typically 20–30% of the NRT prediction — MD
captures recombination that the analytical model ignores.

### Threshold displacement energy (E_d)
The minimum kinetic energy an atom must receive to be permanently displaced
from its lattice site. Below E_d, the displaced atom recombines with its
vacancy within the thermal spike.

E_d is anisotropic — it depends on the direction of the recoil relative to
the crystal axes. It is measured by running many MD simulations with different
recoil directions and energies and finding the threshold for permanent
displacement.

Typical values:
- Si: ~13–20 eV
- Cu: ~19 eV
- GaN (Ga sublattice): ~20–25 eV
- GaN (N sublattice): ~14–20 eV

### Mean square displacement (MSD)
```
MSD(t) = <|r_i(t) - r_i(0)|²>
```

Measures how far atoms have moved on average from their initial positions.
In a crystal at equilibrium, MSD saturates (atoms vibrate around fixed sites).
After a radiation cascade, MSD jumps during the cascade and then plateaus at
a higher value — the plateau height encodes the surviving damage.

MSD is also used to compute diffusion coefficients in liquids and for studying
defect migration:
```
D = MSD(t) / (6t)   [3D diffusion]
```

---

## 9. Radiation damage physics

### The PKA and the cascade
When an energetic particle (neutron, ion, electron) strikes a lattice atom,
it transfers kinetic energy E_PKA. If E_PKA > E_d, the struck atom (the PKA)
is displaced and travels through the lattice, creating further displacements.

The cascade develops in three phases:

**Ballistic phase (~0.1 ps)**
PKA and secondaries collide with lattice atoms. Atom-atom distances are far
below equilibrium — ZBL potential governs. Energy is deposited into a localised
volume.

**Thermal spike (~0.1–10 ps)**
The collision cascade thermalises. Local temperature in the damage zone reaches
10,000–50,000 K. The material in this zone is transiently liquid-like. Atoms
are highly mobile and many vacancy-interstitial pairs recombine spontaneously.
This is self-healing.

**Cooling phase (~10 ps – 1 ns)**
Energy diffuses away from the damage zone via phonon transport. The surviving
defect population freezes in. Further recombination requires thermal activation
over longer timescales (ms–s at room temperature).

### Electronic stopping
At high PKA energies, the PKA also loses energy to the electron cloud (not just
nuclear collisions). This electronic energy loss does not displace atoms directly
but does heat the electronic subsystem.

In classical MD, there are no electrons. Electronic stopping is approximated by
adding a velocity-dependent drag force to fast-moving atoms:
```
F_elec = -γ · v
```

where γ is calibrated to match SRIM/TRIM electronic stopping data.
In LAMMPS: `fix electronic/stopping` or the `-Bragovic` option.
For 5 keV PKA in GaN, electronic stopping is a minor correction but should be
included for quantitative accuracy.

### Self-healing mechanisms
In MD timescales (~nanoseconds):
- Direct recombination during thermal spike: interstitial falls back into
  nearby vacancy during the hot disordered phase
- Focusson recombination: chain of correlated displacements along a crystal
  direction ending in recombination

At experimental timescales (not accessible in classical MD):
- Thermally activated migration of vacancies and interstitials to sinks
  (surfaces, grain boundaries, dislocations)
- Recombination via diffusion — requires T > ~0.3 T_melt for most materials
- Annealing treatments deliberately exploit this to restore crystallinity

### The surviving defect fraction
Defined as:
```
η = N_surviving / N_NRT
```

where N_NRT is the NRT model prediction and N_surviving is the Wigner-Seitz
count after the cascade has cooled. Typical values: 0.2–0.4 for most materials.

A higher η means less efficient self-healing. Materials with lower E_d,
higher thermal conductivity (faster cooling), or more open crystal structures
tend to have lower η.

---

## 10. Computing material properties from MD

### Elastic constants
Apply a small strain ε to the simulation box and measure the resulting stress σ.
The elastic constant C_ijkl = dσ_ij/dε_kl.

In LAMMPS: use the `deform` fix or the built-in `elastic` example scripts.
Must be done on a fully relaxed (NPT-equilibrated) structure.

### Thermal conductivity
Computed via the Green-Kubo relation:
```
κ = V/(3 k_B T²) ∫_0^∞ <J(0)·J(t)> dt
```

where J is the heat flux vector. The integral of the heat flux autocorrelation
function gives κ. Requires long NVE runs (~10 ns) with careful averaging.

Alternatively, non-equilibrium MD (NEMD): impose a temperature gradient and
measure the resulting heat flux. Simpler but has finite-size effects.

### Melting point
Heat the system slowly in NPT and monitor volume and enthalpy.
The discontinuous jump in both quantities marks the melting transition.
MD melting points are typically overestimated vs experiment due to lack of
heterogeneous nucleation sites — superheating artifact.

### Phonon dispersion
Compute the velocity autocorrelation function and Fourier transform:
```
VDOS(ω) = ∫_0^∞ <v(0)·v(t)> e^{iωt} dt
```

The vibrational density of states (VDOS) gives phonon frequencies.
For full dispersion curves: lattice dynamics from finite difference force
constants, or use the PHONON package interfaced with LAMMPS.

### Diffusion coefficient
From MSD in NVE or NVT:
```
D = lim_{t→∞} MSD(t) / (6t)
```

Must run long enough that the MSD has entered the linear diffusive regime
(slope = 1 on a log-log plot of MSD vs t).

---

## 11. Validation and best practices

### Always validate your potential
Before any production run, confirm that your potential reproduces known
experimental or DFT data:
- Lattice constants (< 1% error acceptable)
- Elastic constants (< 5–10% acceptable)
- Phonon frequencies at Γ point
- Melting point order of magnitude

A potential that fails basic validation will give you unphysical results
regardless of how carefully you set up the simulation.

### Size convergence
Run the same simulation at increasing box sizes until your property of interest
stops changing. For radiation damage, this typically means doubling the box
until the surviving defect count per PKA stabilises.

### Statistical averaging
A single MD run is one trajectory. For any quantity with statistical uncertainty,
run 5–10 independent simulations with different random seeds and report mean ± std.
This is especially important for cascade simulations where the damage morphology
varies significantly between runs.

### Energy conservation as a diagnostic
In any NVE run, total energy should be conserved to < 0.01% over the production
period. If it drifts:
- Reduce timestep
- Increase potential cutoff
- Check for atoms overlapping in the initial structure (minimise first)

### The log file
LAMMPS writes everything to the log file. Read it. It tells you:
- Whether the simulation ran stably
- Whether pressure equilibrated
- Whether temperature hit its target
- Any warnings about lost atoms or dangerous neighbour builds

Lost atoms almost always mean a timestep too large for a high-energy event.

---

## Checkpoint questions

Work through these before the radiation damage tasks. No submission required.

**Foundations**
1. Why does the ergodic hypothesis matter for MD? Under what conditions might
   it break down?
2. What happens to energy conservation if you increase the timestep by 10x?
   By 100x?
3. Why can Lennard-Jones not describe a covalent solid like silicon?

**Ensembles**
4. You run NPT and observe that the volume is still drifting at step 50,000.
   What are three possible causes?
5. Why should you never use NVT for a radiation cascade simulation?
6. What is the difference between a thermostat coupling constant that is too
   small vs too large? What would you observe in each case?

**Structure and defects**
7. A g(r) plot shows sharp peaks at 1.95 Å, 3.19 Å, and 3.74 Å. What can
   you infer about the material?
8. After a cascade, Wigner-Seitz analysis shows 80 vacancies and 78 interstitials.
   Is this physically consistent? What does the discrepancy suggest?
9. Why does E_d vary with crystal direction?

**Radiation damage**
10. A 5 keV PKA produces 94 Frenkel pairs initially. After cooling, 31 survive.
    What is the surviving fraction? How does this compare to the NRT model?
11. Why does the thermal spike cause self-healing rather than additional damage?
12. Your cascade damage zone is 8 nm across and your simulation box is 10 nm
    per side. What artifact will appear in your results?

---

## Further reading

- Frenkel & Smit — *Understanding Molecular Simulation* (the standard textbook)
- Allen & Tildesley — *Computer Simulation of Liquids*
- Was — *Fundamentals of Radiation Materials Science* (for the damage physics)
- Nordlund et al. — "Improving atomic displacement and replacement calculations
  with physically realistic damage models" (Nature Communications, 2018)
- LAMMPS documentation — https://docs.lammps.org
- OVITO documentation — https://www.ovito.org/docs/current/
