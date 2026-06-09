# Sunrise HPC — Hybrid MPI + OpenMP + CUDA Fortran Parallel Engine for EDA

> Hybrid parallel GPU engine for EDA associative reductions on NVIDIA DGX A100.  
> 0% numerical error. 8-GPU MPI+CUDA pipeline verified on a 10M-gate synthetic netlist.

---

## Overview

Sunrise HPC is a hybrid parallel computing engine targeting EDA (Electronic Design Automation) associative reductions: area sum, power sum, and max fanout. Built using MPI + OpenMP + CUDA Fortran, it was benchmarked across four pipeline phases on an NVIDIA DGX A100 server.

Publication target: **DAC or ICCAD** — using the real `soc_top` Innovus netlist from the RISC-V SoC thesis project.

---

## Pipeline Phases

| Phase | Implementation | Threads/GPUs | Description |
|-------|---------------|--------------|-------------|
| 1 | Serial Fortran | 1 | Baseline sequential reference |
| 2 | OpenMP | 256 threads | CPU parallel baseline |
| 3 | CUDA Fortran | 1 GPU (A100) | Single-GPU acceleration |
| 4 | MPI + CUDA | 8 GPUs (A100) | Full distributed multi-GPU |

---

## Results

### Numerical Accuracy

| Phase | Numerical Error vs Serial |
|-------|--------------------------|
| Serial Fortran | 0% (reference) |
| OpenMP 256-thread | **0%** ✅ |
| CUDA single-GPU | **0%** ✅ |
| MPI+CUDA 8-GPU | **0%** ✅ |

All four phases verified to **0% numerical error** on a 10M-gate synthetic netlist.

### Timing Breakdown

Timing is reported across three separate components:

| Component | Description |
|-----------|-------------|
| T_parse | File I/O + netlist parsing time |
| T_transfer | Host-to-device memory transfer |
| T_compute | GPU kernel execution time |

> **Key finding:** T_parse dominates at ~97% of wall time — confirming an I/O bottleneck, not a compute bottleneck. The GPU compute itself is highly efficient; the system is I/O bound.

---

## Benchmark Target

| Netlist | Gates | Source |
|---------|-------|--------|
| Synthetic benchmark | 10M gates | Used for development and verification |
| Real target | soc_top Innovus netlist | RISC-V SoC thesis — for DAC/ICCAD publication |

---

## Reductions Implemented

| Reduction | Operation |
|-----------|-----------|
| Area sum | Σ(cell_area) across all gates |
| Power sum | Σ(cell_power) across all gates |
| Max fanout | max(fanout) across all nets |

Scope: associative reductions only. STA (static timing analysis) was intentionally excluded per professor feedback to maintain focused benchmark scope.

---

## Architecture

```
Netlist File (10M gates)
        │
        ▼  T_parse (~97% wall time)
  MPI Rank 0 — File Parser
        │
        ├── Distribute partitions to MPI Ranks 1–7
        │
  ┌─────▼──────┐  ┌────────────┐  ...  ┌────────────┐
  │  MPI Rank 0 │  │ MPI Rank 1 │       │ MPI Rank 7 │
  │  A100 GPU 0 │  │ A100 GPU 1 │       │ A100 GPU 7 │
  └─────┬───────┘  └─────┬──────┘       └─────┬──────┘
        │     T_transfer │                     │
        │     T_compute  │                     │
        └────────────────┴──── MPI_Reduce ─────┘
                                    │
                              Final Results
                    (area_sum, power_sum, max_fanout)
```

---

## Environment & Dependencies

| Component | Version / Detail |
|-----------|-----------------|
| Server | NVIDIA DGX A100 |
| GPUs | 8× NVIDIA A100 80GB |
| Compiler | nvfortran (NVHPC 22.11) |
| MPI | OpenMPI (mpif90 PATH override required) |
| OpenMP | Built into nvfortran |
| CUDA Fortran | NVHPC CUDA Fortran dialect |
| OS | Linux |

---

## Known Environment Fixes

```bash
# 1. mpif90 PATH override (forces nvfortran, not gfortran)
export PATH=/opt/nvidia/hpc_sdk/Linux_x86_64/22.11/comm_libs/openmpi4/openmpi-4.0.5/bin:$PATH

# 2. Reload nvhpc module after environment changes
module unload nvhpc
module load nvhpc/22.11

# 3. Makefiles written in Python to avoid tab-corruption issues
python3 gen_makefile.py > Makefile
```

---

## How to Run

```bash
# Phase 1 — Serial
nvfortran -O2 -o serial_reduce serial_reduce.f90
./serial_reduce netlist_10M.txt

# Phase 2 — OpenMP
nvfortran -O2 -mp -o omp_reduce omp_reduce.f90
OMP_NUM_THREADS=256 ./omp_reduce netlist_10M.txt

# Phase 3 — CUDA Fortran single GPU
nvfortran -O2 -cuda -gpu=cc80 -o cuda_reduce cuda_reduce.cuf
./cuda_reduce netlist_10M.txt

# Phase 4 — MPI + CUDA 8-GPU
nvfortran -O2 -cuda -gpu=cc80 -o mpi_cuda_reduce mpi_cuda_reduce.cuf
mpirun -np 8 ./mpi_cuda_reduce netlist_10M.txt
```

---

## Repository Structure

```
sunrise-hpc-cuda/
├── src/
│   ├── serial_reduce.f90       — Phase 1: serial reference
│   ├── omp_reduce.f90          — Phase 2: OpenMP 256-thread
│   ├── cuda_reduce.cuf         — Phase 3: CUDA Fortran single-GPU
│   └── mpi_cuda_reduce.cuf     — Phase 4: MPI+CUDA 8-GPU
├── scripts/
│   ├── gen_makefile.py         — Python Makefile generator (tab-safe)
│   └── run_all_phases.sh       — Full benchmark pipeline
├── data/
│   └── netlist_10M.txt         — 10M-gate synthetic benchmark netlist
├── results/
│   └── timing_breakdown.csv    — T_parse / T_transfer / T_compute per phase
└── docs/
    └── sunrise_hpc_report.pdf
```

---

## Publication Path

Target venues: **DAC 2026** or **ICCAD 2026**

The paper will benchmark all four phases against the real `soc_top` Innovus netlist from the RISC-V SoC thesis project, replacing the synthetic 10M-gate netlist with real EDA production data.

---

## Author

**Vishnukumar Varatharaja Perumal**  
MS Computer Engineering — San Diego State University (May 2027)  
ASIC Design Researcher — SDSU IoT Laboratory  
[linkedin.com/in/vishnukumarraj](https://linkedin.com/in/vishnukumarraj) · vishnukumarraj414@gmail.com
