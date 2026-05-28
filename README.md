# 2D Molecular Dynamics — Morse and EAM

An interactive teaching tool for **2D molecular dynamics** with two interatomic
potentials (Morse and Embedded-Atom Method), built from scratch.

Includes:

- **`Molecular Dynamics.ipynb`** — a Jupyter notebook deriving the physics
  (potentials, forces, velocity Verlet, virial stress) in NumPy. Smoke tests
  energy conservation for both potentials.
- **`Molecular Dynamics.html`** — a self-contained HTML/JavaScript page that
  runs the same simulation **live in the browser at ~60 FPS** with
  click-to-tweak parameters, atom tracing, MSD/RDF, and live density control.

The notebook embeds the HTML page as an iframe in its final cell, so opening
either file gives you the live simulator. There are **no external dependencies**
for the HTML — open the file in any browser and it works offline.

> Markus J. Buehler, MIT

---

## Quick start

### Live simulator (no Python)

Double-click **`Molecular Dynamics.html`**, or:

```bash
open "Molecular Dynamics.html"          # macOS
xdg-open "Molecular Dynamics.html"      # Linux
start "Molecular Dynamics.html"         # Windows
```

Click **▶ Start**. Tweak any slider while running.

### Jupyter notebook

```bash
jupyter lab "Molecular Dynamics.ipynb"
```

Runs in JupyterLab, classic Jupyter, or VS Code's Jupyter extension. The
notebook's § 5 embeds the live HTML simulator inside an `<iframe srcdoc=...>`,
so you get the 60 FPS animation directly in the cell output.

Requirements (for the Python parts only): `numpy`, `matplotlib`, `ipywidgets`,
`jupyter`. Standard scientific-Python stack.

---

![alt text](assets/image.png)

---

## What you can do

| Control | Effect |
|---|---|
| **Potential** | Switch between **Morse** ($V = D_e[1 - e^{-a(r-r_e)}]^2$) and **EAM** (Finnis–Sinclair: $E_i = -C\sqrt{\rho_i} + \tfrac12 \sum_j \phi(r_{ij})$). Switch on the fly mid-run. |
| **Lattice** | nx, ny atoms on a 2D triangular lattice; spacing $a_\text{lat}$; **periodic** or **reflecting walls** boundary conditions. |
| **Temperature** | Initial $T$, plus three thermostats: **none** (NVE), **rescale**, **Berendsen**. Live-editable target $T$. |
| **Integrator** | Velocity Verlet, live-editable $\Delta t$. |
| **Density** | A `box scale` slider rescales the box (and all positions) live. Recomputes forces, clears RDF, MSD continues with a $\text{ratio}^2$ jump. |
| **Visualisation** | Colour atoms by **KE**, **PE**, **speed**, or **virial stress** (live dropdown). |
| **Atom tracing** | Click any atom to start tracing its trajectory; click again to stop. Up to 8 traced atoms, each in a distinct colour. Trails handle PBC wraps correctly. |
| **Replay** | Scrubs through the last 400 simulation snapshots. |
| **Live time series** | Sliding-window plots of $T(t)$, total energy $E(t)$, pressure $P(t)$, **MSD** (mean-square displacement), and **g(r)** (radial distribution function, time-averaged). |

---

## Physics summary

* **Units:** reduced, with $k_B = 1$ and atom mass $m = 1$. Lengths and energies
  measured in each potential's natural scale.
* **Forces:** computed with full pairwise sums (no neighbour list — $\mathcal{O}(N^2)$
  vectorised in NumPy / inlined in JS). Fine for $N \lesssim 200$, which is
  plenty for visualisation.
* **Velocity Verlet** symplectic integrator; total energy drifts by $\sim 10^{-4}$
  over $10^3$ steps at the default $\Delta t = 0.005$.
* **Per-atom observables:** kinetic energy, potential energy, speed, virial-trace
  scalar (used as a local stress field).
* **Global observables:** temperature $T = \langle KE \rangle / N$ (2D, $k_B = 1$),
  total energy, 2D pressure $P = (2 KE + W) / (d V_\text{box})$ where $W$ is
  the pair-virial sum.
