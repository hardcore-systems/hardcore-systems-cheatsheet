# Modern Power System Simulation & Grid Modeling Cheatsheet

## J-Ladder Hierarchical Model

### L0 One-Line Essence
The essence of power system simulation lies in constructing mathematical models comprising non-linear algebraic equations (steady-state power flow) and high-dimensional differential-algebraic equations (transient stability), solved via numerical iteration methods (Newton-Raphson / implicit integration) and optimization engines for large-scale grid states.

### L1 Four-Sentence Logic
1. **Steady-state Topology**: Based on bus, branch, generator, and ZIP load components, the complex admittance matrix $Y_{bus}$ is constructed to formulate power balance equations.
2. **Non-linear Algebraic Flow**: Newton-Raphson iteration computes the Jacobian matrix gradient or simplifies via fast decoupled models to solve bus voltage magnitude and phase angles.
3. **Electro-mechanical Dynamics**: Integrating the generator Swing Equation and controllers (AVR, Governor) yields DAE systems simulating transient behavior under fault conditions.
4. **Active Security Defense**: Small-signal stability is verified via state-space eigenvalues, while extreme perturbations trigger safety margins (CCT) and load shedding (UFLS/UVLS) to prevent grid collapse.

### L2 Core Data Flow
`Grid Topology & Parameters` ➜ `Construct Y_bus Matrix` ➜ `Newton-Raphson Power Flow (V, theta)` ➜ `OPF Economic Dispatch` ➜ `Fault Perturbation (3-phase Short)` ➜ `Swing Equation Integration` ➜ `Voltage/Frequency Monitoring` ➜ `Active Protection & Defense`

---

## 📂 Core Knowledge Cards (Cards 1-28)

### Card 1: Bus Node Modeling
*   **Theory**: Foundation of load flow. Buses are classified into three types based on boundary conditions: PQ bus (known load injection P & Q), PV bus (known generation P & voltage magnitude V, controlling Q to maintain V), and Slack bus (reference bus, known V & phase $\theta=0$, balances system losses).
*   **Details**: Variables to solve: PQ bus $(V_i, \theta_i)$; PV bus $(Q_i, \theta_i)$. Real/reactive power at Slack bus $(P_s, Q_s)$ are solved directly post-convergence.
*   **Trade-off**: Generator Q limits constrain PV buses. If reactive power $Q$ violates its boundary during iteration, the PV bus temporarily degrades into a PQ bus using the limiting value.

### Card 2: Pi equivalent Line Model
*   **Theory**: Models short/medium lines. Consists of a series impedance $Z = R + jX$ and shunt susceptances $Y/2 = jB/2$ split equally at both terminals.
*   **Details**: Series branch admittance $Y_{series} = 1/Z = G_{series} + jB_{series}$. Shunt charging models reactive support.
*   **Trade-off**: For long transmission lines (>150km), lumped-parameter models fail, requiring hyperbolic correction: $Z' = Z \frac{\sinh(\gamma l)}{\gamma l}$ to prevent voltage profile distortions.

### Card 3: Ybus Admittance Matrix
*   **Theory**: Describes grid connectivity. Grid size $N \times N$. Diagonal element $Y_{ii}$ is the sum of admittances of branches connected to bus $i$. Off-diagonal $Y_{ij}$ is the negative branch admittance between $i$ and $j$.
*   **Details**: Formulations: $Y_{ii} = y_{i0} + \sum y_{ik}$ and $Y_{ij} = -y_{ij}$.
*   **Trade-off**: High-voltage grids are sparse (>99% zero entries). Storing $Y_{bus}$ in CSR formats and employing sparse solvers (e.g., KLU, UMFPACK) is mandatory for memory efficiency.

### Card 4: Generator Limit Cost Curve
*   **Theory**: Defines active power production costs and secure operating zones. Operational generation cost is modeled via quadratic polynomial: $C_i(P_i) = a_i P_i^2 + b_i P_i + c_i$.
*   **Details**: Operational bounds: $P_{min} \leq P \leq P_{max}$ and $Q_{min}(P) \leq Q \leq Q_{max}(P)$, bounded by thermal limits and excitation rotor constraints.
*   **Trade-off**: Neglecting the P-Q limit coupling results in unrealistic OPF dispatches that can trigger generator excitation trips and voltage collapse in physical operations.

