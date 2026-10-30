# Lecture Notes: IBM Heron Hardware and Fractional Gates

Oct 30, 2026 · @Laxmikanta Sutar

## 1. Scope, prerequisites and notation

These notes derive, from linear algebra upward, why IBM Heron QPUs can run RX(θ) and RZZ(θ) as single native gates, what that saves, and where it fails. They follow three IBM Quantum pages: [View backend details](https://quantum.cloud.ibm.com/docs/en/guides/qpu-information), [Fractional gates](https://quantum.cloud.ibm.com/docs/en/guides/fractional-gates) and [Quantum kernels with fractional gates](https://quantum.cloud.ibm.com/docs/en/tutorials/fractional-gates), read on 2026-10-30 against Qiskit 2.5.x and qiskit-ibm-runtime 0.49.

Prerequisites: complex numbers, matrix products and eigenvalues. Everything else (Pauli algebra, SU(2), the Bloch sphere, the two-qubit KAK decomposition) is derived where it is needed. Quantum-kernel background is not required: the kernel example is treated purely as a circuit with a known gate pattern.

| Symbol | Meaning |
| --- | --- |
| X, Y, Z | Pauli matrices |
| I | identity matrix of the size implied by context |
| RX(θ), RY(θ), RZ(θ) | single-qubit rotations exp(−iθP/2) for P = X, Y, Z |
| RZZ(θ) | two-qubit rotation exp(−iθ Z⊗Z/2) |
| A ≃ B | A equals B up to a global phase factor |
| a, b | first and second qubit; basis order 00, 01, 10, 11 (first digit is qubit a) |
| SX | the √X gate, equal to RX(π/2) up to phase |

All angles are in radians. Every identity in sections 3 to 6, 8 and 9 was also checked numerically with random angles (numpy, 2026-10-30), so the algebra is machine-verified as well as derived.

## 2. Part 1: reading IBM QPU hardware details

A QPU card has four layers of information: static identity, the supported instruction set, per-qubit physics, and per-instruction duration and error. Qiskit exposes every layer from the backend object, and the IBM Quantum Platform Compute resources page shows the same data. Source for this whole section: [View backend details](https://quantum.cloud.ibm.com/docs/en/guides/qpu-information).

### 2.1 Listing and filtering backends

QPU names start with ibm\_ and carry a city name; the city does not say where the QPU is hosted. Only backends available on your selected instance or region are returned.

```python
from qiskit_ibm_runtime import QiskitRuntimeService

service = QiskitRuntimeService()
service.backends()                 # every QPU visible on this instance
service.backend("ibm_fez")         # one QPU by name (note: singular)

# filter by configuration or status
service.backends(simulator=False, operational=True, min_num_qubits=100)

# same filters, returns the QPU with the fewest pending jobs
service.least_busy(operational=True, min_num_qubits=100)
```

The docs example returned ibm\_boston, ibm\_fez, ibm\_kingston, ibm\_marrakesh, ibm\_miami, ibm\_phoenix and ibm\_pittsburgh. For custom logic, pass a function to the filters argument that takes a backend and returns True or False.

### 2.2 Static information and versioning

Static attributes change rarely and are plain attributes of the backend object.

```python
backend = service.least_busy(operational=True, simulator=False)
backend.name, backend.backend_version, backend.num_qubits, backend.processor_type
```

For ibm\_fez the docs show version 1.3.37, 156 programmable qubits and processor type family Heron, revision 2. The family (the bird name) fixes the coupling topology and the approximate qubit count; the revision tracks design iterations within a family.

Every QPU carries a version X.Y.Z (major.minor.revision). A circuit compiled for one version is guaranteed to run on that QPU. If only the revision changes, compiled circuits keep running; if the major or minor number changes, they may or may not.

| Level | Typical triggers | Compiled circuit still guaranteed to run? |
| --- | --- | --- |
| Major | Sample change; major control-electronics change; relocation that changes behaviour | No |
| Minor | Warm-up or cool-down cycle; electronics swap that affects operation; a gate dropped for a time because of calibration trouble; CNOT direction change | No |
| Revision | QPU software update; manual calibration to improve fidelity; small electronics change with no operational effect | Yes |

Practical rule: pin the full X.Y.Z with a job's results, and recompile after any major or minor change.

### 2.3 Supported instructions

Each processor family natively supports a small instruction set. The transpiler must translate every gate in your circuit into this set before the circuit can run.

```python
backend.supported_instructions
# ibm_fez: ['cz','id','delay','measure','measure_reset','measure_2','measure_reset_2',
#           'reset','reset_2','rz','sx','x','if_else','store','xslow']
```

| Category | Instructions |
| --- | --- |
| Single-qubit gates | RZ, SX, X, XSlow, ID, delay |
| Two-qubit gates | CZ (Heron, Nighthawk), ECR (Eagle) |
| Fractional gates (opt-in) | RX(θ) single-qubit, RZZ(θ) two-qubit |
| Non-unitary | measure, reset, measure\_*, reset\_*, measure\_reset\*, delay |
| Control flow | if\_else (classical feedforward), store |

Note what is missing from the ibm\_fez output: rx and rzz. Fractional gates appear in the Target only when the backend is loaded with use\_fractional\_gates=True (section 2.7).

### 2.4 Qubit properties and the physics behind them

Qubit properties come from periodic benchmarking jobs. `backend.qubit_properties(q)` returns T1, T2 and frequency; `backend.properties().qubit_property(q)` returns a dictionary of key to (value, timestamp).

**T1, energy relaxation time.** The excited-state population decays exponentially, so after time t a qubit prepared in |1⟩ is still excited with probability exp(−t/T1).

**T2, dephasing time.** The off-diagonal coherence of a superposition decays as exp(−t/T2). IBM reports T2 from a Hahn echo sequence, which cancels slow static detuning, so it includes relaxation plus pure dephasing.

```latex
p_1(t)=p_1(0)\,e^{-t/T_1},\qquad |\rho_{01}(t)|=|\rho_{01}(0)|\,e^{-t/T_2},\qquad \frac{1}{T_2}=\frac{1}{2T_1}+\frac{1}{T_\varphi}\;\Rightarrow\;T_2\le 2T_1
```

The bound T2 ≤ 2·T1 holds because relaxation alone already destroys coherence at half the rate; any extra pure dephasing (rate 1/Tφ) only shortens T2. Docs example, qubit 1: T1 ≈ 207.7 µs, T2 ≈ 246.1 µs, consistent with the bound (2·T1 ≈ 415 µs).

**Readout (SPAM) quantities.** Prepare |0⟩ or |1⟩, measure many times, and tabulate the assignment matrix. The two off-diagonal entries are prob\_meas1\_prep0 = P(1|0) and prob\_meas0\_prep1 = P(0|1); readout\_error is their mean.

```latex
A=\begin{pmatrix}P(0|0)&P(0|1)\\P(1|0)&P(1|1)\end{pmatrix},\qquad \varepsilon_{\mathrm{ro}}=\tfrac12\big[P(0|1)+P(1|0)\big]
```

Check with the docs example for qubit 155: P(0|1) = 0.766 and P(1|0) = 0.071 give (0.766 + 0.071)/2 = 0.419, the reported readout error. That qubit also has T2 ≈ 20.1 µs against T1 ≈ 108.6 µs, so one qubit can be far worse than the median: always inspect per-qubit values before choosing a layout.

Two more entries: readout\_length is the time from the start of the measurement pulse to the digitised result (1.66 µs in the docs example), and init\_error is the residual |1⟩ population after the reset and rep-delay procedure. init\_error differs slightly from prob\_meas1\_prep0, which also includes the discrimination step; with perfect measurement the two coincide.

### 2.5 Instruction properties, benchmarks and error definitions

`backend.target` is a qiskit.transpiler.Target: for every instruction and qubit tuple it stores a duration (seconds in Qiskit, nanoseconds on the Platform page) and an error.

```python
edge = next(iter(backend.target["cz"]))
backend.target["cz"][edge]            # duration=6.8e-08 s (68 ns), error≈0.0077
backend.target["measure"][(0,)]       # duration=1.66e-06 s, error≈0.0684
```

Gate errors come from randomized benchmarking (RB). Random Clifford sequences of length m return the initial state with probability F(m) = A·p^m + B; the fitted decay p gives the average error per Clifford, with d = 2^n the Hilbert-space dimension.

```latex
F(m)=A\,p^{m}+B,\qquad r=\frac{d-1}{d}\,(1-p)\quad(\text{average gate error}),\qquad d=2^{n}
```

| Reported value | Definition from the docs |
| --- | --- |
| 2Q error (per edge) | Two-qubit error on one edge, from the same batch as the median; the best value is the lowest edge |
| 2Q error (median) | Average gate fidelity of the two-qubit gate from RB, measured in isolated batches with at least two qubits between active edges; the figure includes the layer of single-qubit Cliffords |
| 2Q error (layered) | Error per layered gate (EPLG) on a 100-qubit chain, derived from layer fidelity (LF) |
| RZZ error (Heron) | RZZ error averaged over angles, using an RB variant for arbitrary unitaries; needs use\_fractional\_gates=True to read from the Target |
| SX, X, ID, RX error | One-qubit RB; SX, ID and X are assumed to share one error; RX is reported with the same value |
| RZ error | Always 0: RZ is a virtual gate done in software |
| error = 1 | Stale: benchmarking failed for several days; the value is undefined, not proof of a dead qubit or edge |

**EPLG derivation.** The layer fidelity LF\_N is the process fidelity of one layer of two-qubit gates covering an N-qubit chain, which contains N−1 gates. Assuming errors multiply, the per-gate process fidelity is LF^(1/(N−1)). Process infidelity converts to average-gate infidelity by the factor d/(d+1) with d = 4 for two qubits, i.e. 4/5.

```latex
\mathrm{EPLG}_{N}=\frac{4}{5}\Big(1-\mathrm{LF}_{N}^{\,1/(N-1)}\Big)
```

Illustrative value: LF\_100 = 0.5 gives LF^(1/99) = exp(ln 0.5 / 99) ≈ 0.99302, so EPLG\_100 ≈ 0.8 × 0.00698 ≈ 0.0056. EPLG is the honest number for "many gates at once", because the median per-edge figure is measured in isolation and misses crosstalk.

### 2.6 Throughput, status and topology

- MCPS (maximum circuits per second): throughput for a minimal circuit (Hadamard plus one measurement) including measurement, reset and reinitialization.
- CLOPS\_h (circuit layer operations per second): how many layers of a 100 by 100 hardware-aware circuit the QPU executes per unit time.
- Status: BackendStatus gives Active, Paused or Offline plus the pending-jobs count (`backend.status().status_msg`, `.pending_jobs`).
- Topology (coupling map): which qubit pairs support a two-qubit gate; `plot_gate_map(backend)` draws it.

### 2.7 Fetching Heron backends with fractional gates

Fractional gates are off by default. The flag use\_fractional\_gates=True makes the backend's Target include RX(θ) and RZZ(θ).

```python
# Which QPUs support fractional gates at all?
for b in service.backends(use_fractional_gates=True):
    print(b.name, b.processor_type, "rzz" in b.target.operation_names)

# Least busy QPU that supports them (qiskit-ibm-runtime 0.48 or newer)
service.least_busy(fractional_gates=True)

# Same QPU with and without the flag
backend_f = service.backend("ibm_marrakesh", use_fractional_gates=True)
backend_c = service.backend("ibm_marrakesh")
print(backend_c.basis_gates)   # ['cz','id','rz','sx','x']
print(backend_f.basis_gates)   # ['cz','id','rx','rz','rzz','sx','x']
```

The printed basis gates are the docs output for ibm\_marrakesh. The difference between the two lists is exactly rx and rzz. A fake backend supports fractional gates only if the matching real backend does, and fake backends need `fake_backend.refresh(service=service, use_fractional_gates=True)` to switch them on.

## 3. Foundations: qubits, Pauli algebra and rotation gates

Every gate in this session is the exponential of a Pauli operator, and one identity, exp(−iθP/2) = cos(θ/2)·I − i·sin(θ/2)·P, generates RX, RY, RZ and RZZ. This section proves that identity and the four rules (periodicity, additivity, conjugation, parity action) that the later sections reuse.

### 3.1 State space and gates

A qubit state is a unit vector in C², defined up to a global phase, because a phase e^{iφ} multiplies every amplitude and changes no measurement probability. A gate is a 2×2 unitary U (U†U = I). Multiplying U by a phase changes nothing observable, so we may fix det U = 1; the resulting group is SU(2). Two qubits live in C² ⊗ C² = C⁴ and gates are 4×4 unitaries, with the Kronecker product A⊗B acting as A on qubit a and B on qubit b.

### 3.2 Pauli matrices

X, Y and Z are Hermitian, traceless, have eigenvalues +1 and −1, and satisfy:

```latex
X^2=Y^2=Z^2=I,\qquad XY=iZ,\;\;YZ=iX,\;\;ZX=iY,\qquad XY=-YX\;(\text{any two distinct Paulis anticommute})
```

The property P² = I is the only fact needed to exponentiate them. It holds for X, Y, Z, and also for tensor products such as Z⊗Z, because (Z⊗Z)² = Z²⊗Z² = I⊗I.

### 3.3 The exponential identity

For any matrix P with P² = I, the powers alternate: P^(2m) = I and P^(2m+1) = P. Split the exponential series into even and odd terms:

```latex
e^{-i\alpha P}=\sum_{k\ge0}\frac{(-i\alpha)^k}{k!}P^k=\underbrace{\sum_{m\ge0}\frac{(-1)^m\alpha^{2m}}{(2m)!}}_{\cos\alpha}\,I\;-\;i\,\underbrace{\sum_{m\ge0}\frac{(-1)^m\alpha^{2m+1}}{(2m+1)!}}_{\sin\alpha}\,P
```

Setting α = θ/2 and naming the rotation by its generator gives R\_P(θ) = exp(−iθP/2) = cos(θ/2)·I − i·sin(θ/2)·P. The three single-qubit cases are:

```latex
R_X(\theta)=\begin{pmatrix}\cos\frac\theta2&-i\sin\frac\theta2\\-i\sin\frac\theta2&\cos\frac\theta2\end{pmatrix},\quad
R_Y(\theta)=\begin{pmatrix}\cos\frac\theta2&-\sin\frac\theta2\\\sin\frac\theta2&\cos\frac\theta2\end{pmatrix},\quad
R_Z(\theta)=\begin{pmatrix}e^{-i\theta/2}&0\\0&e^{i\theta/2}\end{pmatrix}
```

### 3.4 The two-qubit case: RZZ

Z⊗Z is diagonal in the computational basis with eigenvalue (−1)^(parity): +1 on 00 and 11, −1 on 01 and 10. Applying the same identity gives a diagonal gate:

```latex
R_{ZZ}(\theta)=\cos\tfrac\theta2\,I-i\sin\tfrac\theta2\,Z\otimes Z=\mathrm{diag}\big(e^{-i\theta/2},\,e^{+i\theta/2},\,e^{+i\theta/2},\,e^{-i\theta/2}\big)
```

Reading the diagonal: RZZ(θ) gives states of even parity the phase e^{−iθ/2} and odd parity e^{+iθ/2}, a relative phase of θ between the two parity sectors. That is why it is entangling unless θ is a multiple of π, and why it is the natural building block of Ising-type Hamiltonians and ZZ feature maps.

### 3.5 Four rules used later

**Additivity.** Rotations about the same generator commute and add: R\_P(α)·R\_P(β) = R\_P(α+β), since both are functions of the same matrix P.

**Periodicity.** Put θ = 2π in the identity: cos π = −1 and sin π = 0, so R\_P(2π) = −I. Hence R\_P(θ+2π) = −R\_P(θ), a global phase, and R\_P(θ+4π) = R\_P(θ). The matrix period is 4π; the physical period is 2π. Also R\_P(π) = −iP.

**Conjugation.** For any unitary U, U·f(P)·U† = f(U P U†), so U·R\_P(θ)·U† = exp(−iθ(UPU†)/2). If U P U† = −P the sign of the angle flips; if U P U† = P nothing changes.

```latex
R_P(\theta+2\pi)=-R_P(\theta),\qquad R_P(\pi)=-iP,\qquad XZX=-Z\Rightarrow XR_Z(\theta)X=R_Z(-\theta),\qquad HXH=Z\Rightarrow HR_X(\theta)H=R_Z(\theta)
```

**Native-gate relations.** SX is RX(π/2) up to a phase, X = RX(π) up to a phase (R\_X(π) = −iX), and the Hadamard is H = (X+Z)/√2.

```latex
\mathrm{SX}=\tfrac12\begin{pmatrix}1+i&1-i\\1-i&1+i\end{pmatrix}=e^{i\pi/4}R_X(\pi/2),\qquad \mathrm{CZ}=\mathrm{diag}(1,1,1,-1)
```

## 4. Single-qubit geometry: why RX(θ) costs two SX gates

Every single-qubit gate is a rotation of the Bloch sphere, and the conventional IBM basis (RZ, SX, X) can build an arbitrary RX(θ) only from two SX pulses with RZ frame changes between them. This section proves that, which is exactly the cost the fractional RX(θ) gate removes.

### 4.1 The Bloch sphere

Write a pure state with a polar angle ϑ and an azimuth φ. Its Bloch vector has components equal to the expectation values of the Pauli matrices, and every point of the unit sphere is a state.

```latex
|\psi\rangle=\cos\tfrac{\vartheta}{2}|0\rangle+e^{i\varphi}\sin\tfrac{\vartheta}{2}|1\rangle\;\longleftrightarrow\;\vec r=(\sin\vartheta\cos\varphi,\;\sin\vartheta\sin\varphi,\;\cos\vartheta),\qquad r_k=\langle\psi|\sigma_k|\psi\rangle
```

### 4.2 From SU(2) to rotations of 3-space

A unitary U acts on the Bloch vector by conjugation: U (r·σ) U† = (𝓡r)·σ for a 3×3 rotation matrix 𝓡. For U = exp(−iθ n·σ/2) the two eigenstates of the generator n·σ (the points ±n on the sphere) only pick up phases e^{∓iθ/2}. Their relative phase is θ, and a relative phase is an azimuth around the axis n, so 𝓡 is the rotation by θ about n.

```latex
U\,(\vec r\cdot\vec\sigma)\,U^\dagger=(\mathcal R_U\vec r)\cdot\vec\sigma,\qquad U=e^{-i\theta\,\hat n\cdot\vec\sigma/2}\;\Rightarrow\;\mathcal R_U=\text{rotation by }\theta\text{ about }\hat n,\qquad \mathcal R_{-U}=\mathcal R_U
```

The last equation is the double cover: U and −U give the same rotation, which is why R\_P(θ+2π) = −R\_P(θ) is physically the same gate (section 3.5). A useful special case: RX(θ) rotates the vector by θ in the y–z plane and sends the north pole |0⟩ to (0, −sin θ, cos θ); for θ = π/2 that is the −y state.

### 4.3 What the hardware does natively

- **RZ(θ)** is virtual. In standard practice a Z rotation is a frame change: all later drive pulses on that qubit have their phase shifted in software. It costs no time, and IBM reports its error as 0.
- **SX and X** are calibrated finite-duration pulses, equal to RX(π/2) and RX(π) up to phase, with the same duration. The docs state that ID is a delay of the same length.
- **Fractional RX(θ)** is one pulse of the same duration as SX and X, with a variable amplitude, so its reported error equals theirs.

### 4.4 Euler decompositions

Any U in SU(2) factors as a Z rotation, a rotation about a transverse axis, and another Z rotation (ZYZ or ZXZ Euler angles). Geometric reason: the first RZ spins the sphere about z, the middle rotation tilts the z-axis to any polar angle, and the last RZ chooses the final azimuth. With only virtual RZ and the fixed SX, the middle rotation must be produced by two SX pulses. We now prove the identity that does this for RX(θ).

```latex
\begin{aligned}
\mathrm{SX}\,R_Z(\phi)\,\mathrm{SX}^\dagger&\simeq R_X(\tfrac\pi2)\,R_Z(\phi)\,R_X(-\tfrac\pi2)=R_Y(-\phi)&&\text{($R_X(\pi/2)$ sends }\hat z\to-\hat y)\\
\mathrm{SX}\,R_Z(\phi)\,\mathrm{SX}&\simeq R_Y(-\phi)\,X&&(\mathrm{SX}^2\simeq X)\\
\phi=\theta+\pi:\quad R_Y(-\theta-\pi)\,X&=R_Y(-\theta)\,(iY)\,X=R_Y(-\theta)\,Z\simeq R_Y(-\theta)\,R_Z(\pi)&&(YX=-iZ)\\
R_Z(\tfrac\pi2)\,R_Y(-\theta)\,R_Z(\pi)\,R_Z(\tfrac\pi2)&\simeq R_Z(\tfrac\pi2)\,R_Y(-\theta)\,R_Z(-\tfrac\pi2)=R_X(\theta)&&(R_Z(\tfrac\pi2)\text{ sends }\hat y\to-\hat x)
\end{aligned}
```

Collecting the steps gives the ZSX form of RX, which is what the transpiler emits on a backend without fractional gates:

```latex
R_X(\theta)\;\simeq\;R_Z(\tfrac\pi2)\;\mathrm{SX}\;R_Z(\theta+\pi)\;\mathrm{SX}\;R_Z(\tfrac\pi2)\qquad(\text{matrix product; rightmost acts first})
```

The same factorization, generalized, is the ZSX Euler form of a general single-qubit gate, U(θ,φ,λ) ≃ RZ(φ+π)·SX·RZ(θ+π)·SX·RZ(λ). I confirmed the RX identity numerically for random θ.

### 4.5 Cost accounting

| Operation | Conventional basis | With fractional RX |
| --- | --- | --- |
| RX(θ), any θ | 2 SX pulses plus 3 virtual RZ | 1 RX pulse |
| X or SX | 1 pulse | 1 pulse |
| RZ(θ) | virtual: 0 time, 0 error | virtual |
| General U(θ,φ,λ) | 2 SX pulses plus 3 virtual RZ | 1 RX plus 2 virtual RZ, using the ZXZ rule of section 9 |

Because the fractional RX lasts as long as one SX, an arbitrary-angle rotation takes half the time and passes through one noisy pulse instead of two, which is the source of the docs' claim of a reduction in duration and error by up to a factor of two for single-qubit rotations.

## 5. Two-qubit geometry: the KAK decomposition and the Weyl chamber

Every two-qubit gate equals two layers of single-qubit gates around one canonical gate exp\[i(c1·XX + c2·YY + c3·ZZ)\], and RZZ(θ) sits on the edge of the resulting "chamber" that runs from the identity to CZ. That edge is exactly the hardware window 0 ≤ θ ≤ π/2.

### 5.1 Local equivalence

Single-qubit gates are cheap (virtual RZ or one pulse), so what matters about a two-qubit gate is its class modulo single-qubit gates. Two gates U and V are locally equivalent if V = (A⊗B)·U·(C⊗D) for single-qubit unitaries A, B, C, D. Two-qubit gates are typically the noisiest operations on a device, which is why counting them is the standard cost measure.

### 5.2 The KAK (Cartan) decomposition

The Lie algebra su(4) has dimension 15. It splits into the local generators (X, Y, Z on either qubit alone: 6 dimensions) and the 9 two-body generators σ\_i⊗σ\_j. The largest set of mutually commuting two-body generators is {XX, YY, ZZ}: for example XX·YY = (XY)⊗(XY) = (iZ)⊗(iZ) = −ZZ and YY·XX = (YX)⊗(YX) = (−iZ)⊗(−iZ) = −ZZ, so they commute. Cartan's theorem then says every gate factors as local gates, a commuting exponential, and local gates:

```latex
U=(A_1\otimes B_1)\;\exp\!\big[i\,(c_1\,XX+c_2\,YY+c_3\,ZZ)\big]\;(A_2\otimes B_2),\qquad A_k,B_k\in SU(2),\qquad 15=6+3+6
```

The count confirms it: six parameters for the left local layer, three canonical coefficients, six for the right local layer. Only (c1, c2, c3) carry nonlocal information.

### 5.3 The Weyl chamber

Local Clifford gates permute the three axes (for example H⊗H swaps ZZ with XX) and flip signs of pairs of coefficients, and a shift of any coefficient by π/2 is itself a local gate, because exp(i(π/2)·ZZ) = i·Z⊗Z. Using these symmetries every gate can be moved into a fundamental region, the Weyl chamber, a tetrahedron:

```latex
\frac{\pi}{4}\;\ge\;c_1\;\ge\;c_2\;\ge\;|c_3|
```

| Gate class | (c1, c2, c3) | Two-qubit gates needed if only CNOT or CZ is native |
| --- | --- | --- |
| Identity, any local gate | (0, 0, 0) | 0 |
| RZZ(θ), 0 < θ < π/2 | (θ/2, 0, 0) | 2 |
| CNOT, CZ (equal to RZZ(π/2) up to local gates) | (π/4, 0, 0) | 1 |
| iSWAP | (π/4, π/4, 0) | 2 |
| SWAP | (π/4, π/4, π/4) | 3 |

The gate counts are standard KAK results (Zhang, Vala, Sastry and Whaley 2003; Vidal and Dawson 2004), cited from memory and not re-fetched here; the rest of the table follows from the algebra on this page.

### 5.4 Where RZZ sits

RZZ(θ) = exp(−i(θ/2)·ZZ) has only one nonzero coefficient. Two symmetries bring any θ into the chamber edge:

```latex
c\to -c:\;\;X_a\,e^{-ic\,ZZ}\,X_a=e^{+ic\,ZZ},\qquad c\to c+\tfrac\pi2:\;\;e^{i\frac\pi2 ZZ}=i\,Z\otimes Z\;\text{(a local gate)}
```

```latex
R_{ZZ}(\theta)=e^{-i\frac\theta2 ZZ}\;\sim\;e^{i\frac\theta2 XX}\;\Rightarrow\;(c_1,c_2,c_3)=(\tfrac\theta2,0,0),\qquad 0\le\theta\le\tfrac\pi2\;\Longleftrightarrow\;0\le c_1\le\tfrac\pi4
```

So the whole family RZZ(θ), θ in \[0, π/2\], is the edge of the Weyl chamber from the identity (θ = 0) to the CZ class (θ = π/2), and every other θ is equivalent to a point on this edge. Reducing c = θ/2 modulo π/2 into (−π/4, π/4\] and taking the absolute value is the same as reducing θ modulo π into (−π/2, π/2\] and taking the absolute value. That is exactly the folding map of section 6.

### 5.5 CZ is the end of the edge

For bits a, b let z = 1 − 2·bit. The CZ phase (−1)^(ab) equals exp\[iπ(1 − z\_a)(1 − z\_b)/4\], so expanding the product gives CZ as an exponential of Z-type terms:

```latex
\mathrm{CZ}=e^{\,i\frac\pi4\,(I-Z_a-Z_b+Z_aZ_b)}\;\simeq\;R_{ZZ}\!\left(-\tfrac\pi2\right)\big(R_Z(\tfrac\pi2)\otimes R_Z(\tfrac\pi2)\big)
```

So CZ is RZZ at θ = −π/2 (equivalently +π/2 up to X conjugation) times virtual single-qubit Z rotations. In the conventional basis, the only entangling angle the hardware offers is this one point; every other RZZ angle has to be assembled from two of them (section 6). With the fractional gate the hardware offers the entire edge. I confirmed the CZ identity numerically.

## 6. RZZ from CZ, and the angle-folding identities

With only CZ native, RZZ(θ) costs two CZ gates plus single-qubit gates; with the fractional gate it costs one, but only for θ in \[0, π/2\], and three exact identities map every other angle into that window. This section proves both statements.

### 6.1 RZZ built from two CZ gates

Conjugating Z on the target by a CNOT turns it into a parity operator: CX·(I⊗Z)·CX = Z⊗Z. By the conjugation rule of section 3.5 this carries over to the rotation. Writing CX = H\_b·CZ·H\_b and using H·RZ·H = RX gives the standard two-CZ form:

```latex
R_{ZZ}(\theta)=\mathrm{CX}\,\big(I\otimes R_Z(\theta)\big)\,\mathrm{CX}=H_b\;\mathrm{CZ}\;R_{X,b}(\theta)\;\mathrm{CZ}\;H_b
```

Count the cost on a backend without fractional gates: two CZ gates, plus an RX(θ) in the middle (two SX pulses by section 4.4), plus the two Hadamards. The docs describe this as two CZ gates and several single-qubit gates. With the fractional gate the whole expression is one RZZ(θ). In the tutorial's four-qubit kernel the transpiler output shows exactly this ratio: 18 RZZ gates on the fractional backend against 36 CZ gates on the conventional one.

I confirmed the identity numerically for random θ. Section 5.3 explains why two CZ is the minimum: RZZ(θ) lies strictly inside the chamber edge, and only the end points (identity and the CZ class) are cheaper.

### 6.2 Three identities

Write ZZ for Z⊗Z. All three follow from the facts of section 3.

```latex
\text{(i)}\;\;R_{ZZ}(\theta+2\pi)=-R_{ZZ}(\theta)\qquad\text{(ii)}\;\;R_{ZZ}(\theta+\pi)=R_{ZZ}(\theta)\,(-i\,ZZ)\qquad\text{(iii)}\;\;X_a\,R_{ZZ}(\theta)\,X_a=R_{ZZ}(-\theta)
```

**Proof of (ii).** cos((θ+π)/2) = −sin(θ/2) and sin((θ+π)/2) = cos(θ/2), so the left side is −sin(θ/2)·I − i·cos(θ/2)·ZZ. The right side is (cos(θ/2)·I − i·sin(θ/2)·ZZ)(−i·ZZ) = −i·cos(θ/2)·ZZ − sin(θ/2)·I, because ZZ·ZZ = I. They agree.

**Proof of (iii).** X\_a anticommutes with ZZ (it flips the sign of Z\_a and leaves Z\_b alone), so X\_a·ZZ·X\_a = −ZZ and the exponent changes sign.

### 6.3 The folding algorithm

Given any real θ, let n be the nearest integer to θ/π and φ = θ − nπ, so φ lies in \[−π/2, π/2\]. Identity (ii) applied n times gives RZZ(θ) = RZZ(φ)·(−i·ZZ)^n, and identity (iii) removes a negative sign of φ:

```latex
R_{ZZ}(\theta)=\big[X_a\big]^{s}\,R_{ZZ}(|\varphi|)\,\big[X_a\big]^{s}\,(-i\,ZZ)^{n},\qquad s=\begin{cases}1&\varphi<0\\0&\varphi\ge0\end{cases},\qquad \theta'=|\varphi|=\Big|\big((\theta+\tfrac\pi2)\bmod\pi\big)-\tfrac\pi2\Big|\in[0,\tfrac\pi2]
```

The correction factors are all local: (−i·ZZ)^n is Z⊗Z for odd n (two virtual RZ(π) rotations, free) and a global phase for even n; the X\_a pair appears only when φ is negative and costs two X pulses. The folded angle θ′ is a triangle wave in θ with period π and amplitude π/2. The formula is the same as the chamber argument of section 5.4, and I confirmed the full decomposition numerically for random θ (it holds exactly, not only up to phase).

### 6.4 Worked examples from the kernel circuit

The tutorial's feature map produces angles between about 4.59 and 9.87 rad (section 8.3). Three samples:

| θ requested (rad) | n | φ = θ − nπ | θ′ executed | Local corrections |
| --- | --- | --- | --- | --- |
| 4.58 | 1 | +1.438 | 1.438 | Z⊗Z (virtual) |
| 6.00 | 2 | −0.283 | 0.283 | X on qubit a before and after (2 X pulses) |
| 9.87 | 3 | +0.445 | 0.445 | Z⊗Z (virtual) |

This explains the extra X gates that the tutorial reports after transpiling with fractional gates, and why its Rzz angles that were negative before transpilation become positive afterwards.

### 6.5 Consequence for workflows

The transpiler pass FoldRzzAngle performs this reduction automatically when the backend is passed to generate\_preset\_pass\_manager or transpile, but it needs numbers. For a parameterized RZZ it can only assume that every parameter value will land in \[0, π/2\]; if a value falls outside, the job fails with an error that quotes the allowed range \[0, pi/2\]. Section 8.5 covers the two workflows that follow from this.

## 7. Fractional gates on hardware: use, error model, pros and cons

Fractional gates shorten rotation-heavy circuits and, to first order, cut the error in proportion, but they are experimental, constrain the RZZ angle to \[0, π/2\], and disable several error-mitigation tools. Source: [Fractional gates](https://quantum.cloud.ibm.com/docs/en/guides/fractional-gates), with the kernel tutorial in section 8.

### 7.1 What is supported

- RX(θ) for any real θ (single-qubit) and RZZ(θ) for angles in the window 0 to π/2 (two-qubit), on Heron QPUs, only when the backend is loaded with use\_fractional\_gates=True.
- The guide writes the RZZ window as 0 < θ ≤ π/2; the runtime error message quotes \[0, pi/2\]; the tutorial's limitations list writes 0 < θ < π/2 in one place. At θ = 0 the gate is the identity, so the end-point question has no practical effect.
- The guide's demonstration is a five-qubit Ising-chain circuit with three time steps and RX and RZZ angles of 0.1 rad, transpiled once with and once without the flag at optimization level 3 with ALAP scheduling, then compared as timelines.

Why Ising dynamics fits the window: a first-order Trotter step of H = J·Σ Z\_iZ\_(i+1) + h·Σ X\_i with step size δ is a product of exp(−iJδ·ZZ) and exp(−ihδ·X) factors. Matching exp(−iθZZ/2) gives RZZ(2Jδ) and RX(2hδ), and small Trotter steps keep θ well inside \[0, π/2\].

### 7.2 Error model: what the docs say and what first-order theory adds

The docs state that error rates for fractional gates are not yet reported separately: the error value in the Target of a fractional-gate backend is a copy of the non-fractional counterpart. Their argument is that fractional and conventional gates take the same time, so comparable errors are reasonable, especially when relaxation dominates. (The Platform page separately lists an RZZ error averaged over angles, so treat the numbers as indicative, not final.)

Two first-order facts explain why fewer gates help. If errors are small and independent, circuit fidelity is the product of per-operation fidelities, F ≈ Π(1 − ε\_i) ≈ exp(−Σ ε\_i), so removing half the two-qubit gates halves the two-qubit term in the exponent. And idle relaxation grows with duration. The standard average-fidelity result for a qubit idling for time t (amplitude damping plus dephasing; cited from memory) is:

```latex
\bar F_{\text{idle}}(t)=\tfrac12+\tfrac16\,e^{-t/T_1}+\tfrac13\,e^{-t/T_2}\;\approx\;1-t\Big(\frac{1}{6T_1}+\frac{1}{3T_2}\Big)\quad(t\ll T_{1,2})
```

The error is linear in t, so halving the duration halves the relaxation error. Illustrative numbers with T1 = T2 = 200 µs: t = 10 µs gives error 0.0244, t = 5 µs gives 0.0123.

### 7.3 Pros

| Benefit | Mechanism | Evidence in the sources |
| --- | --- | --- |
| Shorter circuits | RX(θ) is 1 pulse instead of 2 SX; RZZ(θ) is 1 gate instead of 2 CZ plus single-qubit gates | Guide: single-qubit rotations up to 2× shorter and less erroneous; tutorial: 36 CZ become 18 RZZ |
| Less accumulated error | Fewer noisy operations and a shorter schedule | Tutorial simulation: K(x,x) 0.929 against 0.8625 at 4 qubits (section 8.6) |
| Fits rotation-heavy workloads | Many arbitrary-angle RX and RZZ gates | Hamiltonian simulation, QAOA, variational ansätze, quantum kernels |
| Same workflow | One backend flag; RX-only mode keeps the standard patterns workflow | Tutorial section on fractional RX only |
| Neutral on QPU time | Usage time did not change | Tutorial hardware run: 8 s with and 8 s without |

### 7.4 Cons and limits

| Limit | Detail | Mitigation |
| --- | --- | --- |
| Experimental | The use\_fractional\_gates flag may change | Read the qiskit-ibm-runtime release notes before upgrading |
| Error figures are copies | Target errors equal the non-fractional counterpart; the Platform shows an angle-averaged RZZ error | Treat noise models built from them as approximate |
| RZZ angle window | Out-of-range parameter values fail the job; folding needs numbers, so parameters must be bound before transpilation | Bind first (section 8.5), or keep angles inside \[0, π/2\] |
| Weak transpiler support | The transpiler has limited ability to optimize with RZZ | Hand-craft and inspect circuits |
| Unsupported techniques | Pauli twirling, probabilistic error cancellation (PEC), zero-noise extrapolation via probabilistic error amplification (PEA), dynamic circuits | Measurement twirling with TREX is supported |
| Known issue | In qiskit-ibm-runtime 0.47.0, invalid RZZ angles can survive transpilation (issue 2441) | Add FoldRzzAngle, Optimize1qGatesDecomposition and RemoveIdentityEquivalent as post-optimization passes, as in the tutorial |

### 7.5 Decision rules

| Situation | Recommendation |
| --- | --- |
| Many arbitrary-angle RX or RZZ gates (Trotter dynamics, QAOA, variational ansatz, kernel) | Use fractional gates; bind parameters first if RZZ angles can leave \[0, π/2\] |
| Circuit is mostly Clifford or CZ-only | No gain; leave the flag off |
| You need Pauli twirling, PEC, ZNE with PEA, or dynamic circuits | Do not use fractional gates |
| You want a low-risk first step | RX-only mode: remove rzz from the Target and keep the standard workflow (section 9) |

## 8. Case study: fractional gates in a quantum-kernel circuit

The tutorial's kernel circuit is a block of RZ and RZZ rotations with all angles known at run time, so it is the ideal stress test: every RZZ angle lies outside the hardware window, folding is mandatory, and halving the two-qubit count (36 CZ to 18 RZZ at four qubits) raises the measured fidelity. Source: [Quantum kernels with fractional gates](https://quantum.cloud.ibm.com/docs/en/tutorials/fractional-gates). Only the circuit structure matters here; no kernel-method background is needed.

### 8.1 What the circuit measures

The feature map U(x) turns a data vector x into the state |Φ(x)⟩ = U(x)|0…0⟩. The kernel entry is the squared overlap of two such states, and it is estimated by running the mirrored circuit U(y)†·U(x) and counting how often the all-zero string comes out.

```latex
K(\mathbf x,\mathbf y)=\big|\langle\Phi(\mathbf y)|\Phi(\mathbf x)\rangle\big|^2=\big|\langle 0^n|U(\mathbf y)^\dagger U(\mathbf x)|0^n\rangle\big|^2=P(0^n),\qquad K=\cos^2 d_{FS},\quad d_{FS}=\arccos\big|\langle\Phi(\mathbf x)|\Phi(\mathbf y)\rangle\big|
```

Geometrically, |Φ(x)⟩ is a point (a ray) in a (2^n)-dimensional complex projective space, d\_FS is the Fubini–Study angle between two such points, and K is cos² of that angle: 1 for identical states, 0 for orthogonal ones. The tutorial writes the kernel as the inner product ⟨Φ(x)|Φ(y)⟩; the all-zero probability that the hardware reports is its squared magnitude.

For identical inputs the mirrored circuit is the identity, so K(x,x) = 1 ideally. On a real QPU it is below 1, and the tutorial uses this value, called fidelity, as a noise gauge: closer to 1 means a cleaner execution. The mirror doubles the circuit, which makes the circuit a demanding benchmark for the gate set.

### 8.2 The ZZ feature map: gate structure and a closed form

The tutorial builds its own map because Qiskit's built-in zz\_feature\_map did not support RZZ gates at Qiskit 2.4.1 (issue 14469). The structure, for n qubits and r repetitions, is: a Hadamard on every qubit once, then r layers, each an RZ(x\_i) on every qubit followed by RZZ gates on nearest-neighbour pairs (even pairs first, then odd pairs). The parameter vector x has n·r entries and each layer uses its own block of n.

```latex
U(\mathbf x)=L_r\cdots L_1\,H^{\otimes n},\qquad L_k=\Big[\prod_{(i,i+1)}R_{ZZ}\big(\theta^{(k)}_{i,i+1}\big)\Big]\Big[\bigotimes_i R_Z\big(x^{(k)}_i\big)\Big],\qquad \theta^{(k)}_{ij}=\big(\pi-x^{(k)}_i\big)\big(\pi-x^{(k)}_j\big)
```

**Geometry.** After the Hadamards every qubit's Bloch vector sits on the equator, at azimuth 0. RZ(x\_i) rotates it along the equator by azimuth x\_i. RZZ(θ) then adds the relative phase θ between even-parity and odd-parity components (section 3.4), which entangles neighbours by an amount set by the product of two features.

**A closed form.** After the single layer of Hadamards, every later gate (RZ and RZZ) is diagonal in the computational basis, so all of them commute and only add phases. With z\_i = (−1)^(s\_i):

```latex
U(\mathbf x)|0^n\rangle=2^{-n/2}\sum_{s\in\{0,1\}^n}e^{-\frac i2\Phi(s;\mathbf x)}\,|s\rangle,\qquad \Phi(s;\mathbf x)=\sum_{k=1}^{r}\Big[\sum_i x^{(k)}_i z_i+\sum_{(i,j)}\theta^{(k)}_{ij}\,z_iz_j\Big]
```

```latex
K(\mathbf x,\mathbf y)=\Big|\,2^{-n}\sum_{s}e^{-\frac i2\left[\Phi(s;\mathbf x)-\Phi(s;\mathbf y)\right]}\Big|^2
```

I checked both formulas numerically for n = 4, r = 3 (state and kernel agree to machine precision, and K(x,x) = 1). Two consequences. First, the circuit is a Hadamard, one diagonal layer, then a Hadamard, so the repetitions simply accumulate phase. Second, for y = x the diagonal layers cancel exactly, so the mirror would collapse to the identity if the optimizer could merge them; the barrier that the tutorial inserts between U(x) and U(y)† (insert\_barrier=True) stops that and keeps the benchmark honest. (The barrier explanation is my inference; the tutorial's 36 remaining CZ gates show that nothing cancelled.) This closed form applies to the tutorial's hand-built map, not to Qiskit's built-in one.

### 8.3 The angle landscape and why folding is mandatory

The tutorial draws each feature from rng.random, so x lies in \[0, 1). Then π − x lies in (π − 1, π\], and the product of two such factors lies in an interval entirely above the hardware window:

```latex
\theta_{ij}=(\pi-x_i)(\pi-x_j)\in\big((\pi-1)^2,\;\pi^2\big)\approx(4.586,\;9.870),\qquad \tfrac\pi2\approx1.571
```

No RZZ angle in this circuit is inside \[0, π/2\]. Applying section 6.3 over the whole range gives three regions:

| θ range (rad) | n = round(θ/π) | φ = θ − nπ | Local corrections |
| --- | --- | --- | --- |
| 4.586 to 4.712 | 1 | 1.444 to 1.571 | Z⊗Z (virtual) |
| 4.712 to 7.854 | 2 | −1.571 to +1.571 | X pair on qubit a when θ < 2π ≈ 6.283 |
| 7.854 to 9.870 | 3 | −1.571 to +0.445 | Z⊗Z (virtual), plus X pair when θ < 3π ≈ 9.425 |

About half of the middle region and most of the last region need the X pair. That is why the transpiled four-qubit circuit still contains 13 X gates, and why a folded RZZ is cheaper than two CZ gates but not always exactly one gate. The tutorial's post-optimization step runs Optimize1qGatesDecomposition after folding, which can merge such X gates into neighbouring single-qubit gates.

### 8.4 Gate counts

Per mirrored circuit, the RZZ count is 2·r·(n−1): each of the r layers has n−1 neighbour pairs, and the mirror doubles it. On a conventional backend every RZZ becomes two CZ.

| Qubits n | RZZ gates (r = 3) | CZ gates on a conventional backend | Note |
| --- | --- | --- | --- |
| 4 | 18 | 36 | Matches the tutorial's transpiled output |
| 10 | 54 | 108 | Formula |
| 20 | 114 | 228 | Formula |
| 40 | 234 | 468 | Formula; before any routing, assuming the chain maps onto the coupling map |

The tutorial's four-qubit transpiled operation counts (optimization level 2, r = 3):

| Operation | Without fractional gates | With fractional gates |
| --- | --- | --- |
| cz | 36 | 0 |
| rzz | 0 | 18 |
| sx | 80 | 0 |
| x | 0 | 13 |
| rx | 0 | 9 |
| rz (virtual) | 130 | 35 |
| Two-qubit gates | 36 | 18 |
| Physical single-qubit pulses (sx + x + rx) | 80 | 22 |

The last two rows are my sums from the tutorial output. RZ is a frame change and costs no time, so the physical single-qubit pulse count drops from 80 to 22.

### 8.5 Two workflows for the RZZ angle constraint

Both follow the Qiskit patterns workflow. The difference is whether the RZZ angles are known to the transpiler.

```python
# Approach 1: parameterized PUB. Valid only if every RZZ angle stays in [0, pi/2].
pm = generate_preset_pass_manager(backend=backend, optimization_level=2)
t_circuit = pm.run(circuit)
sampler.run([(t_circuit, parameter_values)])

# Approach 2 (recommended in the tutorial): bind first, so FoldRzzAngle sees numbers.
b_circuit = circuit.assign_parameters(parameter_values)
t_circuit = pm.run(b_circuit)
sampler.run([(t_circuit,)])
```

Approach 2 costs more compute and requires storing the bound circuits locally. If you submit approach 1 with an out-of-range value, the job fails with a message of the form "rzz is supported only for angles in the range \[0, pi/2\]". Because of the qiskit-ibm-runtime 0.47.0 issue, the tutorial also sets a post-optimization pass list (section 7.4 and the cell below).

### 8.6 Results

Simulator, small scale: Aer sampler built from the backend (its noise model), 2000 shots, r = 3, optimization level 2, even qubit counts 4 to 10. The value shown is K(x,x).

| Qubits | Without fractional gates | With fractional gates | Gain |
| --- | --- | --- | --- |
| 4 | 0.8625 | 0.929 | +0.0665 |
| 6 | 0.7605 | 0.882 | +0.1215 |
| 8 | 0.702 | 0.8645 | +0.1625 |
| 10 | 0.671 | 0.817 | +0.1460 |

Because the backend Target copies each fractional gate's error from its counterpart (section 7.2), this simulated gain reflects fewer gates and a shorter schedule, not separately calibrated fractional-gate errors.

Hardware, large scale: qubit counts 4 to 40 in steps of 4, Sampler with dynamical decoupling (XY4) enabled, one job with and one without fractional gates. The tutorial plots depth, estimated duration, non-local gate count and fidelity against qubit count; its text gives no numbers for those curves, so none are quoted here. It does report QPU usage: 8 s for both jobs.

### 8.7 Four ways to use fractional gates in the kernel

| Variant | Gates | Workflow | Valid when | Notes |
| --- | --- | --- | --- | --- |
| A. Full, parameterized PUB | RX and RZZ | Transpile once with symbols, send values in the PUB | Every RZZ angle is in \[0, π/2\] | Cheapest in compute |
| B. Full, bind first | RX and RZZ | assign\_parameters, then transpile with FoldRzzAngle | Any angles | The tutorial's recommended route and the one used for the kernel |
| C. RX only | RX | Remove rzz from the Target; standard patterns workflow | Any angles | No angle constraint |
| D. RX only with ZXZ rule | RX | Add a custom equivalence for U (section 9) | Circuits rich in U gates | Under discussion in Qiskit issue 13455 |

### 8.8 Runnable cell: compare gate counts with and without the flag

This cell follows the tutorial's logic with my own feature-map code. I have not executed it here because it needs IBM Quantum credentials and a Heron backend; run it in your environment.

```python
import numpy as np
from qiskit import QuantumCircuit, generate_preset_pass_manager
from qiskit.circuit import ParameterVector
from qiskit.circuit.library import unitary_overlap
from qiskit.transpiler import PassManager
from qiskit.transpiler.passes import Optimize1qGatesDecomposition, RemoveIdentityEquivalent
from qiskit_ibm_runtime import QiskitRuntimeService
from qiskit_ibm_runtime.transpiler.passes import FoldRzzAngle

service = QiskitRuntimeService()
name = service.least_busy(operational=True, simulator=False, min_num_qubits=133).name
backend_c = service.backend(name)                              # no fractional gates
backend_f = service.backend(name, use_fractional_gates=True)   # RX and RZZ added
assert "rzz" in backend_f.target.operation_names, f"{name} has no fractional gates"

def zz_feature_map(n, reps):
    x = ParameterVector("x", n * reps)
    qc = QuantumCircuit(n)
    qc.h(range(n))
    for k in range(reps):
        xs = x[k * n:(k + 1) * n]
        for i in range(n):
            qc.rz(xs[i], i)
        pairs = [(i, i + 1) for i in range(n - 1)]
        for i, j in pairs[0::2] + pairs[1::2]:
            qc.rzz((np.pi - xs[i]) * (np.pi - xs[j]), i, j)
    return qc

def kernel_circuit(n, reps=3):
    fm = zz_feature_map(n, reps)
    qc = unitary_overlap(fm, fm, "x", "y", insert_barrier=True)
    qc.measure_all()
    return qc

def transpile_for(backend, circuit, level=2):
    pm = generate_preset_pass_manager(optimization_level=level, backend=backend)
    if "rzz" in backend.target.operation_names:   # workaround for qiskit-ibm-runtime 0.47.0
        pm.post_optimization = PassManager([
            FoldRzzAngle(),
            Optimize1qGatesDecomposition(target=backend.target),
            RemoveIdentityEquivalent(target=backend.target),
        ])
    return pm.run(circuit)

rng = np.random.default_rng(123)
print(f"{'n':>3} {'backend':>10} {'2Q gates':>9} {'depth':>6} {'duration_us':>12}")
for n in range(4, 44, 4):
    qc = kernel_circuit(n)
    bound = qc.assign_parameters(np.tile(rng.random(qc.num_parameters // 2), 2))  # bind BEFORE transpiling
    for label, be in (("standard", backend_c), ("fractional", backend_f)):
        t = transpile_for(be, bound)
        dur = t.estimate_duration(be.target, unit="u")
        print(f"{n:>3} {label:>10} {t.num_nonlocal_gates():>9} {t.depth():>6} {dur:>12.2f}")
```

## 9. RX-only mode and the ZXZ decomposition of U

Removing rzz from the Target keeps the standard Qiskit patterns workflow, and a ZXZ rule for the U gate lets one RX replace two SX pulses; in the tutorial's example circuit this takes the SX count from 16 to 0 and the RX count to 9. Source: the advanced section of the [kernel tutorial](https://quantum.cloud.ibm.com/docs/en/tutorials/fractional-gates).

### 9.1 Why RX-only is simpler

The workflow changes in section 8.5 exist only because of the RZZ angle window. RX accepts any real angle, so if the Target offers RX but not RZZ, there is nothing to fold: parameterized circuits go through the unchanged workflow, and circuits with many RX or U gates still shrink.

### 9.2 The math: U as ZXZ

Qiskit's U(θ, φ, λ) equals RZ(φ)·RY(θ)·RZ(λ) up to phase. Conjugating RX by a quarter turn about z sends the x-axis to the y-axis (section 4.4), so RY is an RX in disguise, and the Z rotations merge:

```latex
R_Y(\theta)=R_Z(\tfrac\pi2)\,R_X(\theta)\,R_Z(-\tfrac\pi2)\;\Rightarrow\;U(\theta,\varphi,\lambda)\simeq R_Z\!\big(\varphi+\tfrac\pi2\big)\;R_X(\theta)\;R_Z\!\big(\lambda-\tfrac\pi2\big)
```

In circuit order that is rz(λ − π/2), then rx(θ), then rz(φ + π/2), exactly the rule the tutorial registers. Compare the ZSX form of section 4.4, U ≃ RZ(φ+π)·SX·RZ(θ+π)·SX·RZ(λ): two physical SX pulses become one RX pulse. I confirmed the ZXZ identity numerically for random angles.

### 9.3 Runnable cell

Adapted from the tutorial; not executed here because it needs IBM Quantum credentials and the qiskit-basis-constructor package (`pip install qiskit_basis_constructor`).

```python
import numpy as np
from qiskit import generate_preset_pass_manager
from qiskit.circuit import ParameterVector, QuantumCircuit
from qiskit.circuit.library import UGate, n_local
from qiskit.transpiler import Target
from qiskit_basis_constructor import DEFAULT_EQUIVALENCE_LIBRARY
from qiskit_ibm_runtime import QiskitRuntimeService

service = QiskitRuntimeService()
name = service.least_busy(operational=True, simulator=False, min_num_qubits=133).name
backend_c = service.backend(name)
backend_f = service.backend(name, use_fractional_gates=True)

def remove_instruction_from_target(target: Target, gate_name: str) -> Target:
    new = Target(description=target.description, num_qubits=target.num_qubits, dt=target.dt,
                 granularity=target.granularity, min_length=target.min_length,
                 pulse_alignment=target.pulse_alignment, acquire_alignment=target.acquire_alignment,
                 qubit_properties=target.qubit_properties,
                 concurrent_measurements=target.concurrent_measurements)
    for op_name, qarg_map in target.items():
        if op_name == gate_name:
            continue
        if qarg_map == {None: None}:
            qarg_map = None
        new.add_instruction(target.operation_from_name(op_name), qarg_map, name=op_name)
    return new

target_rx_only = remove_instruction_from_target(backend_f.target, "rzz")   # keep rx, drop rzz

qc = n_local(3, "u", "cz", "linear", reps=1)
qc.rzz(1.1, 0, 1)

# 1. conventional basis
print("standard      :", dict(generate_preset_pass_manager(optimization_level=2, backend=backend_c).run(qc).count_ops()))
# 2. fractional RX only
print("RX only       :", dict(generate_preset_pass_manager(optimization_level=2, target=target_rx_only).run(qc).count_ops()))

# 3. RX only plus a ZXZ rule for U (feature under discussion: Qiskit issue 13455)
x = ParameterVector("x", 3)
zxz = QuantumCircuit(1)
zxz.rz(x[2] - np.pi / 2, 0)
zxz.rx(x[0], 0)
zxz.rz(x[1] + np.pi / 2, 0)
DEFAULT_EQUIVALENCE_LIBRARY.add_equivalence(UGate(x[0], x[1], x[2]), zxz)
pm = generate_preset_pass_manager(optimization_level=2, target=target_rx_only,
                                  translation_method="constructor-beta")
print("RX only + ZXZ :", dict(pm.run(qc).count_ops()))
```

### 9.4 Tutorial result

Circuit: a three-qubit n\_local block with U rotations and linear CZ entanglers, plus one RZZ(1.1). Reported operation counts:

| Variant | rz (virtual) | sx | rx | cz | Physical single-qubit pulses |
| --- | --- | --- | --- | --- | --- |
| Conventional basis | 23 | 16 | 0 | 4 | 16 |
| Fractional RX only | 22 | 14 | 1 | 4 | 15 |
| Fractional RX only plus ZXZ rule | 16 | 0 | 9 | 4 | 9 |

The last column is my sum of sx and rx. The ZXZ rule is a custom rule that the tutorial flags as under discussion upstream, so check the release notes before relying on it.

## 10. Worked exercises with answers

Eight short exercises cover each block of the notes; the answers are hand-computed and the arithmetic is shown.

1. **Physical T2.** A qubit reports T1 = 100 µs and T2 = 250 µs. Is that consistent?
   - Answer: No. Section 2.4 shows T2 ≤ 2·T1 = 200 µs, so 250 µs is outside the bound, which points to a stale or mismeasured value.
2. **Readout error.** P(0|1) = 0.03 and P(1|0) = 0.01. What is readout\_error?
   - Answer: the mean, (0.03 + 0.01)/2 = 0.02.
3. **EPLG.** Layer fidelity of a 100-qubit chain is LF = 0.6. Find EPLG\_100.
   - Answer: 0.6^(1/99) = exp(−0.51083/99) = 0.99485, so EPLG = 0.8 × (1 − 0.99485) ≈ 0.0041.
4. **Exponential identity.** Show RZZ(π) = −i·Z⊗Z and explain why it is a local gate.
   - Answer: put θ = π in cos(θ/2)·I − i·sin(θ/2)·ZZ to get cos(π/2) = 0 and sin(π/2) = 1, so RZZ(π) = −i·Z⊗Z. This is a tensor product of single-qubit Paulis, so it is local (c = π/2 in section 5.3). Folding agrees: n = 1, φ = 0, θ′ = 0.
5. **Fold three angles.** Find n, φ, θ′ and the local corrections for θ = 7.0, 5.0 and −0.4.
   - Answer: θ = 7.0: n = 2, φ = 0.717, θ′ = 0.717, no X pair, no Z⊗Z. θ = 5.0: n = 2, φ = −1.283, θ′ = 1.283, X pair on qubit a. θ = −0.4: n = 0, φ = −0.4, θ′ = 0.4, X pair on qubit a.
6. **Gate count.** For the tutorial's kernel with n = 12 qubits and r = 3, how many RZZ gates and, on a conventional backend, how many CZ gates?
   - Answer: RZZ = 2·r·(n−1) = 2·3·11 = 66; CZ = 2 × 66 = 132 (before routing).
7. **Idle error.** With T1 = T2 = 150 µs, a circuit's duration falls from 12 µs to 6 µs. Compare the idle error of one qubit using the formula of section 7.2.
   - Answer: with T1 = T2 = T the formula reduces to error = ½·(1 − e^(−t/T)). At 12 µs: ½(1 − 0.92312) = 0.0384. At 6 µs: ½(1 − 0.96079) = 0.0196. Halving the duration nearly halves the error.
8. **ZXZ.** Using RZ(π/2)·RX(θ)·RZ(−π/2) = RY(θ), derive U(θ, φ, λ) as one RX between two RZ gates.
   - Answer: U ≃ RZ(φ)·RY(θ)·RZ(λ) = RZ(φ)·RZ(π/2)·RX(θ)·RZ(−π/2)·RZ(λ) = RZ(φ + π/2)·RX(θ)·RZ(λ − π/2), using additivity of same-axis rotations.

## 11. References

Pages opened and read on 2026-10-30:

- [View backend details](https://quantum.cloud.ibm.com/docs/en/guides/qpu-information): backend listing, versioning, supported instructions, qubit and instruction properties, EPLG, RZZ error.
- [Fractional gates](https://quantum.cloud.ibm.com/docs/en/guides/fractional-gates): RZZ and RX support, angle constraints, when not to use, backend discovery.
- [Quantum kernels with fractional gates](https://quantum.cloud.ibm.com/docs/en/tutorials/fractional-gates): kernel circuit, workflows, op counts, fidelities, RX-only mode.

Not re-fetched (cited from memory, verify before quoting): the KAK gate-count results (Zhang, Vala, Sastry and Whaley, 2003; Vidal and Dawson, 2004), the idle-fidelity formula in section 7.2, and the average-fidelity factor d/(d+1) used for EPLG. My own derivations (sections 3 to 6, 8.2, 8.3 and 9.2) were verified numerically with numpy on 2026-10-30; the two runnable cells were not executed because they need IBM Quantum credentials.
