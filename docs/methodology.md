# Methodology

## 1. Methodological Overview

Circuit Builder & Simulator was developed as a browser-based schematic editor
and introductory direct-current (DC) circuit analysis tool. Its methodology
combines:

1. grid-based schematic capture;
2. coordinate-based electrical connectivity;
3. ideal steady-state component models;
4. Modified Nodal Analysis (MNA);
5. direct numerical solution by Gaussian elimination; and
6. in-browser presentation, persistence, and export.

The application is implemented as a single-page client-side application in
`index.html`. Semantic HTML defines the interface, CSS and Tailwind CSS provide
the responsive layout, vanilla JavaScript manages state and interactions, and
Scalable Vector Graphics (SVG) represents the circuit diagram. No server is
required for the editor or solver, and circuit data remains in the browser
unless the user explicitly saves or exports it.

## 2. Schematic Capture and Data Representation

### 2.1 Component model

Each placed component is represented by a JavaScript object containing:

- a unique identifier;
- a component type;
- a reference label;
- an editable value;
- horizontal and vertical canvas coordinates; and
- a rotation angle.

Component definitions specify a default label prefix, default value, and one or
more terminal offsets relative to the component center. The included library
contains resistors, capacitors, polarized capacitors, inductors, diodes, LEDs,
NPN and PNP transistors, DC and AC supplies, ammeters, voltmeters, grounds,
switches, bulbs or loads, and junctions.

### 2.2 Placement, rotation, and alignment

Components are placed on a 20-unit grid. Pointer coordinates are rounded to the
nearest grid coordinate so that symbols and wires remain consistently aligned.
Desktop users can drag components from the library, while coarse-pointer and
mobile users can select a component and tap the canvas.

Terminal positions are calculated from each terminal's local offset. For a
component rotated by angle \(\theta\), an offset \((d_x,d_y)\) is transformed as

\[
d'_x=d_x\cos\theta-d_y\sin\theta
\]

\[
d'_y=d_x\sin\theta+d_y\cos\theta
\]

and then translated by the component's canvas position. Rotation is restricted
through the interface to 90-degree increments. When a connected component is
moved or rotated, wire endpoints attached to its terminals are moved to the
new terminal positions.

### 2.3 Wire construction

Wires are stored as straight segments defined by two endpoint coordinates. When
two terminals are not horizontally or vertically aligned, the automatic router
creates an orthogonal L-shaped connection consisting of a horizontal segment
followed by a vertical segment.

Electrical connectivity is endpoint-based:

- wire endpoints at the same coordinate are connected;
- a component terminal is connected when a wire endpoint coincides with it;
- all ground symbols are treated as the same reference node; and
- wires that only cross at intermediate points are not connected.

This rule avoids inferring an electrical junction from a visual crossing. A
shared endpoint or junction component must be used when a connection is
intended.

### 2.4 Rendering

The diagram is redrawn from application state into separate SVG groups for
wires, wire previews, component symbols, and terminals. Symbols are generated
from SVG primitives and paths. Selection indicators and enlarged transparent
hit areas improve pointer interaction without changing the exported schematic.
Component labels are repositioned according to rotation to reduce overlap.

## 3. Interaction and State Management

The editor uses three explicit interaction modes:

- **Select** for choosing, moving, editing, rotating, and deleting items;
- **Wire** for connecting component terminals; and
- **Pan** for changing the visible canvas region.

The camera is implemented through the SVG `viewBox`. Zoom is constrained from
0.35 to 3.0 times the base scale and can be anchored to the pointer or gesture
center. Panning changes the view-box origin, while fit-to-view computes the
bounding box of all components and wires and adds padding before selecting a
scale.

Before a state-changing operation, the current component and wire arrays are
serialized as an undo snapshot. Up to 50 snapshots are retained. Undo transfers
the current state to the redo stack and restores the previous snapshot; redo
performs the reverse operation.