### Card 5: ZIP Load Model
*   **Theory**: Captures static load dependency on voltage magnitude. Comprises Constant Impedance (Z), Constant Current (I), and Constant Power (P) terms.
*   **Details**: Equations: $P = P_0 [ a_1(V/V_0)^2 + a_2(V/V_0) + a_3 ]$ with $a_1 + a_2 + a_3 = 1$.
*   **Trade-off**: Simply modeling loads as constant power ($a_3=1$) during deep voltage drops causes severe numerical divergence in transient solvers and overestimates collapse speed.

### Card 6: Power Flow Equations
*   **Theory**: Non-linear equations expressing active and reactive power balance at each node.
*   **Details**: Polar coordinates balance:
    $$P_i = V_i \sum V_j (G_{ij}\cos\theta_{ij} + B_{ij}\sin\theta_{ij})$$
    $$Q_i = V_i \sum V_j (G_{ij}\sin\theta_{ij} - B_{ij}\cos\theta_{ij})$$
*   **Trade-off**: Non-convex system requiring a good initial guess (flat start: $V_i=1.0, \theta_i=0$) to guarantee convergence to a physically meaningful solution.

### Card 7: Newton-Raphson Method
*   **Theory**: Linearizes power flow equations via Taylor expansion, solving variables iteratively using derivatives of the Jacobian matrix.
*   **Details**: Step: $\begin{bmatrix} \Delta P \\ \Delta Q \end{bmatrix} = J \begin{bmatrix} \Delta \theta \\ \Delta V / V \end{bmatrix}$. Updated variables: $\theta^{(k+1)} = \theta^{(k)} + \Delta\theta$.
*   **Trade-off**: Recomputing and factorizing the Jacobian at each iteration is computationally heavy. Near static stability limits, the Jacobian becomes singular, causing divergence.

### Card 8: Fast Decoupled Power Flow
*   **Theory**: Simplifies Jacobian under the physical assumption that $X \gg R$, which decouples P-$\theta$ and Q-$V$ channels in high-voltage grids.
*   **Details**: Form: $\Delta P / V = B' \Delta \theta$ and $\Delta Q / V = B'' \Delta V$. Constant matrices $B'$ and $B''$ are factorized once.
*   **Trade-off**: Extremely fast and robust for sparse systems, but fails or converges poorly in distribution grids where the $R/X$ ratio is high.

### Card 9: DC Power Flow
*   **Theory**: A completely linearized formulation assuming flat voltages ($V=1.0$), negligible resistance ($R \approx 0$), and small angle differences.
*   **Details**: Linear relation: $P = B \theta$. Line power flow: $P_{ij} = b_{ij}(\theta_i - \theta_j)$.
*   **Trade-off**: Ignores reactive power, voltage variations, and network losses. Unsuitable for voltage stability assessments but excellent for rapid market clearing.

### Card 10: Optimal Power Flow (OPF)
*   **Theory**: Minimizes generation costs or transmission losses subject to operational bounds.
*   **Details**: Form: $\min \sum C_i(P_i)$ subject to non-linear flow equality $h(V,\theta)=0$ and inequality constraints ($V_{min} \leq V \leq V_{max}$, branch flow limits $S \leq S_{max}$).
*   **Trade-off**: A highly non-convex NP-hard problem. Numerical interior-point methods require careful initializations to avoid stalling in local minima.

### Card 11: Swing Equation
*   **Theory**: Models rotor dynamics of synchronous machines when mechanical input and electromagnetic output power are imbalanced.
*   **Details**: Equations:
    $$\frac{d\delta}{dt} = \omega_b (\omega - 1)$$
    $$2H \frac{d\omega}{dt} = P_m - P_e - D(\omega - 1)$$
*   **Trade-off**: Ideal for transient screens, but overestimating damping $D$ masks low-frequency inter-area oscillation risks.

### Card 12: Detailed Generator Model
*   **Theory**: Models electromotive forces and magnetic flux paths. Varies from 3rd-order (transient EMF) to 6th-order (subtransient double-axis) formulations.
*   **Details**: Tracks state variables: $\delta, \omega, e'_d, e'_q, e''_d, e''_q$.
*   **Trade-off**: Accurately simulates machine-grid dynamics but increases parameter complexity, demanding microsecond simulation time steps.

