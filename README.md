# ⚛️ Quantum ESPRESSO: Full GPU & CPU Optimization Guide
### Optimized for Arch Linux + NVIDIA Turing + AMD Ryzen

This repository provides a failsafe, step-by-step guide to installing **Quantum ESPRESSO 7.5** with full GPU acceleration (CUDA) and CPU optimization. This guide is specifically designed to overcome common pitfalls on Arch Linux, such as MPI hangs, library relocation errors, and architecture detection failures.

---

## 🖥️ Target System Specifications
This configuration is verified and benchmarked on the following hardware:

*   **Operating System**: Arch Linux (Rolling)
*   **CPU**: AMD Ryzen 5 3600 (Zen 2 Architecture)
*   **GPU**: NVIDIA GeForce GTX 1660 SUPER (Turing / Compute Capability 7.5)
*   **Compiler Infrastructure**: NVIDIA HPC SDK (NVHPC)
*   **Shell Environment**: Zsh (HyDE/Oh-My-Zsh compatible)

---

## 🚀 1. Prerequisites
Before starting, ensure your system has the base-development tools and the official NVIDIA HPC compilers.

```bash
# Update system and install core build tools
sudo pacman -Syu base-devel git cmake gcc openmpi

# Install the NVIDIA HPC SDK (Official repository)
sudo pacman -S nvhpc
```

---

## 🧠 2. Smart Environment Configuration
Quantum ESPRESSO requires direct access to high-performance libraries. We use a **"Smart Detection"** script to ensure your setup never breaks, even after system updates.

Add this block to your `~/.zshrc` or `~/.config/zsh/.zshrc`:

```bash
# --- Quantum ESPRESSO & NVIDIA HPC Environment ---
export NVARCH=Linux_x86_64
export NVVERSION=$(ls /opt/nvidia/hpc_sdk/$NVARCH/ | sort -V | tail -n 1) # Auto-detect latest
export NVROOT=/opt/nvidia/hpc_sdk/$NVARCH/$NVVERSION
export NVCOMPILERS=$NVROOT/compilers
export NVCOMM=$NVROOT/comm_libs/mpi
export NVOMPI=$(find $NVROOT/comm_libs -name "ompi" -type d | head -n 1) # Auto-detect MPI

export OPAL_PREFIX=$NVOMPI
export PATH=$NVCOMPILERS/bin:$NVCOMM/bin:$PATH
export LD_LIBRARY_PATH=$NVCOMPILERS/lib:$NVOMPI/lib:$NVCOMM/lib:$LD_LIBRARY_PATH

# --- Definite Anti-Hang Flags for Consumer Hardware ---
# Forces MPI to stay local and prevents searching for expensive networking cards
export UCX_TLS=shm,self,cuda,tcp
export OMPI_MCA_btl=self,vader,tcp
export OMPI_MCA_pml=ob1
export OMPI_MCA_osc=pt2pt
export OMPI_MCA_coll=^hcoll
export OMPI_MCA_plm_rsh_agent=/bin/false

# --- Useful Aliases ---
alias pw.x="mpirun -np 1 pw.x"
alias check-qe="mpirun -np 1 pw.x -input /dev/null"
# ------------------------------------------------
```

> [!IMPORTANT]
> **Don't forget to reload:** Run `source ~/.zshrc` after saving the file.

---

## 🛠️ 3. Compilation & Global Installation

### Step A: Clean Workspace
```bash
git clone https://github.com/QEF/q-e.git
cd q-e
mkdir build && cd build
```

### Step B: Optimized Configuration
This command is the "Secret Sauce." It targets your specific Ryzen CPU and Turing GPU architecture.

```bash
cmake -DCMAKE_C_COMPILER=$NVCOMPILERS/bin/nvc \
      -DCMAKE_CXX_COMPILER=$NVCOMPILERS/bin/nvc++ \
      -DCMAKE_Fortran_COMPILER=$NVCOMPILERS/bin/nvfortran \
      -DCMAKE_Fortran_FLAGS="-tp=zen2 -O3" \
      -DCMAKE_BUILD_TYPE=Release \
      -DQE_ENABLE_CUDA=ON \
      -DQE_ENABLE_OPENACC=ON \
      -DQE_GPU_ARCH=sm_75 \
      -DNVFORTRAN_CUDA_CC=75 \
      -DQE_ENABLE_MPI=ON \
      -DQE_ENABLE_MPI_GPU_AWARE=ON \
      -DCMAKE_INSTALL_PREFIX=/usr/local \
      ..
```

### Step C: Build
```bash
make -j$(nproc)
sudo env "PATH=$PATH" "LD_LIBRARY_PATH=$LD_LIBRARY_PATH" make install
```

---

## ✅ 4. Verification ("Proof of Life")

To confirm your GPU is working, run the automatic check:
```bash
check-qe
```

**Look for these specific success indicators:**
- [x] `GPU acceleration is ACTIVE. 1 visible GPUs per MPI rank`
- [x] `GPU-aware MPI enabled`

In the timing summary at the very bottom, look for lines ending in **`GPU`**. If you see numbers there (ex: `0.08s GPU`), the math was handled by your NVIDIA card!

---

## 💡 Troubleshooting for Amateurs

| If you see... | It means... | Solution |
| :--- | :--- | :--- |
| **Terminal Hangs** | MPI is searching for Infiniband hardware. | Ensure the `OMPI_MCA` flags from Section 2 are in your `.zshrc`. |
| **pw.x: command not found** | `/usr/local/bin` isn't in your PATH. | Add `export PATH=/usr/local/bin:$PATH` to your config. |
| **No GPU message** | QE didn't get a "real" task to initialize. | Give it a real input file or use `check-qe`. |
| **Sudo errors** | Sudo is hiding your NVIDIA libraries. | Always use the `sudo env "PATH=$PATH" ...` format for installing. |

---

**Happy Simulating! 🚀**