The interface adapts at mobile, tablet, and desktop breakpoints. It supports
mouse, pointer, touch, keyboard, and pinch gestures. Modal panels use focus
management, focus trapping, ARIA states, and visible feedback. Reduced-motion
preferences are also respected.

## 4. Electrical Network Construction

Before analysis, the application converts drawing coordinates into electrical
nets. It creates a disjoint-set structure, also known as union-find:

1. every component terminal coordinate is registered;
2. the two endpoints of every wire segment are merged;
3. all ground-terminal coordinates are merged; and
4. each terminal is assigned the representative identifier of its connected
   net.

This procedure collapses any chain of wire segments into one electrical node.
Because only segment endpoints are merged, intermediate crossings remain
electrically separate by design.

## 5. DC Modeling Assumptions

The simulation performs linear, steady-state DC analysis. The implemented
models are:

| Component | DC model |
| --- | --- |
| Resistor | Linear resistance \(R>0\) |
| DC supply | Ideal independent voltage source |
| Ammeter | Ideal zero-volt branch |
| Voltmeter | Ideal open circuit; voltage is measured after solving |
| Capacitor or polarized capacitor | Open circuit at DC steady state |
| Inductor | Ideal zero-volt branch or short circuit |
| Ground | Zero-volt reference node |
| Junction | Connectivity point |

Capacitors and voltmeters do not contribute conductive branches to the MNA
matrix. A voltmeter is valid only when both of its terminals belong to nodes in
the solved conductive network.

AC supplies, switches, bulbs or loads, diodes, LEDs, and transistors are
available for schematic drawing but are not modeled by the DC solver. Their
presence causes validation to stop the simulation rather than produce an
approximate result.

At least one DC supply is required. If a connected ground is present, it defines
the zero-volt reference. Otherwise, the second terminal of the first DC supply
is used as the reference node. Multiple ideal DC sources are permitted provided
their voltage constraints are consistent.

## 6. Modified Nodal Analysis

### 6.1 Unknown vector

After selecting the reference node, the solver constructs an unknown vector
containing:

1. the voltage of every conductive non-reference node; and
2. one branch-current unknown for every DC source, ammeter, and inductor.

If there are \(n\) non-reference nodes and \(m\) ideal-voltage branches, the
linear system has \(n+m\) unknowns and is written as

\[
\begin{bmatrix}
\mathbf{G} & \mathbf{B}\\
\mathbf{B}^{T} & \mathbf{0}
\end{bmatrix}
\begin{bmatrix}
\mathbf{v}\\
\mathbf{i}
\end{bmatrix}
=
\begin{bmatrix}
\mathbf{0}\\
\mathbf{e}
\end{bmatrix}.
\]

Here, \(\mathbf{G}\) is the nodal conductance matrix, \(\mathbf{B}\) describes
the incidence of ideal-voltage branches, \(\mathbf{v}\) contains unknown node
voltages, \(\mathbf{i}\) contains ideal-branch currents, and \(\mathbf{e}\)
contains imposed branch voltages.

### 6.2 Resistor stamping

For a resistor of resistance \(R\) between nodes \(a\) and \(b\), the
conductance is

\[
g=\frac{1}{R}.
\]

The solver adds \(g\) to the diagonal entries for nodes \(a\) and \(b\), and
adds \(-g\) to the two corresponding off-diagonal entries. Entries associated
with the reference node are omitted because its voltage is fixed at zero.

### 6.3 Ideal-voltage branch stamping

For each ideal-voltage branch, the solver adds a branch-current column and a
constraint row. The positive terminal receives a coefficient of \(+1\), and the
negative terminal receives a coefficient of \(-1\). The right-hand-side value
is:

- the entered source voltage for a DC supply; or
- zero for an ideal ammeter or inductor.

The first terminal in the DC-supply definition is treated as positive, matching
the symbol's long plate and plus sign. Rotating the symbol rotates its terminal
polarity with it.

