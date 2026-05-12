# Moving Body Adaptive Mesh Simulation — FSP Circular Pin Tool | FreeFEM++

<p align="center">
  <img src="https://img.shields.io/badge/FreeFEM++-Simulation-blue?style=for-the-badge&logo=gnu&logoColor=white"/>
  <img src="https://img.shields.io/badge/Laplace-Heat%20Equation-orange?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Adaptive-Mesh%20Refinement-red?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Circular-Pin%20Motion-purple?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/ParaView-VTK%20Export-green?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/DOI-10.5281%2Fzenodo.20147244-blue?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey?style=for-the-badge"/>
</p>

<p align="center">
  A 2D finite element animation of a <b>circular pin tool translating through a rectangular workpiece</b> using FreeFEM++.
  Solves the Laplace (steady-state heat) equation at each frame as a hot circular pin sweeps
  from the left wall to the right wall, simulating the thermal field in Friction Stir Processing (FSP).
  The mesh adapts every frame: fine elements cluster along the pin boundary,
  coarse elements fill the far field.
</p>

<img width="1008" height="772" alt="fsp" src="https://github.com/user-attachments/assets/d8d4453a-f15b-444f-95cb-6328b40edccb" />

---

## Physics

The simulation models steady-state heat conduction around a moving circular pin tool embedded in a cold rectangular workpiece, representative of the thermal field in Friction Stir Processing:

- **Laplace equation** solved at every animation frame (quasi-static thermal field)
- **Dirichlet boundary conditions**: pin boundary held at `u = Tpeak` (hot), rectangle walls held at `u = Tamb` (cold)
- **Adaptive mesh refinement** per frame driven by the temperature gradient metric
- **Rigid body motion**: pin translates linearly along the centreline
- **Output fields**: temperature, gradient magnitude, element size map, and signed distance to pin boundary

---

## Geometry

```
y = Ly  |__________________________________|  u = 0  (top wall,    label 3)
        |                                  |
        |          u = 0 (cold domain)     |
        |                                  |
        |          O  <- circular pin      |  u = 1 on pin (label 5)
        |         ( )   --> translates --> |
        |          O    R = 0.15 m         |
        |                                  |
y = 0   |__________________________________|  u = 0  (bottom wall, label 1)
       x=0                              x = Lx

        label 4 = left wall   label 2 = right wall
```

- Rectangle: `Lx = 5.0 m` wide, `Ly = 2.0 m` tall
- Pin radius: `R = 0.15 m`
- Safety margin from walls: `0.35 m` (>= R + 0.05)
- Pin travels along the horizontal centreline `y = Ly/2`

---

## Motion Parameters

| Parameter | Symbol | Value | Description |
|-----------|--------|-------|-------------|
| Total frames | Nsteps | 60 | Animation length |
| Full rotations | Nrot | 3 | Rotations tracked during one transit (visualisation only) |
| Pin boundary segments | Npin | 60 | Mesh resolution on pin circle |
| Adaptive passes per frame | Nadapt | 3 | Refinement iterations per frame |

### Kinematic Equations

```
xc(step)  = margin + step * (Lx - 2*margin) / (Nsteps - 1)     [linear sweep, left to right]
yc        = Ly / 2                                               [fixed at centreline]
phi(step) = step * 2 * pi * Nrot / (Nsteps - 1)                 [rotation angle, tracked for output]
```

### Circular Pin Parametrisation

```
x(t) = xc + R * cos(t)
y(t) = yc + R * sin(t)
```

where `t in [0, 2*pi]` traces the pin boundary. A circle is rotationally invariant, so `phi` does not appear in the border expression — it is tracked only for console and VTU output.

---

## Governing Equation

### Laplace (Steady-State Heat Conduction)

```
-div( grad(u) ) = 0      in  Omega \ Pin
u = Tpeak                on  Gamma_pin
u = Tamb                 on  Gamma_rect
```

The weak form solved by FreeFEM++ at each frame:

```
integral_Omega [ grad(u) . grad(v) ] dOmega = 0     for all v in H^1_0
```

---

## Adaptive Mesh Strategy

At each frame the mesh is rebuilt and then refined iteratively. FE spaces are redeclared inside the loop after the final mesh is settled to prevent index out-of-bounds errors on the P0 element arrays.

| Pass | Action |
|------|--------|
| 1 | `buildmesh` at new pin position; coarse initial solve |
| 2 | `adaptmesh` on gradient metric, re-solve |
| 3 | Final `adaptmesh` pass, final solve; derived fields computed on settled mesh |

