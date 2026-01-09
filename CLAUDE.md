# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Bao is a lightweight, open-source embedded hypervisor providing strong isolation and real-time guarantees through static partitioning. It is written primarily in C with architecture-specific assembly, targeting ARMv8 (AArch64/AArch32) and RISC-V (RV64/RV32) platforms.

**Key design principles**: Minimal Trusted Computing Base (TCB), no external dependencies, static resource partitioning at VM instantiation, 1:1 virtual-to-physical CPU mapping (no scheduler), pass-through I/O only, and direct virtual-to-physical interrupt mapping.

## Build Commands

```bash
# Build for a specific platform and configuration (both required)
make PLATFORM=<platform> CONFIG=<config>

# Examples:
make PLATFORM=qemu-aarch64-virt CONFIG=null
make PLATFORM=fvp-a CONFIG=null GIC_VERSION=GICV3
make PLATFORM=rpi4 CONFIG=linux_demo

# Clean build artifacts
make clean

# Run CI checks
make ci                    # Run all CI checks
make format-check          # Check code formatting
make license-check         # Check license headers

# Optional build parameters
DEBUG=y                    # Enable debug symbols
CROSS_COMPILE=clang        # Use Clang instead of GCC
OPTIMIZATIONS=0-3          # Optimization level (default: 2)
```

**Output artifacts** are placed in `bin/<PLATFORM>/<CONFIG>/`:
- `bao.elf` - ELF binary
- `bao.bin` - Raw binary
- `bao.asm` - Disassembly
- `bao.elf.txt` - Readelf output

## Architecture and Code Organization

The codebase separates architecture-independent core from architecture/platform-specific code:

### Core (`src/core/`)
Architecture-independent hypervisor functionality:
- `vm.c`, `vmm.c`, `cpu.c` - VM and CPU management
- `mem.c` - Memory management
- `interrupts.c` - Interrupt handling
- `console.c`, `ipc.c`, `shmem.c`, `remio.c` - Console, IPC, shared memory, remote I/O
- `hypercall.c` - Hypercall interface
- `mmu/` or `mpu/` - Memory protection (select one per architecture)

### Architecture (`src/arch/`)
- `armv8-a/` - ARMv8 Application profile (AArch64, AArch32)
- `armv8-r/` - ARMv8 Real-time profile (AArch64, AArch32)
- `riscv/` - RISC-V RV64/RV32

Each architecture directory contains boot code, exception handling, interrupt controller support (GIC for ARM, AIA/PLIC for RISC-V), page table management, and virtualization extensions.

### Platform (`src/platform/`)
Platform-specific code and hardware drivers. Each platform has:
- `platform.mk` - Platform configuration (CPU, drivers, memory map)
- `<platform>_desc.c` - Platform description (memory regions, CPU count, etc.)

Supported platforms include QEMU variants, Arm FVPs, and hardware boards (rpi4, zcu102, zcu104, ultra96, tx2, hikey960, imx8qm, imx8mp-verdin, mps3-an536).

### Configuration (`configs/`)
VM configurations are C files defining the `config` struct with VM images, memory regions, CPU affinity, device assignments, and IPC configurations. Example structure:

```c
struct config config = {
    .shmemlist_size = N,
    .shmemlist = (struct shmem[]) { ... },
    .vmlist_size = N,
    .vmlist = (struct vm_config[]) {
        {
            .image = VM_IMAGE_BUILTIN(name, base_addr),
            .entry = entry_point,
            .cpu_affinity = cpu_mask,
            .platform = {
                .cpu_num = N,
                .region_num = N,
                .regions = (struct vm_mem_region[]) { ... },
                .dev_num = N,
                .devs = (struct vm_dev_region[]) { ... },
                .ipc_num = N,
                .ipcs = (struct ipc[]) { ... },
                .arch = { ... },
            },
        },
    },
};
```

## Key Technical Concepts

- **Static Partitioning**: Resources assigned at compile-time, no runtime scheduling
- **Memory Coloring**: Cache allocation mechanism for real-time guarantees
- **Two-Stage Translation**: Hardware-assisted address translation for memory isolation
- **No Privileged VM**: Hypervisor is the only privileged software component

## Adding Platform or Configuration

**New platform**: Create `src/platform/<platform>/` with `platform.mk`, `<platform>_desc.c`, and `objects.mk`.

**New configuration**: Create `configs/<config>/config.c` and build with `make PLATFORM=<plat> CONFIG=<config>`.

## Documentation

- Main documentation: https://bao-project.readthedocs.io/
- Demo configurations: https://github.com/bao-project/bao-demos
- Project website: http://www.bao-project.org/

## Development Notes

- Use QEMU platforms (`qemu-aarch64-virt`, `qemu-riscv64-virt`, `qemu-riscv32-virt`) for rapid testing
- CI runs on GitHub Actions across multiple platforms (see `.github/workflows/`)
- Code formatting and license headers are enforced via CI
- License: Apache-2.0
