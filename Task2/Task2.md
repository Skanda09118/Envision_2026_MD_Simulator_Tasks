# Task 2 — Visualise and Analyse Your First Simulation

## Note
A simulation you cannot visualise or analyse is not useful. OVITO is the primary
analysis tool you will use throughout this program — for structure inspection,
defect counting, and radiation damage quantification. Get comfortable with it now.

This task works entirely with the dump file you generated in Task 1.
You do not need to run any new simulations.

---

## Task 1 — Install OVITO

Download and install OVITO Basic (free) from the official site.

https://www.ovito.org/downloads/

OVITO Basic covers everything you need for this program. Do not pay for Pro.

Verify it opens without errors before moving on.

---

## Task 2 — Load your trajectory

Open OVITO. Import `dump.firstLAMMPS.lammpstrj` using:
```
File → Load File
```

You should see your atoms appear in the viewport. Use the playback controls at
the bottom to step through frames.

Figure out how to:
- Rotate, pan, and zoom the viewport
- Jump to the first and last frame
- Play the trajectory as an animation

---

## Task 3 — Colour by potential energy

By default OVITO colours all atoms the same. Change this:
```
Rendering → Color Mapping → Potential Energy
```

The exact menu path may differ slightly depending on your OVITO version — figure
it out. What you are looking for is a way to map a per-atom scalar quantity to
a colour scale.

Observe:
- Which atoms have the highest potential energy?
- Where are they located relative to the rest?
- Does the distribution change between the first and last frame?

---

## Task 4 — Compute the radial distribution function

Add the Coordination Analysis modifier to your pipeline:
```
Add Modification → Coordination Analysis
```

Set the cutoff to 3.0 (in LJ reduced units). Run it and export the g(r) plot.

What to look for:
- Position of the first peak — this is the nearest-neighbour distance
- Height of the first peak — sharp means ordered, broad means disordered
- Does g(r) approach 1.0 at large r? If not, your cutoff may be too small.

Save the g(r) data as `rdf_LJ.txt`.

---

## Task 5 — Render and export

Export a single frame as an image:
```
Render → Render Active Viewport
```

Name it `frame_LJ.png`. Include it in your PR for this task.

The image should show atoms coloured by potential energy with the colour scale
visible.

---

## Task 6 — Write your observations

Create a file called `observations_task2.md` in your branch. Write answers to
these in your own words — 2–4 sentences each:

- What does each frame in the trajectory represent physically?
- Why do atoms near the box edges appear to jump to the opposite face between
  frames?
- In your NVE run, total energy should be conserved. Open the LAMMPS log file
  from Task 1 and check whether etotal is stable. Report the drift as a
  percentage of the initial value.
- What does the first peak of your g(r) tell you about the LJ system?

---

## Submission

Push to your branch and open a PR containing:
- `frame_LJ.png`
- `rdf_LJ.txt`
- `observations_task2.md`

---

## Resources

- OVITO beginner tutorial — https://www.youtube.com/watch?v=hMsNbFoarcE
- OVITO coordination analysis docs — https://www.ovito.org/docs/current/reference/pipelines/modifiers/coordination_analysis.html
- RDF explanation — Theory Module, Section 7