### Refinement Parameters

| Parameter | Value | Role |
|-----------|-------|------|
| `err` | 0.008 | Target interpolation error |
| `nbvx` | 50 000 | Maximum number of vertices |
| `hmax` | `Lx / 15 = 0.33 m` | Largest allowed element size |
| `hmin` | `R / 20 = 0.0075 m` | Smallest allowed element size |

---

## Output Fields

Each `.vtu` file contains the following fields:

| Field | Element Type | Description |
|-------|-------------|-------------|
| `Temperature` | P1 (nodal) | Laplace solution; Tamb (cold walls) to Tpeak (hot pin) |
| `GradMag` | P0 (element) | `\|grad(T)\|`; bright ring traces the pin boundary |
| `MeshSize` | P0 (element) | `hTriangle`; visualises adaptive refinement pattern |
| `PinDist` | P0 (element) | Signed distance to pin surface; zero contour = pin boundary |

### Signed Distance Formula

```
PinDist(x, y) = sqrt( (x - xc)^2 + (y - yc)^2 ) - R

PinDist > 0   outside pin
PinDist = 0   on pin surface  (zero contour)
PinDist < 0   inside pin hole
```

---

## Repository Structure

```
fsp-circular-freefem/
|
|-- fsp_circular.edp               # Main FreeFEM++ simulation script
|-- fsp_circular/
|   |-- fsp_circular.pvd           # ParaView collection file
|   |-- frame_000.vtu              # Frame 0  (pin at left margin)
|   |-- frame_001.vtu              # Frame 1
|   |-- ...
|   |-- frame_059.vtu              # Frame 59 (pin at right margin)
|-- README.md
```

---

## How to Run

### Requirements

- FreeFEM++ v4.10 or later: https://freefem.org
- ParaView v5.x or later: https://www.paraview.org
- Output directory `D:\freefem++\fsp_circular\` must exist before running (the script creates it automatically via `system("mkdir ...")`)

### Step 1 — Run the simulation

```bash
FreeFem++ fsp_circular.edp
```

The script will:
1. Build initial mesh with pin at the left margin (`xc = margin`)
2. Enter the animation loop over 60 frames
3. Update pin position at each frame
4. Rebuild mesh, run 3-pass adaptive solve (Laplace equation)
5. Compute derived fields: `GradMag`, `MeshSize`, `PinDist`
6. Save `.vtu` file per frame and append entry to `fsp_circular.pvd`

Console output per frame:
```
Step 1/60   xc=0.35   phi=0.00 deg    nv=1198   nt=2291
Step 2/60   xc=0.42   phi=18.37 deg   nv=1441   nt=2778
Step 3/60   xc=0.49   phi=36.73 deg   nv=1387   nt=2669
...
Step 60/60  xc=4.65   phi=1080.00 deg nv=1312   nt=2519
```

### Step 2 — Open in ParaView

1. `File > Open` → navigate to `D:\freefem++\fsp_circular\`
2. Change *Files of type* to `All Files (*.*)`
3. Select `fsp_circular.pvd` → OK
4. Choose **PVD Reader** when prompted → OK
5. Click `Apply`

---

## Suggested ParaView Visualisations

### View 1 — Adaptive Mesh Animation
```
View -> Show Edges ON
Colour field: MeshSize
Colour map:   Cool to Warm (reversed)
Press Play
```
Watch the dense triangle cluster sweep from left to right, tracking the pin position precisely. The mesh coarsens immediately outside the pin and in the far field.

### View 2 — Temperature Field
```
Colour field: Temperature
Colour map:   Black-Body Radiation
Press Play
```
The hot pin heats its surroundings; isotherms distort as the pin advances. The advancing side (where tool travel and rotation add) builds a slightly asymmetric thermal wake when advection is enabled.

### View 3 — Gradient Magnitude (Boundary Detector)
```
Colour field: GradMag
Colour map:   Inferno
Press Play
```
A bright ring traces the exact pin boundary at every frame. The highest gradients occur where the pin surface is closest to the cold workpiece walls.

### View 4 — Zero Contour of PinDist
```
Colour field: PinDist
Filters > Contour
  Contour By: PinDist
  Value:      0.0
  Apply
  Colour:     White, LineWidth 2
