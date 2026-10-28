<div align="center">

# ⚡ Mico-gpu

**A from-scratch GPU in Verilog — built for people who want to understand what's actually happening inside the chip.**

![Language](https://img.shields.io/badge/Language-SystemVerilog-blueviolet?style=flat-square)
![Files](https://img.shields.io/badge/Source%20Files-%3C15-success?style=flat-square)
![License](https://img.shields.io/badge/License-Educational-orange?style=flat-square)
![Author](https://img.shields.io/badge/Author-Aryan%20Gahlot-blue?style=flat-square)

</div>

---

## 📌 Why does this exist?

Learning how a CPU works is easy — tutorials, books, and open-source implementations are everywhere.

**GPUs are a completely different story.**

The GPU industry is fiercely competitive. Architectural details are kept proprietary, and while GPU *programming* resources are abundant, resources explaining GPU *hardware internals* are almost nonexistent.

Projects like [Miaow](https://github.com/VerticalResearchGroup/miaow) and [VeriGPU](https://github.com/hughperkins/VeriGPU/tree/main) exist — but they target full feature-parity with production hardware. Great for production, overwhelming for learning.

> **Mico-gpu strips away everything non-essential and exposes the three ideas that matter most:**

| Pillar | Question it answers |
|--------|-------------------|
| 🏗 **Architecture** | What does a GPU actually look like structurally? |
| ⚡ **Parallelization** | How is SIMD physically built in hardware? |
| 🧠 **Memory** | How does a GPU cope with bandwidth limitations? |

---

## 🗺 Table of Contents

| Section | What you'll find |
|---------|-----------------|
| [Architecture](#-architecture) | GPU, memory, and core layout |
| [ISA](#-isa) | The 11-instruction set |
| [Execution](#-execution) | How instructions flow through the pipeline |
| [Kernels](#-kernels) | Matrix addition & multiplication examples |
| [Simulation](#-simulation) | How to run it yourself |
| [Advanced Topics](#-advanced-functionality) | What real GPUs add on top |
| [Next Steps](#-next-steps) | Roadmap & contribution guide |

---

## 🏗 Architecture

<p float="left">
  <img src="/docs/images/gpu.png" alt="GPU Architecture Diagram" width="48%">
  <img src="/docs/images/core.png" alt="Core Architecture Diagram" width="48%">
</p>

### The Big Picture

Mico-gpu runs **one kernel at a time**. Launching a kernel is a four-step process:

```
1. Write kernel code → program memory
2. Write input data  → data memory
3. Set thread count  → device control register
4. Assert start signal → execution begins
```

The GPU is composed of five main units:

```
┌─────────────────────────────────────────────┐
│                  Mico-gpu                   │
│                                             │
│  ┌─────────┐   ┌────────────┐              │
│  │   DCR   │   │ Dispatcher │              │
│  └─────────┘   └────────────┘              │
│                                             │
│  ┌──────┐ ┌──────┐ ┌──────┐  (N cores)    │
│  │ Core │ │ Core │ │ Core │               │
│  └──────┘ └──────┘ └──────┘               │
│                                             │
│  ┌────────────────┐  ┌───────────────────┐ │
│  │ Data Mem Ctrl  │  │ Prog Mem Ctrl     │ │
│  └────────────────┘  └───────────────────┘ │
│                   ┌───────┐                │
│                   │ Cache │                │
│                   └───────┘                │
└─────────────────────────────────────────────┘
```

---

### 🎛 Device Control Register

Holds metadata for kernel execution. In Mico-gpu this is a single value: `thread_count` — the total threads to launch.

---

### 🚦 Dispatcher

After launch, the dispatcher groups threads into **blocks** and routes each block to an available compute core. Once all blocks are processed, it signals kernel completion.

---

### 💾 Memory

Mico-gpu separates **data memory** and **program memory** for clarity.

#### Specifications

| | Data Memory | Program Memory |
|---|---|---|
| **Addressability** | 8-bit (256 rows) | 8-bit (256 rows) |
| **Data Width** | 8-bit (values 0–255) | 16-bit (one instruction/row) |

#### Memory Controllers

Global memory has fixed bandwidth. When simultaneous requests from cores exceed capacity, the memory controller:
- **Throttles** outgoing requests to match bandwidth
- **Routes** responses back to the correct core/thread

#### Cache *(WIP)*

Repeated requests for the same data skip global memory entirely — fetched data is stored on-device in SRAM and served from there on subsequent requests, preserving bandwidth for new data.

---

### 🖥 Core

Each core processes one **block** at a time. For every thread in that block, it provides a dedicated:

```
Thread N
├── Register File  ← thread-local data storage
├── ALU            ← arithmetic & logic
├── LSU            ← memory load/store
└── PC             ← instruction pointer + NZP register
```

Shared across all threads in a core:
- **Scheduler** — orchestrates instruction execution
- **Fetcher** — pulls instructions from program memory
- **Decoder** — translates instructions into control signals

#### Scheduler

Runs all threads **in lock-step** — every thread executes the same instruction simultaneously. A new block is only picked up after the current one finishes.

> Real GPUs use **pipelining** and **warp scheduling** to overlap execution and hide memory latency.

#### Register Files

Every thread has its own private register file — this is what enables SIMD. Same instruction, different data per thread.

Three **read-only** special registers are baked into every register file:

| Register | Value |
|----------|-------|
| `%blockIdx` | Index of the current block |
| `%blockDim` | Number of threads per block |
| `%threadIdx` | Index of this thread within the block |

#### ALU

Handles arithmetic: `ADD` `SUB` `MUL` `DIV`

Also handles `CMP` — computes the sign of the difference between two registers and writes **Negative / Zero / Positive** into the **NZP register** inside the PC unit (used for branching).

#### LSU

Handles `LDR` and `STR` — including the async wait cycles while the memory controller processes the request.

#### PC

Tracks the next instruction. Increments by 1 by default. With `BRnzp`, checks the NZP register and conditionally jumps to a target address — **this is how loops and conditionals work**.

> Mico-gpu assumes all threads converge to the same PC after every instruction. Real GPUs handle **branch divergence** when threads split to different addresses.

---

## 📋 ISA

![ISA Diagram](/docs/images/isa.png)

Mico-gpu implements **11 instructions** — enough to run real SIMD kernels.

| Instruction | Type | Description |
|-------------|------|-------------|
| `ADD` | Arithmetic | `Rd = Rs1 + Rs2` |
| `SUB` | Arithmetic | `Rd = Rs1 - Rs2` |
| `MUL` | Arithmetic | `Rd = Rs1 * Rs2` |
| `DIV` | Arithmetic | `Rd = Rs1 / Rs2` |
| `CMP` | Compare | Set NZP register based on `Rs1 - Rs2` |
| `BRnzp` | Branch | Jump if NZP matches condition |
| `LDR` | Memory | Load from global memory |
| `STR` | Memory | Store to global memory |
| `CONST` | Immediate | Load constant into register |
| `RET` | Control | Signal end of thread execution |

**Register layout:** 4-bit encoding → 16 registers total

- `R0` – `R12` → General purpose (read/write)
- `R13` → `%blockIdx` *(read-only)*
- `R14` → `%blockDim` *(read-only)*
- `R15` → `%threadIdx` *(read-only)*

---

## ⚙ Execution

### Core Execution Flow

Each instruction steps through 6 stages:

```
  ┌─────────┐
  │  FETCH  │  Pull instruction at current PC from program memory
  └────┬────┘
       ▼
  ┌─────────┐
  │ DECODE  │  Translate instruction into control signals
  └────┬────┘
       ▼
  ┌─────────┐
  │ REQUEST │  Issue memory request (LDR / STR only)
  └────┬────┘
       ▼
  ┌─────────┐
  │  WAIT   │  Wait for memory response (LDR / STR only)
  └────┬────┘
       ▼
  ┌─────────┐
  │ EXECUTE │  Perform the computation
  └────┬────┘
       ▼
  ┌─────────┐
  │ UPDATE  │  Write results to register files / NZP register
  └─────────┘
```

### Thread Execution

![Thread Diagram](/docs/images/thread.png)

Each thread follows the above path independently on its own data — closely resembling a CPU datapath. The key difference: the SIMD special registers (`%blockIdx`, `%blockDim`, `%threadIdx`) give every thread a unique identity while running identical instructions.

---

## 🧪 Kernels

Two proof-of-concept kernels ship with Mico-gpu to demonstrate SIMD programming end-to-end.

---

### ➕ Matrix Addition

**Task:** Add two `1×8` matrices element-wise using **8 parallel threads** (one per element).

**Demonstrates:** SIMD addressing with `%threadIdx`, async memory with `LDR`/`STR`

```asm
; matadd.asm
.threads 8
.data 0 1 2 3 4 5 6 7          ; matrix A (1×8)
.data 0 1 2 3 4 5 6 7          ; matrix B (1×8)

MUL R0, %blockIdx, %blockDim
ADD R0, R0, %threadIdx         ; i = blockIdx * blockDim + threadIdx

CONST R1, #0                   ; baseA
CONST R2, #8                   ; baseB
CONST R3, #16                  ; baseC (output)

ADD R4, R1, R0                 ; addr(A[i])
LDR R4, R4                     ; A[i] = mem[addr]

ADD R5, R2, R0                 ; addr(B[i])
LDR R5, R5                     ; B[i] = mem[addr]

ADD R6, R4, R5                 ; C[i] = A[i] + B[i]

ADD R7, R3, R0                 ; addr(C[i])
STR R7, R6                     ; mem[addr] = C[i]

RET
```

---

### ✖ Matrix Multiplication

**Task:** Multiply two `2×2` matrices using **4 parallel threads** (one per output element).

**Demonstrates:** Inner loop with `CMP` + `BRnzp` branching, multi-step computation per thread

```asm
; matmul.asm
.threads 4
.data 1 2 3 4                  ; matrix A (2×2)
.data 1 2 3 4                  ; matrix B (2×2)

MUL R0, %blockIdx, %blockDim
ADD R0, R0, %threadIdx         ; i = blockIdx * blockDim + threadIdx

CONST R1, #1                   ; increment
CONST R2, #2                   ; N (inner dimension)
CONST R3, #0                   ; baseA
CONST R4, #4                   ; baseB
CONST R5, #8                   ; baseC

DIV R6, R0, R2                 ; row = i / N
MUL R7, R6, R2
SUB R7, R0, R7                 ; col = i % N

CONST R8, #0                   ; acc = 0
CONST R9, #0                   ; k = 0

LOOP:
  MUL R10, R6, R2
  ADD R10, R10, R9
  ADD R10, R10, R3             ; addr(A[row][k])
  LDR R10, R10

  MUL R11, R9, R2
  ADD R11, R11, R7
  ADD R11, R11, R4             ; addr(B[k][col])
  LDR R11, R11

  MUL R12, R10, R11
  ADD R8, R8, R12              ; acc += A[row][k] * B[k][col]

  ADD R9, R9, R1               ; k++

  CMP R9, R2
  BRn LOOP                    ; repeat while k < N

ADD R9, R5, R0                 ; addr(C[i])
STR R9, R8                     ; mem[addr] = acc

RET
```

---

## 🚀 Simulation

Run either kernel from scratch with full execution traces.

### Prerequisites

```bash
# Verilog compiler
brew install icarus-verilog

# Python test framework
pip3 install cocotb

# SystemVerilog → Verilog converter
# Download from: https://github.com/zachjs/sv2v/releases
# Unzip and add binary to $PATH
```

### Setup

```bash
mkdir build
```

### Run

```bash
make test_matadd    # simulate matrix addition
make test_matmul    # simulate matrix multiplication
```

### Output

Logs land in `test/logs/` containing three sections:

```
[1] Initial data memory state   → your input matrices
[2] Cycle-by-cycle trace        → every thread, every core, every cycle
[3] Final data memory state     → the computed result
```

![Execution Trace](/docs/images/trace.png)

---

## 🔬 Advanced Functionality

Mico-gpu is intentionally minimal. Here's what production GPUs layer on top — and why each matters:

| Feature | What it does | Why it matters |
|---------|-------------|----------------|
| **Multi-level Cache** | Caches data at multiple levels, some private to cores | Reduces global memory round-trips dramatically |
| **Shared Memory** | Fast on-chip memory shared between threads in a block | Lets threads exchange intermediate results |
| **Memory Coalescing** | Merges adjacent memory requests into one transaction | Cuts addressing overhead for sequential access patterns |
| **Pipelining** | Overlaps multiple in-flight instructions | Keeps execution units busy during memory waits |
| **Warp Scheduling** | Interleaves thread batches to hide latency | Maximizes core utilization across many threads |
| **Branch Divergence** | Handles threads branching to different PCs | Enables truly independent per-thread control flow |
| **Barriers** | Lets threads wait for each other at sync points | Required for safe shared memory usage |

---

## 🛣 Next Steps

Planned work — PRs welcome on any of these:

- [ ] Instruction cache
- [ ] Tiny Tapeout 7 adapter
- [ ] Basic branch divergence
- [ ] Basic memory coalescing
- [ ] Basic pipelining
- [ ] Control flow & register optimizations
- [ ] Simple graphics kernel or graphics hardware demo

---

<div align="center">

---

*© 2026 **Aryan Gahlot**. All rights reserved.*

*Free to use, study, and modify for personal and educational purposes with attribution.*

</div>