* **MSD** uses an *unwrapped* trajectory, so periodic boundary crossings don't
  cause spurious resets. Reference positions are set at Reset.
* **RDF** $g(r)$ is a time-averaged histogram of pairwise distances (minimum-image
  for periodic BCs), out to $r_\text{max} = \tfrac{1}{2}\min(L_x, L_y)$. Auto-clears
  when the density changes (box scale slider).
* **Potential conventions:** Morse is implemented in the textbook $V_A = D_e (1-e^{-a(r-r_e)})^2$
  form ($V_A \ge 0$, asymptote at $D_e$). The cohesion-comparison plot in the
  notebook uses the equivalent $V_B = V_A - D_e$ so well depths are directly
  comparable with EAM. Forces are unaffected by this constant shift.

---

## Suggested classroom experiments

* **Energy conservation.** Set `thermostat = none`. Watch the $E$ trace stay flat.
  Push $\Delta t$ past 0.015 mid-run — the integrator destabilises and $E$
  drifts. Estimate the breakdown $\Delta t$ from the Morse curvature
  $k = 2 a^2 D_e$.
* **Melting on the fly.** Equilibrate cold ($T_\text{init} = 0.02$), switch to
  Berendsen, and ramp $T_\text{target}$. Watch the hexagonal lattice break
  down — sharp peaks in g(r) broaden; MSD switches from flat (vibrations) to
  linear growth (diffusion).
* **Morse vs. EAM live.** Equilibrate as Morse, then switch the potential
  dropdown to EAM mid-run. The lattice readjusts; observe how stiffer pair
  bonds (Morse) hold the crystal more tightly than the softer many-body EAM well.
* **Surface energy.** Switch BC to `walls`, use EAM. Atoms at the walls have
  fewer neighbours $\Rightarrow$ lower $\rho_i$ $\Rightarrow$ less-negative
  $-C\sqrt{\rho_i}$ $\Rightarrow$ surface atoms are *less stable*. Colour by
  **PE** to see this directly.
* **Pressure–density isotherm.** Vary the `box scale` slider in a thermostatted
  run; record the pressure at each setting to trace out a $P(\rho)$ curve.
* **Diffusion coefficient.** In a liquid state (high $T$, Berendsen), the MSD
  slope gives $D = \text{slope} / (2 d)$ where $d = 2$.
* **Connection to ML potentials.** Note that every quantity the panel displays —
  per-atom energy, force, stress — is exactly what modern neural-network
  interatomic potentials (GAP, ACE, NEP, MACE, Allegro, …) are trained on.

---

## Implementation notes

The HTML file is **pure vanilla JavaScript** — no React, no Vue, no D3, no
jQuery, no bundler. About 1000 lines of HTML + CSS + JS:

* Physics functions (`computeForcesMorse`, `computeForcesEAM`, `step`, …)
* `requestAnimationFrame` animation loop
* Plain `<canvas>` rendering with the Agg-equivalent 2D context
* `<input type="range">` sliders + `<select>` dropdowns wired with `addEventListener`

The notebook iframe embeds the page via `<iframe srcdoc="…">`, so the simulation
runs in an isolated browsing context inside the cell — no comm channel, no
widget-render round-trip, identical 60 FPS performance whether the page is
opened standalone or inside the notebook.

---

## Files

```
Molecular Dynamics.ipynb     Jupyter notebook: physics derivation + embedded simulator
Molecular Dynamics.html      Standalone live simulator (~45 KB, no dependencies)
README.md                    This file
```

---

## License

Released under the **MIT License** — see [`LICENSE`](LICENSE) in the
repository.

## Citation

If you use this code in teaching or research, please cite:

```bibtex
@software{buehler2026md2d,
  author = {Buehler, Markus J.},
  title  = {2D Molecular Dynamics: An Interactive Teaching Tool},
  year   = {2026},
  url    = {https://github.com/lamm-mit/MolecularDynamicsModules},
  note   = {Interactive simulator: standalone HTML/JavaScript page with companion Jupyter notebook}
}
```