### Card 13: AVR & Governor Controls
*   **Theory**: AVR controls field excitation to hold terminal voltage; Governor regulates mechanical input power to control speed and grid frequency.
*   **Details**: AVR model (e.g., IEEE DC1A): $T_A \dot{E}_{fd} = -E_{fd} + K_A(V_{ref} - V_t)$. Governor: $T_g \dot{P}_{valve} = -P_{valve} + P_{set} - \frac{1}{R_d}(\omega - 1)$.
*   **Trade-off**: Excessive AVR gain ($K_A$) introduces negative damping (causing oscillation), while governor valve dead-bands lag during sudden system frequency drops.

### Card 14: DAE Integration Algorithms
*   **Theory**: Power dynamics form a rigid DAE set: $dx/dt = f(x, y)$ (dynamic states), $g(x, y) = 0$ (algebraic network constraints).
*   **Details**: Solved via implicit methods like the Trapezoidal Rule: $x_{n+1} = x_n + \frac{\Delta t}{2}[f(x_n, y_n) + f(x_{n+1}, y_{n+1})]$.
*   **Trade-off**: DAE systems are numerically stiff. Explicit integration methods (e.g., RK4) require tiny steps to avoid numerical explosion.

### Card 15: Network Interface Admittance
*   **Theory**: Bridges machine equations and the grid admittance equations by modeling generators as voltage sources behind transient reactances.
*   **Details**: Norton equivalent current: $I_{inj} = \frac{E' e^{j(\delta - \pi/2)}}{jX'_d}$. Augmented admittance: $I = (Y_{bus} + Y_{gen}) V$.
*   **Trade-off**: Rapid change in generator states during near-faults can trigger numerical current spikes if time-step convergence is not strictly bounded.

### Card 16: Small-Signal Stability
*   **Theory**: Evaluates robustness against minor load fluctuations by linearizing the DAE around the current operating point.
*   **Details**: Form: $\Delta \dot{x} = A \Delta x$. Eigenvalues $\lambda_i = \sigma_i \pm j\omega_i$ dictate stability. Damping ratio: $\zeta = \frac{-\sigma_i}{\sqrt{\sigma_i^2 + \omega_i^2}}$.
*   **Trade-off**: Damping ratios below 3% trigger low-frequency power oscillations. Ignoring multi-machine interactions can result in unpredicted inter-area oscillations.

### Card 17: Equal Area Criterion (EAC)
*   **Theory**: Graphical analysis of transient stability for single-machine systems under large disturbances.
*   **Details**: Assesses rotor swing angle. Accelerating area $A_{acc} = \int (P_m - P_e^{fault}) d\delta$ must be compensated by decelerating area $A_{dec} = \int (P_e^{post} - P_m) d\delta$.
*   **Trade-off**: Strictly limited to SMIB systems. Multi-machine networks require full time-domain simulations.

### Card 18: Critical Clearing Time (CCT)
*   **Theory**: Maximum duration a fault can persist without causing the generator to lose synchronism.
*   **Details**: Derived by integrating the swing equation: $\int_{\delta_0}^{\delta_{cct}} (P_m - P_{e,fault}) d\delta = \int_{\delta_{cct}}^{\delta_{max}} (P_{e,post} - P_m) d\delta$.
*   **Trade-off**: Low-inertia renewable grids compress the CCT drastically, requiring ultra-fast sub-cycle relaying and circuit breaker clearing.

### Card 19: Droop & AGC Controls
*   **Theory**: Two levels of frequency defense. Primary droop control regulates speed in seconds; Secondary AGC adjusts base generation setpoints in minutes.
*   **Details**: Droop equation: $P_{gen} - P_{set} = -\frac{1}{R_d}(f - f_0)$. AGC balances Area Control Error: $ACE = \Delta P_{interchange} + B \Delta f$.
*   **Trade-off**: Droop control is proportional (steady-state error remains), while AGC is integral (zero error) but acts too slowly to arrest the initial rate of frequency decline.

