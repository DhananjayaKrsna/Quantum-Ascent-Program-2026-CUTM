# IBM Quantum Hardware and Fractional Gates: a beginner-friendly guide

Last updated: 2026-10-30

This repository explains two things about IBM's Heron quantum computers:

1. **How to look at a real quantum computer's specifications** (how long its qubits last, how often its gates make mistakes, which gates it can run).
2. **What "fractional gates" are**, why they can make circuits shorter and more accurate, and when they are the wrong choice.

You do not need to know quantum computing already. Every new word is explained the first time it appears, and there is a glossary at the bottom. If you can read a little Python and remember high-school trigonometry, you can follow everything here.

---

## Contents

1. [What you will learn](#what-you-will-learn)
2. [What is in this repository](#what-is-in-this-repository)
3. [The 10-minute version](#the-10-minute-version)
4. [Part 1: Reading a quantum computer's specification sheet](#part-1-reading-a-quantum-computers-specification-sheet)
5. [Part 2: Fractional gates in plain language](#part-2-fractional-gates-in-plain-language)
6. [Part 3: The mathematics, step by step](#part-3-the-mathematics-step-by-step)
7. [Part 4: Pros and cons](#part-4-pros-and-cons)
8. [Part 5: A worked example (quantum kernels)](#part-5-a-worked-example-quantum-kernels)
9. [Try it yourself](#try-it-yourself)
10. [Suggested reading order](#suggested-reading-order)
11. [Honest limitations of this material](#honest-limitations-of-this-material)
12. [Glossary](#glossary)
13. [Sources](#sources)

---

## What you will learn

- How to list IBM quantum computers and read their key numbers (qubit count, T1, T2, readout error, gate error, gate duration).
- How to ask IBM only for the machines that support fractional gates (`use_fractional_gates=True`).
- What the two fractional gates, `RX(θ)` and `RZZ(θ)`, actually do, with the matrices written out.
- Why a fractional `RZZ` replaces two `CZ` gates, and why a fractional `RX` replaces two `SX` gates.
- The "angle folding" trick that lets any angle be used even though the hardware only accepts angles between 0 and π/2 for `RZZ`.
- The trade-offs: what you gain, what you lose, and a simple rule for deciding.
- A real example from IBM's tutorial, followed from the geometry and maths side.

## What is in this repository

| Item | What it is | Who it is for |
| --- | --- | --- |
| `README.md` (this file) | Plain-language overview with the key equations | Everyone, start here |
| Lecture notes | Long, detailed notes with full derivations, worked exercises and runnable code | Anyone who wants the proofs |
| Slide deck (28 slides, about 45 minutes) | Point-wise presentation with speaker notes | Presenters and students |

The file names in your copy of the repository may differ. The lecture notes and slides are the long-form versions of what is summarised here.

## The 10-minute version

- A **qubit** is the quantum version of a bit. A **gate** is an operation on one or two qubits. A **circuit** is a list of gates.
- Real hardware makes small mistakes on every gate. **Two-qubit gates make the most mistakes**, so fewer of them means a better answer.
- On IBM Heron machines the usual two-qubit gate is `CZ`. Many algorithms need a gate called `RZZ(θ)` that can have any angle θ. With only `CZ` available, **one `RZZ(θ)` costs two `CZ` gates** plus extra single-qubit gates.
- **Fractional gates** let the hardware run `RZZ(θ)` directly as one gate (for θ between 0 and π/2), and `RX(θ)` directly for any θ.
- In the IBM tutorial's four-qubit example, switching them on cut the two-qubit gate count from **36 to 18**, and the physical single-qubit pulses from **80 to 22**.
- The price: they only exist on Heron machines, some error-reduction tools do not work with them, and the circuit must have its numbers filled in before it is compiled.

---

## Part 1: Reading a quantum computer's specification sheet

Think of a quantum computer like a car. Before you drive it, you check the engine size, fuel economy and reliability. IBM publishes the equivalent numbers for each of its machines, and you can read them with a few lines of Python.

### The key terms

| Term | Plain meaning | Good direction |
| --- | --- | --- |
| **Qubit count** | How many qubits the machine has (for example, 156 on `ibm_fez`) | More is not always better; quality matters too |
| **Processor type** | The chip family, such as Heron r2 | Newer revisions are generally better |
| **T1** | How long a qubit stays "excited" before it relaxes to its resting state. Measured in microseconds (µs) | Higher |
| **T2** | How long a qubit keeps the delicate "phase" information of a superposition | Higher |
| **Readout error** | How often measuring a qubit gives the wrong answer | Lower |
| **Gate error** | How often a gate does the wrong thing | Lower |
| **Gate duration** | How long a gate takes, in nanoseconds (ns) | Shorter |
| **Coupling map** | Which pairs of qubits are physically connected and can run a two-qubit gate | Needed for compiling |
| **Basis gates** | The small set of gates the hardware can actually run. Everything else is built out of these | Fractional gates add `rx` and `rzz` |
| **EPLG** | "Error per layered gate": a single figure for how good a whole chain of two-qubit gates is | Lower |

### One rule you can check by eye: T2 can never exceed 2 × T1

A qubit that is losing energy (T1) is also losing phase information, at half that rate at least. Extra noise can only shorten T2. So:

```math
T_2 \le 2\,T_1
```

For example, a qubit with T1 ≈ 207.7 µs can have at most T2 ≈ 415 µs. If you ever see a T2 above that, something is wrong with the data you are reading.

The remaining part of the signal that survives after time $t$ is described by simple exponentials:

```math
p_1(t) = p_1(0)\,e^{-t/T_1}, \qquad |\rho_{01}(t)| = |\rho_{01}(0)|\,e^{-t/T_2}
```

Here $p_1$ is the probability that the qubit is in the excited state, and $\rho_{01}$ is the number that stores its phase information. Both shrink exponentially, and T1 and T2 are simply the "time constants" of that shrinking.

### How to look at the hardware in code

```python
from qiskit_ibm_runtime import QiskitRuntimeService

service = QiskitRuntimeService()              # uses your saved IBM account
backend = service.backend("ibm_fez")          # pick one machine by name

print(backend.name, backend.num_qubits)       # name and qubit count
print(backend.processor_type)                 # chip family
print(backend.operation_names)                # basis gates it can run

target = backend.target
q0 = target.qubit_properties[0]
print("T1 of qubit 0 (s):", q0.t1)
print("T2 of qubit 0 (s):", q0.t2)
print("Readout error of qubit 0:", target["measure"][(0,)].error)
```

If you are unsure what a name means, look it up in the glossary below. The lecture notes explain each field in more depth, including how IBM measures T2 (a "Hahn echo" experiment) and how to read per-gate errors and durations from `backend.target`.

### Finding machines that support fractional gates

```python
from qiskit_ibm_runtime import QiskitRuntimeService

service = QiskitRuntimeService()

# Ask only for Heron machines that offer fractional gates
backends = service.backends(use_fractional_gates=True)

for b in backends:
    print(b.name, b.num_qubits, b.status().pending_jobs, "jobs waiting")
```

Pick the one with the fewest waiting jobs if you want the shortest queue. Without the flag, the basis gates are `cz, id, rz, sx, x`. With it, they become `cz, id, rx, rz, rzz, sx, x`.

---

## Part 2: Fractional gates in plain language

### Gates as rotations

Every single-qubit gate is a rotation of a point on a sphere (called the **Bloch sphere**). The north pole is the state `|0⟩`, the south pole is `|1⟩`. A gate such as `RX(θ)` rotates that point by the angle θ around the x-axis.

- `RZ(θ)` rotates around the z-axis. On IBM hardware this is "virtual": it costs no time and adds no error, because it is done by changing the phase reference of the control signals.
- `SX` is a fixed quarter-turn around x (θ = π/2). `X` is a fixed half-turn (θ = π). These are real, calibrated pulses.

### The problem

The hardware only has fixed turns (`SX`, `X`). To rotate around x by an arbitrary angle, it combines them with free `RZ` rotations. That costs **two `SX` pulses** every time:

```math
R_X(\theta) \simeq R_Z\!\left(\tfrac{\pi}{2}\right)\, \mathrm{SX}\, R_Z(\theta + \pi)\, \mathrm{SX}\, R_Z\!\left(\tfrac{\pi}{2}\right)
```

The symbol ≃ means "equal up to a global phase", which nobody can ever measure.

For two qubits the only native entangling gate is `CZ`. An arbitrary-angle `RZZ(θ)` costs **two `CZ` gates**.

### The fix

A **fractional gate** is a gate whose angle is chosen freely when you write the circuit, and which the hardware runs as a single pulse sequence:

| Gate | Allowed angles | Replaces |
| --- | --- | --- |
| `RX(θ)` | any real θ | two `SX` pulses |
| `RZZ(θ)` | 0 ≤ θ ≤ π/2 | two `CZ` gates plus single-qubit gates |

They are available on **Heron** machines only, and you switch them on with `use_fractional_gates=True`.

---

## Part 3: The mathematics, step by step

This section is the short version. The lecture notes contain the full proofs. Every step below only needs matrix multiplication and the cosine and sine you learned at school.

### Step 1: A gate is $e^{-i\alpha P}$ for a simple matrix $P$

Take any matrix $P$ with the property $P^2 = I$ (it squares to the identity matrix). The Pauli matrices $X$, $Y$, $Z$ all have this property, and so does the two-qubit matrix $Z \otimes Z$. For such a matrix the exponential collapses to a short formula, because the even powers of $P$ become $I$ and the odd powers become $P$:

```math
e^{-i\alpha P} = \cos\alpha \; I \; - \; i \sin\alpha \; P
```

Every gate in this guide is an application of this one line.

### Step 2: The two fractional gates

Set $\alpha = \theta/2$ and choose $P = X$ for the single-qubit gate:

```math
R_X(\theta) = e^{-i\theta X/2} = \begin{pmatrix} \cos\frac{\theta}{2} & -i\sin\frac{\theta}{2} \\ -i\sin\frac{\theta}{2} & \cos\frac{\theta}{2} \end{pmatrix}
```

Choose $P = Z \otimes Z$ for the two-qubit gate. This matrix is diagonal, so the gate just multiplies each basis state by a phase that depends on whether the two bits are equal (even parity) or different (odd parity):

```math
R_{ZZ}(\theta) = e^{-i\theta\, Z\otimes Z/2} = \begin{pmatrix} e^{-i\theta/2} & 0 & 0 & 0 \\ 0 & e^{+i\theta/2} & 0 & 0 \\ 0 & 0 & e^{+i\theta/2} & 0 \\ 0 & 0 & 0 & e^{-i\theta/2} \end{pmatrix}
```

The rows and columns are ordered $|00\rangle, |01\rangle, |10\rangle, |11\rangle$. States `00` and `11` get one phase, states `01` and `10` get the opposite phase. The difference between them is θ, and that relative phase is what creates entanglement.

### Step 3: What `RZZ(θ)` costs without fractional gates

Conjugating a gate on one qubit by a `CNOT` turns a $Z$ on the target into $Z \otimes Z$. That gives:

```math
R_{ZZ}(\theta) = \mathrm{CX}\,\bigl(I \otimes R_Z(\theta)\bigr)\,\mathrm{CX}
```

Writing each `CX` as a `CZ` between two Hadamard gates ($H$) on the second qubit $b$ gives the form the hardware actually runs:

```math
R_{ZZ}(\theta) = H_b \cdot \mathrm{CZ} \cdot R_X(\theta)_b \cdot \mathrm{CZ} \cdot H_b
```

So one `RZZ(θ)` becomes **two `CZ` gates**, one `RX(θ)` (itself two `SX` pulses), and some Hadamards. This is the cost that the fractional gate removes.

### Step 4: Why the angle window is 0 to π/2

Any two-qubit gate is, up to single-qubit gates on each side, a gate described by three numbers $(c_1, c_2, c_3)$. These live in a small region called the **Weyl chamber**:

```math
\frac{\pi}{4} \ge c_1 \ge c_2 \ge |c_3|
```

The gate `RZZ(θ)` sits on one edge of this region at $(\theta/2,\, 0,\, 0)$. At θ = 0 it is the identity. At θ = π/2 it equals `CZ` up to single-qubit gates. So the allowed window 0 ≤ θ ≤ π/2 is exactly the whole edge from "do nothing" to "CZ". Every angle in between needs two `CZ` gates in the old method and one fractional gate in the new method.

### Step 5: Angle folding, so any angle becomes legal

Three exact facts, each following from Step 1:

1. $R_{ZZ}(\theta + 2\pi) = -R_{ZZ}(\theta)$. Only a global sign changes, which is unobservable.
2. $R_{ZZ}(\theta + \pi) = R_{ZZ}(\theta)\,(-i\,Z\otimes Z)$. The extra part is a pair of single-qubit $Z$ rotations, which are free.
3. $X_a\,R_{ZZ}(\theta)\,X_a = R_{ZZ}(-\theta)$. Flipping one qubit with $X$ changes the sign of the angle.

Together they map any requested angle θ into the hardware window. The folded angle is:

```math
\theta' = \left|\,\bigl((\theta + \tfrac{\pi}{2}) \bmod \pi\bigr) - \tfrac{\pi}{2}\,\right|
```

This is a triangle wave with period π and height π/2. Qiskit's `FoldRzzAngle` pass does this for you, **but only if θ is already a number when you compile**. That is why the workflow is "fill in the numbers first, then compile".

A small worked example: take θ = 5.0.

- $\theta + \pi/2 = 6.5708$
- Subtract $\pi$ twice (6.5708 − 3.1416 − 3.1416 = 0.2876), so $(\theta + \pi/2) \bmod \pi = 0.2876$
- $\theta' = |0.2876 - 1.5708| = 1.2832$

So an `RZZ(5.0)` is executed as `RZZ(1.2832)` plus free `Z` rotations and possibly a pair of `X` gates. You can check this yourself in the notes' exercises.

### Step 6: The same idea for one qubit

For `RX(θ)` any angle is accepted directly, so no folding is needed. The saving comes from replacing the two-pulse `SX` recipe with one pulse of the same length as an `SX`, with a variable amplitude.

---

## Part 4: Pros and cons

### Pros

- **Fewer two-qubit gates.** One `RZZ` instead of two `CZ`. Two-qubit gates are the noisiest operation, so this matters most.
- **Fewer single-qubit pulses.** One `RX` instead of two `SX`.
- **Shorter circuits.** Qubits lose information while they wait, so a shorter circuit loses less.
- **Compact programs.** Each rotation in your algorithm stays one gate.

### Cons and current limits

- **Heron only.** Other processor families do not have them.
- **Narrow `RZZ` window.** Only 0 to π/2 is native. Other angles rely on folding.
- **Bind before you compile.** Angles must be numbers first, so you compile once per data point instead of once overall.
- **Some error-reduction tools are unavailable.** Pauli twirling, probabilistic error cancellation (PEC), and zero-noise extrapolation using probabilistic error amplification do not work with fractional gates. TREX readout mitigation does.
- **No dynamic circuits** (mid-circuit measurement with classical decisions).
- **The compiler cannot compare angles by error.** In the machine description, a fractional gate's error is a copy of the matching ordinary gate's error.
- **A known issue in `qiskit-ibm-runtime` 0.47.0** (issue 2441). Upgrade, or run the `FoldRzzAngle` pass yourself.

Check the current IBM documentation before relying on any of these limits, because they change between releases.

### A simple decision rule

| Your situation | Use fractional gates? |
| --- | --- |
| Many arbitrary-angle two-qubit gates (Ising-type circuits, feature maps, Trotter steps) | Yes |
| You need twirling, PEC or PEA-based extrapolation | No |
| You use dynamic circuits | No |
| A parameter sweep with angles outside 0 to π/2 | Yes, but bind the numbers first |

---

## Part 5: A worked example (quantum kernels)

IBM's tutorial uses fractional gates in a "quantum kernel" experiment. You do not need to know what a kernel is. All you need is the following.

**What is measured.** Two data points $x$ and $y$ are loaded into the circuit. The circuit runs the loading for $x$, then undoes the loading for $y$, then measures all qubits. The probability of getting all zeros is the kernel value:

```math
K(x, y) = \bigl|\langle \Phi(y) | \Phi(x) \rangle\bigr|^2 = P(0^n)
```

When $x = y$ the ideal answer is exactly 1. So any shortfall below 1 on hardware is a direct measure of noise. That makes `K(x, x)` a handy score for comparing "with fractional gates" against "without".

**What the circuit does.** After one layer of Hadamard gates, the circuit repeats rounds of:

- `RZ(x_i)` on each qubit, and
- `RZZ(θ_ij)` between neighbouring qubits, with $\theta_{ij} = (\pi - x_i)(\pi - x_j)$.

Every gate after the Hadamards only adds phases to basis states, so the state has a closed form (here $z_i = \pm 1$ and `reps` is the number of rounds):

```math
\langle z | \Phi(x) \rangle = 2^{-n/2}\exp\!\Bigl(-\tfrac{i\,\mathrm{reps}}{2}\Bigl[\sum_i x_i z_i + \sum_{(i,j)} \theta_{ij}\, z_i z_j\Bigr]\Bigr)
```

**Where the angles land.** In the tutorial's data every `RZZ` angle falls between about 4.59 and 9.87, entirely outside the hardware window of 0 to π/2. So every single `RZZ` is folded using the rules in Part 3, Step 5.

**What you gain (four-qubit circuit):**

| Gate | Without fractional gates | With fractional gates |
| --- | --- | --- |
| `cz` | 36 | 0 |
| `rzz` | 0 | 18 |
| `sx` | 80 | 0 |
| `x` | 0 | 13 |
| `rx` | 0 | 9 |
| `rz` (virtual, free) | 130 | 35 |

Two-qubit gates fall from 36 to 18. Physical single-qubit pulses (`sx`, `x`, `rx`) fall from 80 to 22.

**Noisy-simulator results for `K(x, x)` (ideal value is 1):**

| Qubits | Without | With | Gain |
| --- | --- | --- | --- |
| 4 | 0.8625 | 0.929 | +0.0665 |
| 6 | 0.7605 | 0.882 | +0.1215 |
| 8 | 0.702 | 0.8645 | +0.1625 |
| 10 | 0.671 | 0.817 | +0.146 |

The tutorial also runs on real hardware from 4 to 40 qubits, but shows plots rather than tables, so this guide quotes no hardware numbers.

**A second variation: "RX-only" mode and the ZXZ rule.** If you remove `rzz` from the machine description and keep only `rx`, the compiler can write any single-qubit gate as $R_Z(a)\,R_X(b)\,R_Z(c)$, which needs only one `RX` pulse instead of two `SX` pulses. In the tutorial's `n_local` example, physical single-qubit pulses go from 16 (conventional) to 15 (RX-only, with the usual decomposition) to 9 (RX-only with the ZXZ rule), and the two-qubit gate count stays at 4 `CZ`.

---

## Try it yourself

### 1. Install

```bash
pip install "qiskit>=2.0" "qiskit-ibm-runtime>=0.48" numpy
```

Version 0.48 or later avoids the known issue in 0.47.0.

### 2. Save your IBM Quantum credentials (once)

Create a free account on the IBM Quantum Platform, copy your API key, then run:

```python
from qiskit_ibm_runtime import QiskitRuntimeService

QiskitRuntimeService.save_account(
    token="PASTE_YOUR_API_KEY_HERE",
    overwrite=True,
)
```

If the call asks for a channel or an instance, use the values shown on your account page.

### 3. Run the snippets above

The two code blocks in Part 1 are complete and can be pasted into a notebook cell. The notes contain longer runnable cells for the kernel example and for the RX-only variation.

---

## Suggested reading order

| If you are... | Read |
| --- | --- |
| Completely new | The 10-minute version, then Part 1, then Part 2, then the glossary |
| Comfortable with matrices | Parts 3 to 5, then the lecture notes' exercises |
| Preparing to teach this | The slide deck first, then the lecture notes for the derivations |
| Only want the practical decision | The 10-minute version and the decision rule in Part 4 |

## Honest limitations of this material

- The two long runnable code cells in the lecture notes were **not executed**, because they need IBM credentials. Treat them as carefully checked but untested.
- The mathematical identities in this README were checked numerically before being written down.
- Some references and standard gate-count results (for example, that an arbitrary-angle `RZZ` needs two `CZ`) are cited from memory in the notes, and the notes mark them.
- Hardware numbers change daily. The values quoted from IBM's documentation (for example, `ibm_fez` with 156 qubits) are examples from the time of writing.
- The docs disagree slightly about the `RZZ` window ends (`0 < θ ≤ π/2`, `[0, π/2]` and `0 < θ < π/2` all appear). Check the current page before relying on the exact endpoints.

---

## Glossary

| Word | Meaning |
| --- | --- |
| **Backend** | A quantum computer (or simulator) that you send circuits to |
| **Basis gates** | The short list of gates a machine can run natively |
| **Bloch sphere** | A sphere whose points represent all possible states of one qubit |
| **Circuit** | An ordered list of gates applied to qubits |
| **CZ** | A two-qubit gate that flips the sign of the state when both qubits are `1` |
| **Coherence** | How long a qubit keeps its quantum information (T1 and T2 measure this) |
| **Entanglement** | A link between qubits that cannot be described qubit by qubit |
| **Fractional gate** | A gate with a freely chosen angle that runs as one gate (`RX`, and `RZZ` for angles 0 to π/2) |
| **Gate** | An operation on one or two qubits |
| **Global phase** | A factor that multiplies the whole state and cannot be measured |
| **Heron** | IBM's current family of quantum processors |
| **Hadamard (H)** | A single-qubit gate that turns `0` or `1` into an equal mix of both |
| **Pauli matrices** | The three basic single-qubit matrices $X$, $Y$, $Z$ |
| **Pulse** | The microwave signal that physically performs a gate |
| **Qubit** | The quantum version of a bit |
| **RX, RZ, RZZ** | Rotation gates around the x, z axes, and the two-qubit z-z rotation |
| **SX** | A quarter-turn around x. Short for "square root of X" |
| **T1, T2** | Energy-relaxation time and phase-coherence time |
| **Target** | The Qiskit object that lists a machine's gates with their errors and durations |
| **Transpile** | To rewrite a circuit using only the gates a specific machine supports |
| **Virtual gate** | A gate done in software by shifting a phase reference, at zero cost and zero error (`RZ`) |
| **Weyl chamber** | The small region that classifies every two-qubit gate up to single-qubit gates |

## Sources

- IBM Quantum documentation: [QPU information](https://quantum.cloud.ibm.com/docs/en/guides/qpu-information)
- IBM Quantum documentation: [Fractional gates](https://quantum.cloud.ibm.com/docs/en/guides/fractional-gates)
- IBM Quantum tutorial: [Quantum kernels with fractional gates](https://quantum.cloud.ibm.com/docs/en/tutorials/fractional-gates)
- Derivations, proofs, exercises and runnable cells: the lecture notes in this repository
