# TF-A (Trusted Firmware-A) Notebook

Personal notes and learnings about Trusted Firmware-A.

---

## System Registers
### MPIDR (Multiprocessor Affinity Register)

**Date**: 2025-10-20

MPIDR stands for **Multiprocessor Affinity Register** - it's an ARM architecture system register that uniquely identifies each CPU core in a multi-core system.

#### What is MPIDR?

**Purpose**: Provides a unique identifier for each CPU core, allowing software to identify which CPU it's running on and to target specific CPUs for operations like power management.

**Full Name**: `MPIDR_EL1` (Multiprocessor Affinity Register, Exception Level 1)

#### Structure

The MPIDR is organized hierarchically to represent the physical topology of the system:

```
MPIDR_EL1 (64-bit register)
┌────────┬─────────┬─────────┬─────────┬─────────┐
│ Aff3   │  Aff2   │  Aff1   │  Aff0   │  Other  │
│[39:32] │ [23:16] │ [15:8]  │ [7:0]   │  bits   │
└────────┴─────────┴─────────┴─────────┴─────────┘
   │         │         │         │
   │         │         │         └─ Core within cluster
   │         │         └─────────── Cluster within group
   │         └───────────────────── Cluster group
   └─────────────────────────────── System level
```

**Affinity Levels**:
- **Aff0** [7:0]: Core number within a cluster (e.g., core 0, 1, 2, 3)
- **Aff1** [15:8]: Cluster number (e.g., cluster 0 or 1 in big.LITTLE)
- **Aff2** [23:16]: Higher-level grouping (less commonly used)
- **Aff3** [39:32]: System-level affinity (multi-socket systems)

**Example MPIDR values**:
- `0x00000000` - Cluster 0, Core 0 (typically the primary/boot CPU)
- `0x00000001` - Cluster 0, Core 1
- `0x00000100` - Cluster 1, Core 0 (big.LITTLE: different cluster)
- `0x00000101` - Cluster 1, Core 1

#### Usage in TF-A

##### 1. CPU Identification

```c
// Get current CPU's MPIDR
u_register_t mpidr = read_mpidr_el1();

// Extract affinity levels
unsigned int aff0 = MPIDR_AFFLVL0_VAL(mpidr);  // Core
unsigned int aff1 = MPIDR_AFFLVL1_VAL(mpidr);  // Cluster
```

Location: `include/arch/aarch64/arch_helpers.h`

##### 2. PSCI CPU Targeting

When you want to power on a specific CPU, you specify its MPIDR:

```c
// PSCI CPU_ON call
int psci_cpu_on(u_register_t target_cpu,    // <- MPIDR of target CPU
                uintptr_t entrypoint,
                u_register_t context_id);
```

**Example from code**:
```c
// lib/psci/psci_on.c
int psci_cpu_on(u_register_t target_cpu,
                uintptr_t entrypoint,
                u_register_t context_id)
{
    // Validate the MPIDR
    rc = psci_validate_mpidr(target_cpu);
    if (rc != PSCI_E_SUCCESS)
        return PSCI_E_INVALID_PARAMS;

    // Convert MPIDR to CPU index
    int cpu_idx = plat_core_pos_by_mpidr(target_cpu);

    // Power on the CPU
    ...
}
```

##### 3. Platform Topology Mapping

Platforms implement `plat_core_pos_by_mpidr()` to convert MPIDR to a linear CPU index:

```c
// plat/arm/board/fvp/fvp_topology.c
int plat_core_pos_by_mpidr(u_register_t mpidr)
{
    unsigned int cluster_id, cpu_id;

    mpidr &= MPIDR_AFFINITY_MASK;  // Mask irrelevant bits

    if (mpidr & ~(MPIDR_CLUSTER_MASK | MPIDR_CPU_MASK))
        return -1;  // Invalid MPIDR

    cluster_id = (mpidr >> MPIDR_AFF1_SHIFT) & MPIDR_AFFLVL_MASK;
    cpu_id = (mpidr >> MPIDR_AFF0_SHIFT) & MPIDR_AFFLVL_MASK;

    // Example: 4 cores per cluster, 2 clusters
    if (cluster_id >= 2 || cpu_id >= 4)
        return -1;

    return (cluster_id * 4) + cpu_id;
    // Results: 0-3 (cluster 0), 4-7 (cluster 1)
}
```

##### 4. PSCI Data Structures

TF-A uses MPIDR to track CPU state:

```c
// lib/psci/psci_setup.c
typedef struct psci_cpu_pd_node {
    u_register_t mpidr;           // CPU's MPIDR
    unsigned int parent_node;
    // ... state information
} psci_cpu_pd_node_t;

// Array indexed by linear CPU ID, stores MPIDR
psci_cpu_pd_node_t psci_cpu_pd_nodes[PLATFORM_CORE_COUNT];
```

##### 5. Key Code References

**Reading MPIDR**:
- `include/arch/aarch64/arch_helpers.h:read_mpidr_el1()`
- Assembly: `MRS X0, MPIDR_EL1`

**MPIDR validation**:
- `lib/psci/psci_common.c:psci_validate_mpidr()` - checks if MPIDR is valid for platform

**MPIDR to index conversion**:
- Platform-specific: `plat/<vendor>/<platform>_topology.c:plat_core_pos_by_mpidr()`

**Usage in PSCI**:
- `lib/psci/psci_on.c:psci_cpu_on()` - line 159
- `lib/psci/psci_setup.c` - CPU initialization with MPIDR

#### Important Notes

1. **MPIDR is read-only** - assigned by hardware, cannot be changed by software
2. **Not necessarily sequential** - MPIDR values may have gaps (e.g., 0x0, 0x1, 0x100, 0x101)
3. **Platform-specific** - Different SoCs use different affinity level schemes
4. **Invalid MPIDR marker**: TF-A uses `PSCI_INVALID_MPIDR` (typically 0xFFFFFFFF) for uninitialized/invalid values

#### Example: big.LITTLE System

On a system with 4 LITTLE cores (cluster 0) and 4 big cores (cluster 1):

```
LITTLE cores:
  Core 0: MPIDR = 0x00000000  → Linear index 0
  Core 1: MPIDR = 0x00000001  → Linear index 1
  Core 2: MPIDR = 0x00000002  → Linear index 2
  Core 3: MPIDR = 0x00000003  → Linear index 3

big cores:
  Core 0: MPIDR = 0x00000100  → Linear index 4
  Core 1: MPIDR = 0x00000101  → Linear index 5
  Core 2: MPIDR = 0x00000102  → Linear index 6
  Core 3: MPIDR = 0x00000103  → Linear index 7
```

The MPIDR allows TF-A to:
- Identify which core is running (primary vs secondary)
- Target specific cores for power operations
- Map physical CPU topology to software data structures
- Route interrupts to specific cores

**Key takeaway**: MPIDR is the fundamental CPU identifier in ARM systems, essential for multi-core power management and topology description.

---

### SCTLR_EL3 (System Control Register for EL3)

**Date**: 2025-10-20

SCTLR_EL3 is the **System Control Register** for Exception Level 3 - it controls various aspects of the processor's behavior including memory system configuration, alignment checking, and caching.

#### What is SCTLR_EL3?

**Purpose**: Provides system-level control of the processor at EL3, including:
- MMU enable/disable
- Cache enable/disable
- Alignment checking
- Endianness
- Instruction/data access controls

**Full Name**: `SCTLR_EL3` (System Control Register, Exception Level 3)

#### Register Structure

SCTLR_EL3 is a 64-bit register with many control bits. Key bits defined in `include/arch/aarch64/arch.h:559-590`:

```c
#define SCTLR_M_BIT      (ULL(1) << 0)   // MMU enable
#define SCTLR_A_BIT      (ULL(1) << 1)   // Alignment check enable
#define SCTLR_C_BIT      (ULL(1) << 2)   // Data cache enable
#define SCTLR_SA_BIT     (ULL(1) << 3)   // Stack Alignment check
#define SCTLR_I_BIT      (ULL(1) << 12)  // Instruction cache enable
#define SCTLR_WXN_BIT    (ULL(1) << 19)  // Write permission implies XN
#define SCTLR_EE_BIT     (ULL(1) << 25)  // Exception Endianness
// ... and many more
```

#### SCTLR_EL3.M (Bit 0) - MMU Enable

**Purpose**: Controls whether the Memory Management Unit (MMU) is enabled at EL3.

**Values**:
- **0**: MMU disabled
  - All memory accesses are treated as physical addresses
  - No address translation
  - No memory permissions enforced
  - Used during early boot before page tables are set up
- **1**: MMU enabled
  - Virtual to physical address translation active
  - Page table permissions enforced
  - Memory attributes (cacheable, device, etc.) applied
  - Required for secure operation

**Critical for**:
- Memory protection and isolation
- Enabling secure/non-secure memory separation
- Allowing use of virtual addresses
- Enforcing execute-never (XN) permissions

**Usage in TF-A**:

```c
// Check if MMU is enabled
if (read_sctlr_el3() & SCTLR_M_BIT) {
    // MMU is enabled
}

// Enable MMU (typically done via enable_mmu_el3())
write_sctlr_el3(read_sctlr_el3() | SCTLR_M_BIT);
```

**Example from code** (`plat/nvidia/tegra/lib/debug/profiler.c:100`):
```c
// Only flush caches if MMU is enabled
if ((read_sctlr_el3() & SCTLR_M_BIT) != U(0)) {
    flush_dcache_range(/* ... */);
}
```

**When is MMU disabled?**:
- Very early boot (BL1/BL2/BL31 entrypoint, before `init_xlat_tables()`)
- Before warm boot resume (CPU power on sequence)
- Temporarily for certain low-level operations

**When MMU is enabled**:
- After `enable_mmu_el3()` is called
- During normal BL31 runtime operation
- Throughout SMC handling and service execution

#### SCTLR_EL3.C (Bit 2) - Data Cache Enable

**Purpose**: Controls whether data caching is enabled at EL3.

**Values**:
- **0**: Data cache disabled
  - All data accesses go directly to memory (uncached)
  - Slower but guarantees memory coherency
  - Used when working with device memory or shared data structures
- **1**: Data cache enabled
  - Data reads/writes can be cached in L1/L2/L3 data caches
  - Significantly improves performance
  - Requires cache maintenance operations for coherency

**Critical for**:
- Performance (cached memory access is 10-100x faster)
- Coherency with other CPUs and DMA devices
- Proper handling of device registers (must be uncached)

**Usage in TF-A**:

```c
// Check if data cache is enabled
if (read_sctlr_el3() & SCTLR_C_BIT) {
    // Data cache is enabled
    clean_dcache_range(addr, size);  // Ensure coherency
}

// Disable data cache (for special operations)
write_sctlr_el3(read_sctlr_el3() & ~SCTLR_C_BIT);
// Do uncached operations...
// Re-enable data cache
write_sctlr_el3(read_sctlr_el3() | SCTLR_C_BIT);
```

**Example from code** (`drivers/renesas/common/auth/auth_mod.c:115-126`):
```c
// Disable cache before cryptographic operation
write_sctlr_el3(read_sctlr_el3() & ~SCTLR_C_BIT);

// Perform security-sensitive operation
// (ensures data goes to memory, not cached)
verify_signature(/* ... */);

// Re-enable cache
write_sctlr_el3(read_sctlr_el3() | SCTLR_C_BIT);
```

**When is data cache disabled?**:
- Very early boot before cache initialization
- During certain secure operations (to prevent cache-based attacks)
- When accessing hardware that is not cache-coherent
- During CPU power down sequences

**When is data cache enabled?**:
- After early initialization (`el3_arch_init_common` macro)
- During normal BL31 runtime
- Throughout most firmware execution

#### Typical SCTLR_EL3 Initialization

From `include/arch/aarch64/el3_common_macros.S:19-46`:

```assembly
el3_arch_init_common:
    /* Enable I-cache, Alignment check, Stack Alignment check */
    mov_imm x1, (SCTLR_I_BIT | SCTLR_A_BIT | SCTLR_SA_BIT)
    mrs     x0, sctlr_el3       // Read current value
    orr     x0, x0, x1           // Set bits
    msr     sctlr_el3, x0        // Write back
    isb                          // Instruction barrier
```

**Note**: MMU (M bit) and data cache (C bit) are **NOT** enabled here. They are enabled later via:
- `enable_mmu_el3()` function (sets both M and C bits together)
- Located in `lib/xlat_tables_v2/aarch64/enable_mmu.S`

#### Important SCTLR_EL3 Bits Summary

| Bit | Name | Purpose | When Set |
|-----|------|---------|----------|
| 0 | M | MMU enable | After page tables configured |
| 1 | A | Alignment check | Early init |
| 2 | C | Data cache enable | After caches initialized |
| 3 | SA | Stack Pointer alignment check | Early init |
| 12 | I | Instruction cache enable | Early init |
| 19 | WXN | Write implies XN (no-execute) | Security hardening |
| 25 | EE | Exception endianness | Platform dependent |

#### Relationship Between M and C Bits

**Critical requirement**: You cannot enable caching (C=1) without enabling the MMU (M=1).

From ARM Architecture Reference Manual:
> "When SCTLR_ELx.M is 0, the PE behaves as if the value of SCTLR_ELx.C is 0 for all purposes other than returning the value of a direct read of the bit."

