# Note
If you are on Windows, use Windows Subsystem for Linux (WSL). All commands below should be run inside your WSL terminal.

---

### Task 6 — Download and install LAMMPS

Install LAMMPS on your system using the package manager. Verify the installation worked by checking the version.
```bash
sudo apt update && sudo apt install lammps -y
```

**Tutorial videos**
- Installing WSL — https://www.youtube.com/watch?v=zZf4YH4WiZo
- Installing LAMMPS on WSL — https://www.youtube.com/watch?v=kOuxvF_KvCk

---

### Task 7 — Watch: What is Molecular Dynamics?

Before running any simulations, build some intuition for what LAMMPS is actually doing under the hood.

**Watch this before moving on** — https://www.youtube.com/watch?v=lLFEqKl3sm4

There's nothing to submit here, just watch it.

---

### Task 8 — Watch: Your first LAMMPS input script

These videos walk through the structure of a LAMMPS input file line by line. Pay attention to how units, atom styles, potentials, and the run command all fit together.

**Watch these before moving on**
- https://www.youtube.com/watch?v=GXA2PyqKYdY
- https://www.youtube.com/watch?v=7Ila18g8zSY
- https://www.youtube.com/watch?v=BOJPl9A7-K8

Nothing to submit, just watch.

---

### Task 9 — Run your first LAMMPS simulation

This is the hello world of molecular dynamics — a small box of atoms interacting via a Lennard-Jones potential, the simplest model of how real atoms push and pull each other.

Create a file called `in.firstLAMMPS`, read through the script below until you understand what each block is doing, then paste it in.
```lammps
# A handful of atoms in a box, interacting via Lennard-Jones

units           lj
atom_style      atomic
boundary        p p p

# Create a small 3D box and fill it with atoms on an FCC lattice
lattice         fcc 0.8442
region          simbox block 0 3 0 3 0 3
create_box      1 simbox
create_atoms    1 box

# Define the atoms and interactions
mass            1 1.0
pair_style      lj/cut 2.5
pair_coeff      1 1 1.0 1.0 2.5

# Give atoms a random initial velocity (temperature = 1.0 in LJ units)
velocity        all create 1.0 42 loop geom

# Use NVE integration — constant energy, no thermostat
fix             1 all nve

# Print energy and temperature every 100 steps
thermo          100
thermo_style    custom step temp pe ke etotal

# Run for 500 steps and write atom positions to a file
dump            1 all atom 100 dump.firstLAMMPS.lammpstrj
run             500
```

Run it with:
```bash
lammps -in in.firstLAMMPS
```

When it finishes, a file called `dump.firstLAMMPS.lammpstrj` will appear. This contains the positions of every atom at each snapshot — the raw output of your first MD simulation. Open it and take a look at what's inside.

**Questions to think about before the next session**
- What does `boundary p p p` mean and why does it matter?
- What is NVE integration conserving?
- What do `pe`, `ke`, and `etotal` represent in the thermo output?
- How do we visualise the dumped file? (answered in the next task)