### Card 20: BESS SoC Modeling
*   **Theory**: Battery Energy Storage Systems act as voltage sources. The State of Charge (SoC) tracks remaining energy capacity.
*   **Details**: SoC integration: $SoC(t) = SoC(0) - \frac{\eta}{E_{nominal}} \int I_{batt}(\tau) d\tau$. Terminal voltage: $V_{batt} = V_{oc}(SoC) - I_{batt} R_{int}$.
*   **Trade-off**: Ignoring internal resistance $R_{internal}$ underestimates thermal limits, risking battery disconnects under high-load black start scenarios.

### Card 21: Inverter Droop Control
*   **Theory**: Mimics physical generator behavior by establishing voltage and frequency references in grid-connected power electronics.
*   **Details**: Equations: $\omega = \omega_{ref} - m_p (P_{calc} - P_{ref})$ and $V = V_{ref} - n_q (Q_{calc} - Q_{ref})$.
*   **Trade-off**: Lacking physical rotor mass, inverter droop controls have very low damping. Too low a low-pass filter cutoff in measurements can induce high-frequency oscillations.

### Card 22: Active Distribution Grid (ADN)
*   **Theory**: High penetration of single-phase solar generators forces modeling of unbalanced three-phase distribution topologies.
*   **Details**: Uses a $3 \times 3$ impedance matrix for branches. State estimation uses Weighted Least Squares (WLS) optimization: $\min [z - h(x)]^T R^{-1} [z - h(x)]$.
*   **Trade-off**: Three-phase estimation increases complexity by 9x. Low telemetry density causes system observability failures.

### Card 23: Virtual Synchronous Generator (VSG)
*   **Theory**: Implements swing equations in inverter controls to provide virtual inertia and damping to low-inertia grids.
*   **Details**: Control loops: $J_v \dot{\omega}_v = T_m - T_e - D_v(\omega_v - \omega_0)$.
*   **Trade-off**: High virtual inertia improves frequency stability, but excessive transient current demands can trigger overcurrent shutdowns of the inverter IGBTs.

### Card 24: Islanding & Black Start
*   **Theory**: Islanding detection isolates microgrids for safety; Black Start re-energizes a dead grid using local battery grids and virtual generators.
*   **Details**: Active frequency drift shifts phase during islanding to trigger trips. Black Start uses Grid-Forming inverters to establish voltage reference without grid presence.
*   **Trade-off**: Active detection blind spots (NDZ) occur when load and generation match. Cold-load transformer inrush currents can easily collapse black start inverters.

### Card 25: Wind DFIG & PMSG LVRT
*   **Theory**: Ensures wind turbines remain connected during deep grid faults to avoid cascading generation loss.
*   **Details**: DFIG clamps rotor overcurrents via a Crowbar circuit, turning it into an induction machine. PMSG dissipates excess power in a DC Chopper to protect DC link caps.
*   **Trade-off**: Activating the DFIG Crowbar consumes significant reactive power from the grid, degrading local voltage recovery during the fault.

### Card 26: Solar MPPT Algorithms
*   **Theory**: Continuously tracks maximum output point of PV arrays under changing solar irradiance.
*   **Details**: Perturb and Observe (P&O) steps voltage periodically to maximize power; Incremental Conductance (INC) solves $dI/dV = -I/V$.
*   **Trade-off**: P&O oscillates around peak and fails under fast weather transients. Partially shaded arrays exhibit multi-peak curves, requiring global algorithms (e.g., PSO).

### Card 27: LMP Pricing Structure
*   **Theory**: Price of supplying the next MW of load at a specific node, reflecting marginal costs of congestion and losses.
*   **Details**: Composition: $LMP_i = \lambda_{system} + \mu_{congestion,i} + \eta_{loss,i}$. Congestion is calculated using Shift Factors (SF).
*   **Trade-off**: Grid congestion splits prices, creating congestion surpluses. Poor Financial Transmission Right (FTR) distribution penalizes consumers in congested zones.

### Card 28: Inertia Dilution
*   **Theory**: Inverter-based resources do not contribute physical inertia, diluting system equivalent inertia as fossil plants are decommissioned.
*   **Details**: Rate of Change of Frequency (RoCoF) limit: $RoCoF = \frac{f_0 \Delta P}{2 H_{system}}$.
*   **Trade-off**: High RoCoF triggers fast protection trips. Mitigating it requires expensive synchronous condensers or reserving active headroom in wind/solar grids.