This means:
- Setting C=1 with M=0 has no effect (no caching occurs)
- Both M and C are typically set together via `enable_mmu_el3()`
- Both are cleared together during MMU disable

**Typical sequence**:
```c
// 1. Setup page tables
init_xlat_tables();

// 2. Enable MMU + caches (sets M=1, C=1, I=1)
enable_mmu_el3(0);
```

#### Code References

**SCTLR_EL3 bit definitions**:
- `include/arch/aarch64/arch.h:559-590`

**Reading/Writing SCTLR_EL3**:
- `include/arch/aarch64/arch_helpers.h:read_sctlr_el3()`, `write_sctlr_el3()`
- Assembly: `MRS X0, SCTLR_EL3` / `MSR SCTLR_EL3, X0`

**Initialization**:
- `include/arch/aarch64/el3_common_macros.S:19` - `el3_arch_init_common` macro
- Used by: `bl1/aarch64/bl1_entrypoint.S`, `bl31/aarch64/bl31_entrypoint.S`

**MMU enable**:
- `lib/xlat_tables_v2/aarch64/enable_mmu.S:enable_mmu_el3()`
- Called from: `bl31/bl31_main.c:bl31_main()` → `bl31_plat_arch_setup()`

**Usage examples**:
- Cache management: `drivers/renesas/common/auth/auth_mod.c:115`
- MMU check: `plat/nvidia/tegra/lib/debug/profiler.c:100`
- DRTM (Dynamic Root of Trust): `services/std_svc/drtm/drtm_main.c:557`

#### Important Notes

1. **MMU and caches are interdependent**: C bit only works when M=1
2. **Performance impact**: Disabling C has dramatic performance penalty (10-100x slower)
3. **Security considerations**:
   - M=0 means no memory protection (security risk)
   - Some operations temporarily disable C to prevent cache timing attacks
4. **Cache coherency**: When C=1, must use cache maintenance operations
   - `clean_dcache_range()` - write cache to memory
   - `inv_dcache_range()` - invalidate cache
   - `flush_dcache_range()` - clean + invalidate

#### Key Takeaways

- **SCTLR_EL3.M (bit 0)**: Enables the MMU - required for memory protection and virtual addressing
- **SCTLR_EL3.C (bit 2)**: Enables data caching - critical for performance but requires MMU enabled (M=1)
- Both are typically enabled together after page table setup
- Both can be temporarily disabled for special low-level operations
- Understanding these bits is essential for debugging boot issues, cache coherency problems, and performance analysis

---

### Why MPIDR_EL1 vs SCTLR_EL3? (Register Naming Convention)

**Date**: 2025-10-20

This is a great question that highlights an important aspect of ARM's system register architecture!

#### The Answer: Different Register Types

The suffix (\_EL1, \_EL2, \_EL3) indicates **which Exception Level the register belongs to** and **how it behaves**.

#### MPIDR_EL1 - Read-Only Identification Register

**MPIDR_EL1** returns the **same value** regardless of which Exception Level you're running at.

**Key characteristics**:
- **Read-only** - cannot be modified by software
- **Same value everywhere** - reading from EL3, EL2, or EL1 returns **identical CPU identification**
- **Hardware-assigned** - the physical CPU ID doesn't change
- **Single instance** - there is **no MPIDR_EL2 or MPIDR_EL3**

**Code definition** (`include/arch/aarch64/arch_helpers.h:436`):
```c
DEFINE_SYSREG_READ_FUNC(mpidr_el1)  // Only read function, no write
```

**Access from any Exception Level**:
```assembly
; From EL3 (BL31 running at EL3)
MRS X0, MPIDR_EL1    ; Returns 0x00000000 (cluster 0, core 0)

; From EL2 (Hypervisor)
MRS X0, MPIDR_EL1    ; Returns 0x00000000 (same value!)

; From EL1 (Linux kernel)
MRS X0, MPIDR_EL1    ; Returns 0x00000000 (same value!)
```

**Why it's named _EL1**:
- Historical reasons - originally accessible from EL1
- ARM architecture allows higher ELs to access it too
- The naming just indicates the "base" exception level
- Think of it as "available at EL1 and above"

**Similar read-only registers with _EL1 suffix**:
- `MIDR_EL1` - Main ID Register (CPU type identification)
- `ID_AA64PFR0_EL1` - Processor Feature Register (what features CPU has)
- `CLIDR_EL1` - Cache Level ID Register

These all return the same value regardless of current EL because they describe **fixed hardware properties**.

#### SCTLR_EL1/EL2/EL3 - Separate Control Registers