Press Play
```
The white contour line is the exact pin circle recovered from the signed distance field, independent of the mesh triangulation.

---

## Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `buildmesh` fails / negative area | Pin protrudes outside rectangle | Increase `margin`; verify `R + 0.05 <= margin` |
| Mesh too coarse near pin | `Npin` too low | Increase `Npin` (try 80–100) |
| `Out of bound` in `pinDist` loop | FE space declared before final `adaptmesh` | Declare `fespace Ph` inside the final `else` branch after mesh is settled |
| Slow runtime | `nbvx` or `Nadapt` too high | Reduce `nbvx` to 20 000 or set `Nadapt = 2` |
| `.vtu` files not created | Output directory missing | Script auto-creates via `system("mkdir ...")` — check drive letter |
| PVD does not animate in ParaView | Wrong reader selected | Choose **PVD Reader**, not XML or Legacy VTK |
| Temperature field all zero | Boundary labels mismatched | Confirm pin uses `label=5` and `on(5, u=Tpeak)` in solve |
| Syntax error on `T_peak` / `T_amb` | FreeFEM++ forbids underscores in identifiers | Use `Tpeak` and `Tamb` throughout |
| Garbled characters in comments | Non-ASCII encoding (em-dashes, smart quotes) from copy-paste | Replace with plain ASCII hyphens before running |

---

## Extending the Model

| Extension | What to change |
|-----------|----------------|
| More frames | Increase `Nsteps`; motion equations auto-scale |
| Larger / smaller pin | Change `R`; also adjust `margin` and `Npin` |
| Shoulder heat source | Add Gaussian source term `- int2d(Th)( Q0*exp(-((x-xc)^2+(y-yc)^2)/Rs^2)*v )` |
| Material stirring (advection) | Add `+ int2d(Th)( Pe*(vx*dx(u)+vy*dy(u))*v )` with rotational velocity field |
| Advancing/retreating asymmetry | Enable advection — asymmetry appears automatically |
| Multiple pins | Add `border Pin2(...)` and subtract from `buildmesh` |
| Poisson source term | Add `+ int2d(Th)(f*v)` to the variational form |
| Time-dependent (unsteady) | Replace `solve` with implicit Euler formulation using `dt` |
| 3D extrusion | Switch to `mesh3`, `buildlayers`, `int3d`, `P13d` elements |
| Variable wall temperature | Assign different Dirichlet values per label (1, 2, 3, 4) |
| Non-uniform feed rate | Replace linear `xc` formula with any `xc(step)` function |

---

## FSP Physics Background

In real Friction Stir Processing the dominant heat source is frictional contact between the **rotating shoulder** (radius ≈ 2R) and the workpiece surface. The pin stirs the plasticised material below. Key zones visible in the temperature field:

| Zone | Description |
|------|-------------|
| Stirred nugget | Directly under pin; highest temperature, fully recrystallised |
| TMAZ | Thermomechanically affected zone; plastically deformed but not fully stirred |
| HAZ | Heat-affected zone; heated above solvus but not mechanically deformed |
| Far field | Unaffected base material; ambient temperature |

The Laplace model here captures the conductive thermal field. For higher fidelity, add the Gaussian shoulder heat source and the advection term (see Extensions above).

---

## Citation

If you use this code in your research, please cite:

```bibtex
@software{mishra_2026_fsp_circular,
  author    = {Mishra, A.},
  title     = {Moving Body Adaptive Mesh Simulation — FSP Circular Pin Tool | FreeFEM++},
  year      = {2026},
  publisher = {Zenodo},
  doi       = {10.5281/zenodo.20147244},
  url       = {https://doi.org/10.5281/zenodo.20147244}
}
```

Plain text citation:

> Mishra, A. (2026). *Moving Body Adaptive Mesh Simulation — FSP Circular Pin Tool | FreeFEM++*. Zenodo. https://doi.org/10.5281/zenodo.20147244

---

## Author

**akshansh11**  
GitHub: https://github.com/akshansh11

---

## License

<p>
<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
<img alt="Creative Commons Licence" style="border-width:0" src="https://i.creativecommons.org/l/by-nc/4.0/88x31.png"/>
</a>
<br/>
This work is licensed under a
<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">Creative Commons Attribution-NonCommercial 4.0 International License</a>.
</p>

You are free to:

- **Share** — copy and redistribute the material in any medium or format
- **Adapt** — remix, transform, and build upon the material

Under the following terms:

- **Attribution** — You must give appropriate credit to akshansh11 and provide a link to this repository
- **NonCommercial** — You may not use the material for commercial purposes

Copyright 2026 akshansh11. All rights reserved for commercial use.