## 7. Numerical Solution

The assembled linear system is solved by Gaussian elimination with partial
pivoting:

1. for each column, the row with the largest absolute candidate pivot is
   selected;
2. rows are exchanged when required;
3. entries below the pivot are eliminated; and
4. the unknowns are recovered by back substitution.

A pivot magnitude below \(10^{-12}\) is treated as singular. A singular system
can result from floating nodes, open conductive paths, redundant or conflicting
ideal constraints, or other topologies without a unique DC solution.

The implementation is intended for small instructional networks. It uses dense
JavaScript arrays rather than the sparse-matrix techniques normally used by
large-scale circuit simulators.

## 8. Value Parsing and Measurement Extraction

Entered resistor and source values are converted to base SI units before matrix
assembly. The parser accepts decimal and scientific notation followed by an
optional engineering prefix:

| Prefix | Multiplier |
| --- | ---: |
| `p` | \(10^{-12}\) |
| `n` | \(10^{-9}\) |
| `u` or `µ` | \(10^{-6}\) |
| `m` | \(10^{-3}\) |
| `k` or `K` | \(10^{3}\) |
| `M` | \(10^{6}\) |
| `G` | \(10^{9}\) |
| `T` | \(10^{12}\) |

Text following the recognized numeric value and prefix can serve as a displayed
unit. Resistance values must be greater than zero.

After the system is solved:

- voltmeter output is the signed potential difference from terminal 1 to
  terminal 2, \(V_\text{meter}=V_1-V_2\); and
- ammeter output is the magnitude of the solved current through its zero-volt
  branch.

Results are formatted to three decimal places with an appropriate nano, micro,
milli, base, kilo, or mega prefix.

## 9. Validation and Error Handling

Validation occurs before and during solution. The application reports an error
when it detects:

- no DC supply;
- an AC supply;
- a component type not supported by the DC model;
- an empty, malformed, or non-finite source or resistance value;
- a resistance less than or equal to zero;
- a ground symbol not connected to the conductive DC network;
- a voltmeter not connected across two solved nodes; or
- a singular matrix associated with a floating, open, or conflicting circuit.

Successful results are associated with a serialization of the current
components and wires. Any subsequent schematic change invalidates and hides the
previous result, preventing stale readings from being presented as current.

## 10. Persistence and Export

Project saving serializes the component array, wire array, and reference-label
counters to a JSON file. Loading parses this structure and restores the editable
schematic in the browser.

For graphical export, the application clones the current SVG, removes terminal
markers, enlarged wire hit areas, and selection styling, then crops the output
to the schematic bounds with padding. SVG export preserves vector geometry.
PNG export rasterizes the cleaned SVG at twice its displayed dimensions on a
white canvas. Clipboard export uses the same PNG-generation process.

The current PDF command invokes the PNG export pathway; it does not create a
native PDF document.

## 11. Scope and Limitations

The methodology is intentionally narrower than a general-purpose SPICE
workflow. It does not include transient analysis, AC small-signal analysis,
frequency response, nonlinear iteration, semiconductor device equations,
switch state modeling, thermal effects, component tolerances, parasitics, or
uncertainty propagation. Capacitors and inductors are represented only by their
ideal DC steady-state limits.

The application is therefore appropriate for schematic communication, Ohm's
law, series and parallel resistor networks, Kirchhoff's laws, ideal meter
placement, and introductory DC network analysis. Results should be interpreted
within these assumptions and should not replace instrument measurements or
specialized circuit simulation when device-level or time-dependent behavior is
important.

## 12. Implementation Review Basis

This methodology was prepared from a direct review of the application's
implemented behavior in `index.html`, including its component definitions,
coordinate transformations, wire routing, union-find network construction, MNA
matrix assembly, numerical solver, validation logic, measurement formatting,
state persistence, responsive interaction handling, and export functions.