**SCTLR** exists as **three completely independent registers**, one for each Exception Level:
- **SCTLR_EL1** - System Control Register for EL1
- **SCTLR_EL2** - System Control Register for EL2
- **SCTLR_EL3** - System Control Register for EL3

**Key characteristics**:
- **Read-write** - can be modified by software at that privilege level
- **Different values** - each EL has its own completely independent SCTLR instance
- **Different purposes** - each EL needs different MMU/cache settings
- **Privilege isolation** - EL1 cannot access SCTLR_EL3 (would trap to EL3)

**Code definitions** (`include/arch/aarch64/arch_helpers.h:447-449`):
```c
DEFINE_SYSREG_RW_FUNCS(sctlr_el1)  // Read and write functions
DEFINE_SYSREG_RW_FUNCS(sctlr_el2)  // Separate register!
DEFINE_SYSREG_RW_FUNCS(sctlr_el3)  // Separate register!
```

**Three Separate Registers Example**:
```c
// In BL31 running at EL3
uint64_t sctlr_el3 = read_sctlr_el3();
// Returns: 0x30C5183D (M=1, C=1, I=1 - MMU and caches enabled)

// BL31 can also read/write SCTLR for lower ELs
uint64_t sctlr_el1 = read_sctlr_el1();
// Returns: 0x30C50838 (different value - EL1's MMU might be disabled)

// These are COMPLETELY DIFFERENT registers with different values!
```

**Why separate registers are needed**:

Each Exception Level needs independent control because:

1. **EL3** (TF-A): May enable MMU/caches immediately after page table setup
2. **EL2** (Hypervisor): May have different MMU configuration
3. **EL1** (Linux kernel): Starts with MMU disabled, enables it during boot

**Example - Different MMU states simultaneously**:
```
Time: During Linux boot

EL3 (BL31):     SCTLR_EL3.M = 1  (MMU enabled, handling SMC calls)
EL2 (KVM):      SCTLR_EL2.M = 1  (MMU enabled, hypervisor running)
EL1 (Linux):    SCTLR_EL1.M = 0  (MMU disabled, early boot code)
```

All three can have **different SCTLR values at the same time**!

#### Register Access Rules

**Privilege Model**:
- Higher ELs can access lower EL registers
- Lower ELs **cannot** access higher EL registers

```c
// Running at EL3 (BL31):
read_sctlr_el3();  ✓ Allowed - current EL
read_sctlr_el2();  ✓ Allowed - lower EL
read_sctlr_el1();  ✓ Allowed - lower EL
read_mpidr_el1();  ✓ Allowed - identification register

// Running at EL1 (Linux kernel):
read_sctlr_el1();  ✓ Allowed - current EL
read_mpidr_el1();  ✓ Allowed - identification register
read_sctlr_el3();  ✗ TRAP! - would cause exception to EL3
read_sctlr_el2();  ✗ TRAP! - would cause exception to EL2
```

#### Summary Table

| Register | Type | EL1 Access | EL2 Access | EL3 Access | Values |
|----------|------|-----------|-----------|-----------|--------|
| **MPIDR_EL1** | Read-only ID | Read | Read | Read | Same value everywhere |
| **SCTLR_EL1** | Read-write control | Read/Write | Read/Write | Read/Write | EL1's value |
| **SCTLR_EL2** | Read-write control | TRAP | Read/Write | Read/Write | EL2's value |
| **SCTLR_EL3** | Read-write control | TRAP | TRAP | Read/Write | EL3's value |

#### Practical Implications in TF-A

**When BL31 prepares to jump to Normal World**:

```c
// lib/el3_runtime/context_mgmt.c
static void cm_init_context_common(cpu_context_t *ctx, const entry_point_info_t *ep)
{
    // Setup SCTLR_EL1 for the Normal World OS
    sctlr_el1 = SCTLR_EL1_RES1;  // Reserved bits set to 1
    sctlr_el1 &= ~(SCTLR_M_BIT);  // Disable MMU for early boot
    sctlr_el1 &= ~(SCTLR_C_BIT);  // Disable data cache
    write_ctx_reg(get_el1_sysregs_ctx(ctx), CTX_SCTLR_EL1, sctlr_el1);

    // But SCTLR_EL3 remains unchanged!
    // BL31 keeps its MMU/cache enabled while preparing EL1's state
}
```

**Key takeaway**:
- **MPIDR_EL1** = Same value everywhere (it's an ID register)
- **SCTLR_EL1/EL2/EL3** = Three separate registers with independent values (control registers)
- The suffix indicates which Exception Level the register belongs to, not just which can access it

---
