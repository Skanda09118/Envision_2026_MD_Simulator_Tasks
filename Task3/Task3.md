# Task 3 — Real Potentials and Crystalline Silicon

## Note
Lennard-Jones is too simple to describe any real covalent or metallic material.
This task introduces the Tersoff potential — the same class of potential used
to model GaN in the radiation damage simulations you will run later.

Silicon is used here because it is single-element, well-documented, and the
potential file ships with LAMMPS. Master the workflow here and the jump to
two-element GaN is straightforward.

Pay attention to three things that are different from Task 1:
- Units are now `metal` not `lj` — all numbers have physical meaning
- The potential is read from an external file, not defined inline
- You are using NVT not NVE — there is a thermostat

---

## Task 1 — Get the potential file

LAMMPS ships with a library of potential files. Find `SiC.tersoff` in your
LAMMPS installation:
```
locate SiC.tersoff
```

If that returns nothing, download it directly from the LAMMPS potentials
repository:

https://github.com/lammps/lammps/tree/develop/potentials

Copy it into your working directory for this task. The simulation script will
look for it there.

Open the file and read the header comments. You do not need to understand every
parameter — but note what elements it contains and what reference it cites.

---

## Task 2 — Run the simulation

Create a file called `in.Si_tersoff` and paste the script below.
Read through every line before you run it. Every command should make sense
from the theory module before you execute anything.
```
# Crystalline silicon — Tersoff potential, NVT at 300 K

units           metal
atom_style      atomic
boundary        p p p

# Diamond cubic lattice, experimental lattice constant 5.431 Angstrom
lattice         diamond 5.431
region          simbox block 0 5 0 5 0 5
create_box      1 simbox
create_atoms    1 box

mass            1 28.085

# Tersoff potential — reads parameters from external file
pair_style      tersoff
pair_coeff      * * SiC.tersoff Si

# Initialise velocities at 300 K
velocity        all create 300.0 12345 loop geom

# NVT thermostat — Nose-Hoover, coupling constant 0.1 ps
fix             1 all nvt temp 300.0 300.0 0.1

timestep        0.001

thermo          100
thermo_style    custom step temp pe ke etotal press vol

dump            1 all atom 200 dump.Si_tersoff.lammpstrj
run             5000
```

Run it with:
```
lammps -in in.Si_tersoff
```

Watch the thermo output as it runs. Temperature should stabilise near 300 K
within the first 1000 steps.

---

## Task 3 — Check equilibration

Open the LAMMPS log file. Plot or inspect the temperature column over time.

Answer these before moving on:
- At what step does temperature stabilise near 300 K?
- Is etotal conserved in an NVT run? Should it be? (refer to the theory module)
- What is the average pressure after equilibration? Is it close to zero?

If pressure is far from zero (> ±5 kbar), the lattice constant in the script
does not match the potential's equilibrium value. Note this — you will fix it
properly in Task 4 using NPT.

---

## Task 4 — Visualise with CNA

Load `dump.Si_tersoff.lammpstrj` in OVITO.

Add the Common Neighbour Analysis modifier:
```
Add Modification → Common Neighbor Analysis
```

OVITO will colour each atom by its local crystal structure. In a well-equilibrated
diamond cubic crystal you should see almost all atoms coloured as
`Cubic Diamond` (shown in cyan by default).

Observe:
- What fraction of atoms are identified as non-diamond?
- Where are they located — surface, interior, or random?
- Does the fraction change between the first and last frame?

Export a rendered image as `frame_Si_CNA.png`.

---

## Task 5 — Compute and compare RDF

Add the Coordination Analysis modifier with a cutoff of 3.5 Å.

Export the g(r) as `rdf_Si.txt`.

Compare it to your `rdf_LJ.txt` from Task 2. Write your comparison in
`observations_task3.md`:

- How many peaks are visible within 5 Å? What does each correspond to
  physically in the diamond cubic structure?
- The first peak in LJ g(r) is at r ≈ 1.12σ. The first peak in Si g(r)
  should be near 2.35 Å — the Si-Si bond length. Does yours match?
- What does the sharpness of the peaks tell you about the degree of order?

---

## Task 6 — Write your observations

Add to `observations_task3.md`:

- What is the physical meaning of `units metal` vs `units lj`? List the
  units of energy, length, and time in each.
- Why does the `pair_coeff` line reference `SiC.tersoff` but only specify `Si`?
  What does LAMMPS do with the C parameters in that file?
- Why use NVT for this run rather than NVE?
- The thermostat coupling constant is set to 0.1. What would you expect to
  observe if you changed it to 0.001? To 10.0? (Answer from theory — you do
  not need to run these.)

---

## Submission

Push to your branch and open a PR containing:
- `in.Si_tersoff`
- `frame_Si_CNA.png`
- `rdf_Si.txt`
- `observations_task3.md`

---

## Resources

- Tersoff potential — Theory Module, Section 3
- CNA — Theory Module, Section 7
- NVT ensemble — Theory Module, Section 4
- LAMMPS Tersoff docs — https://docs.lammps.org/pair_tersoff.html
- OVITO CNA docs — https://www.ovito.org/docs/current/reference/pipelines/modifiers/common_neighbor_analysis.html
