# Trusted Firmware-A (TF-A) - Comprehensive Codebase Analysis

**IMPORTANT: Always check ~/.claude/commands directory for custom slash commands before attempting to execute them as bash commands. Slash commands (starting with /) are Claude Code custom commands, not bash commands.**

## 1. High-Level Architecture Survey

### Top-Level Directory Structure

```
trusted-firmware-a/
├── bl1/                    # Boot Loader stage 1 (ROM firmware)
├── bl2/                    # Boot Loader stage 2 (Trusted boot firmware)
├── bl2u/                   # Boot Loader 2 Update (firmware update)
├── bl31/                   # Boot Loader stage 31 (EL3 Runtime Monitor - AArch64)
├── bl32/                   # Boot Loader stage 32 (Secure-EL1 Payload/TEE)
├── common/                 # Shared bootloader utilities
├── lib/                    # Core libraries (PSCI, xlat tables, CPU support, etc.)
├── plat/                   # Platform-specific implementations (24 vendors)
├── drivers/                # Hardware drivers (34+ categories)
├── services/               # Runtime services (SMC handlers)
├── include/                # Public header files
├── tools/                  # Build and packaging tools
├── fdts/                   # Device tree sources
├── docs/                   # Sphinx-based documentation
├── make_helpers/           # Build system infrastructure
└── contrib/                # Contributed utilities
```

### Main Entry Points

**Build System:**
- `Makefile` - Root makefile orchestrating the entire build
- `make_helpers/defaults.mk` - Default build configuration
- `make_helpers/toolchain.mk` - Compiler and linker setup
- Each BL stage has its own makefile: `bl1/bl1.mk`, `bl2/bl2.mk`, `bl31/bl31.mk`, etc.

**Code Entry Points:**
- `bl1/aarch64/bl1_entrypoint.S` - First code executed (Boot ROM)
- `bl2/aarch64/bl2_entrypoint.S` - Trusted boot firmware entry
- `bl31/aarch64/bl31_entrypoint.S` - EL3 runtime monitor entry
- `bl31/bl31_main.c:bl31_main()` - BL31 C entry point

**Runtime Entry Points:**
- `bl31/aarch64/runtime_exceptions.S` - Exception vectors (SMC, IRQ, FIQ, etc.)
- `common/runtime_svc.c` - Runtime service dispatcher
- Services registered via `DECLARE_RT_SVC()` macro

### Configuration Files

- `pyproject.toml` - Python dependencies (Poetry-based)
- `plat/<vendor>/platform.mk` - Platform-specific build configuration
- `plat/<vendor>/platform_def.h` - Platform memory maps and limits
- `include/plat/common/platform.h` - Platform API contracts
- Device trees in `fdts/` - Hardware descriptions for platforms

### Build System Architecture

**390+ Makefile fragments** across the codebase with modular organization:
- Main `Makefile` includes sub-makefiles conditionally based on build options
- Platform selection via `PLAT=<platform>` triggers inclusion of `plat/<vendor>/platform.mk`
- Feature-based inclusion: SPD (Secure Payload Dispatcher), RME, PSCI, etc.
- Each bootloader stage (BL1, BL2, BL31, etc.) has dedicated makefiles
- Linker scripts are templated (`.ld.S`) and preprocessed

**Key Build Dependencies:**
- Cross-compiler toolchain (GCC, Clang, or ARM Compiler 6)
- Python 3.8+ with Poetry for tooling
- Optional: OpenSSL for certificate generation
- Optional: Device Tree Compiler (dtc) for FDT processing

## 2. Component Mapping

### Major Functional Modules

#### Boot Stages (Sequential Execution)
```
BL1 (ROM) → BL2 (Trusted Boot) → BL31 (Runtime Monitor) → BL32 (TEE) → BL33 (OS Loader)
                                       ↓
                                  Exception/SMC Handling
                                       ↓
                              Runtime Services (PSCI, SDEI, etc.)
```

**BL1** (`bl1/`):
- ARM Trusted Board Boot authentication
- Platform initialization
- BL2 loading and verification
- Key files: `bl1_main.c`, `aarch64/bl1_entrypoint.S`

**BL2** (`bl2/`):
- Hardware initialization
- Load BL31, BL32, BL33 images
- Firmware Image Package (FIP) parsing
- Secure Partition loading (if enabled)
- Key files: `bl2_main.c`, `bl2_image_load_v2.c`

**BL31** (`bl31/`):
- EL3 exception level runtime firmware
- SMC (Secure Monitor Call) handling
- Interrupt management (GIC initialization)
- World switching (Secure ↔ Normal)
- Key files:
  - `bl31_main.c` - Initialization and setup
  - `interrupt_mgmt.c` - Interrupt routing
  - `bl31_traps.c` - Trap handling (system register access)
  - `aarch64/runtime_exceptions.S` - Exception vectors

**BL32** (`bl32/`):
- Secure Payload options:
  - `optee/` - OP-TEE OS integration
  - `sp_min/` - Minimal AArch32 secure payload
  - `tsp/` - Test Secure Payload (for validation)

### Core Libraries

**Power State Coordination Interface** (`lib/psci/`):
- CPU power on/off, suspend/resume
- System power management
- Platform-specific hooks: `plat/<vendor>/*_pm.c`
- Files: `psci_main.c`, `psci_on.c`, `psci_off.c`, `psci_suspend.c`

**Translation Tables** (`lib/xlat_tables_v2/`):
- MMU configuration and page table management
- Dynamic memory mapping support
- Memory attribute management (cacheable, device, secure/non-secure)
- Files: `xlat_tables_core.c`, `xlat_tables_context.c`

**CPU Support** (`lib/cpus/`):
- CPU-specific initialization code
- Errata workarounds for ARM cores
- Per-CPU power management hooks
- Organization: `aarch64/<cpu_name>.S`, `aarch32/<cpu_name>.S`

**EL3 Runtime** (`lib/el3_runtime/`):
- CPU context save/restore
- Exception level transitions
- Architecture state management
- Files: `context_mgmt.c`, `cpu_data_array.S`

**Extensions** (`lib/extensions/`):
- ARM architecture feature support:
  - `pauth/` - Pointer Authentication
  - `sve/` - Scalable Vector Extension
  - `amu/` - Activity Monitor Unit
  - `mpam/` - Memory Partitioning and Monitoring
  - `sme/` - Scalable Matrix Extension

**Firmware Configuration** (`lib/fconf/`):
- Dynamic firmware configuration framework
- Device tree based configuration
- Files: `fconf.c`, `fconf_dyn_cfg_getter.c`

**Granule Protection Tables** (`lib/gpt_rme/`):
- Realm Management Extension support
- Memory isolation for confidential computing
- Files: `gpt_rme.c`

### Runtime Services Architecture

Services are registered using `DECLARE_RT_SVC()` macro and dispatched via SMC calls:

**Standard Services** (`services/std_svc/`):
- PSCI - Power management
- SDEI - Software Delegated Exception Interface
- TRNG - True Random Number Generator
- SPMD - Secure Partition Manager Dispatcher
- SPMC - Secure Partition Manager Core (EL3 or S-EL2)
- RMMD - Realm Management Monitor Dispatcher
- DRTM - Dynamic Root of Trust for Measurement

**Architecture Services** (`services/arm_arch_svc/`):
- ARM-specific SMC handlers
- Errata ABI services
- LFA (LLVM Frame Address) services

**Secure Payload Dispatchers** (`services/spd/`):
- `opteed/` - OP-TEE Dispatcher
- `trusty/` - Trusty OS Dispatcher
- `tlkd/` - Trusted Little Kernel Dispatcher
- `tspd/` - Test Secure Payload Dispatcher

### Driver Organization

**34+ driver categories** in `drivers/`:

**ARM-Specific** (`drivers/arm/`):
- GIC (Generic Interrupt Controller) - v2, v3, v600, v700
- TZC (TrustZone Address Space Controller)
- CCI/CCN (Cache Coherent Interconnect/Network)
- DSU (DynamIQ Shared Unit)
- RSS (Runtime Security Subsystem)

**Storage** (`drivers/`):
- `io/` - Generic I/O abstraction layer
- `mmc/` - MultiMediaCard/SD/eMMC
- `mtd/` - Memory Technology Devices (NAND, NOR flash)
- `ufs/` - Universal Flash Storage
- `usb/` - USB device

**Security** (`drivers/auth/`):
- Certificate chain validation
- Cryptographic operations (mbedTLS integration)
- Image authentication

**Console/UART** (`drivers/console/`):
- Multi-console support
- Platform-agnostic console framework
- UART drivers (PL011, 16550, etc.)

### Platform Structure

**24 vendor directories** in `plat/`:
- allwinner, amd, amlogic, arm, aspeed, brcm, hisilicon, imx, intel, marvell, mediatek, nuvoton, nvidia, nxp, qemu, qti, renesas, rockchip, rpi, socionext, st, ti, xilinx

**Common Platform Code** (`plat/common/`):
- Generic platform initialization
- Default implementations
- Shared utilities

**ARM Reference Platforms** (`plat/arm/`):
- `board/` - Development boards (FVP, Juno, TC, RD)
- `css/` - Coherent Subsystem platforms
- `soc/` - System-on-Chip specific code
- `common/` - ARM platform helpers

Each platform must implement:
- `platform.mk` - Build configuration
- `platform_def.h` - Memory maps, core counts, limits
- `<plat>_bl<N>_setup.c` - Stage-specific setup
- `<plat>_topology.c` - CPU topology description
- `<plat>_pm.c` - PSCI power management hooks

### Tools and Utilities

**Binary Tools** (C-based):
- `tools/fiptool/` - Firmware Image Package creation
- `tools/cert_create/` - Certificate generation for Trusted Boot
- `tools/encrypt_fw/` - Firmware encryption

**Python Tools** (Poetry-managed):
- `tools/memory/` - Memory map analysis and visualization
- `tools/sptool/` - Secure Partition packaging
- `tools/cot_dt2c/` - Chain of Trust device tree to C converter
- `contrib/libtl/tlc/` - Transfer List Compiler

### Test Infrastructure

- `bl32/tsp/` - Test Secure Payload for BL31/BL32 interaction testing
- External: TF-A Tests repository (not in this tree)
- CI integration with Jenkins
- Coverity Scan for static analysis

### Documentation Structure

**Sphinx-based docs** in `docs/`:
- `getting_started/` - Build guides, prerequisites
- `design/` - Architecture and design documents
- `components/` - Feature-specific docs (PSCI, SDEI, RME)
- `plat/` - Platform porting guides
- `process/` - Contributing, coding style
- `design_documents/` - Detailed technical specs

## 3. Technology Stack Identification

### Programming Languages

**Primary Languages:**
- **C** (C99 standard): ~1,887 source files
  - Core implementation language
  - Strict MISRA-C compliance in critical paths
- **Assembly** (GNU AS syntax): ~398 files
  - Architecture-specific code (`.S` files)
  - Exception vectors, context switching, low-level initialization
  - Used in: `bl*/aarch64/*.S`, `bl*/aarch32/*.S`, `lib/aarch64/*.S`
- **Python** (3.8+): Build and analysis tools
- **Make**: Build system (GNU Make 3.81+)
- **Device Tree Source**: Hardware descriptions

**Language Versions:**
- C: C99 with GNU extensions
- Python: ^3.8 (specified in pyproject.toml)
- Assembly: ARMv8-A, ARMv7-A instruction sets

### Frameworks and Libraries

**Internal Libraries:**
- Custom libc implementation (`lib/libc/`) - subset of standard C library
- xlat_tables_v2 - Memory translation framework
- PSCI Library - Power management framework
- Compiler runtime (`lib/compiler-rt/`) - from LLVM project

**External Dependencies:**

**Python Packages (via Poetry):**
```toml
[tool.poetry.dependencies]
python = "^3.8"
cot-dt2c = {path = "tools/cot_dt2c", develop = true}
memory = {path = "tools/memory", develop = true}
tlc = {path = "contrib/libtl/tlc", develop = true}

[tool.poetry.group.docs.dependencies]
sphinx = "^5.3.0"
myst-parser = "^0.18.1"
sphinxcontrib-plantuml = "^0.24.1"
sphinx-rtd-theme = "^1.1.1"

[tool.poetry.group.ci.dependencies]
click = "^8.1.3"
fdt = "^0.3.0"
```

**Third-Party C Libraries:**
- libfdt (Device Tree manipulation) - `lib/libfdt/`
- zlib (compression) - `lib/zlib/`
- mbedTLS (cryptography) - integrated for Trusted Boot
- libc subset - `lib/libc/` (minimal standard C library)

### Build Tools and Dependency Managers

**Build System:**
- GNU Make 3.81+ (primary build orchestrator)
- Cross-compilation toolchains:
  - GCC (aarch64-none-elf-, arm-none-eabi-)
  - Clang/LLVM 14.0.0+
  - ARM Compiler 6
- GNU binutils (linker, assembler, objcopy)

**Dependency Management:**
- Poetry (Python package management)
- Git submodules (not actively used; dependencies are vendored)

**Additional Tools:**
- Device Tree Compiler (dtc) - for FDT processing
- OpenSSL - certificate generation
- checkpatch.pl (Linux kernel style checker)
- cscope - code navigation

### Database Technologies

**Not Applicable** - This is firmware; no database systems.

### Testing Frameworks

**Built-in Test Components:**
- Test Secure Payload (TSP) - `bl32/tsp/`
- Platform-specific test configurations

**External Testing:**
- TF-A Tests (separate repository)
- QEMU and ARM Fast Models for emulation
- Foundation Platform (FVP) for validation

**CI/CD:**
- Jenkins for continuous integration
- Gerrit for code review
- Coverity Scan for static analysis

### Deployment Tools

**Firmware Packaging:**
- fiptool - creates Firmware Image Package (FIP)
- sptool - packages Secure Partitions
- cert_create - generates certificate chains

**Target Deployment:**
- Platform-specific flash tools
- JTAG/SWD debuggers
- Network boot (TFTP) for development

## 4. Design Pattern Analysis

### Architectural Patterns

**Multi-Stage Boot Architecture:**
- Chain-of-trust boot pattern
- Each stage authenticates the next (Trusted Board Boot)
- Stage progression: BL1 (ROM) → BL2 (RAM) → BL31 (Runtime) → BL32/BL33
- Separation of concerns: boot-time vs runtime firmware

**Layered Architecture:**
```
┌─────────────────────────────────────────┐
│  Platform-Specific (plat/<vendor>/)     │
├─────────────────────────────────────────┤
│  Runtime Services (services/)           │
├─────────────────────────────────────────┤
│  Libraries (lib/psci, lib/xlat_tables)  │
├─────────────────────────────────────────┤
│  Drivers (drivers/)                     │
├─────────────────────────────────────────┤
│  Core/Common (common/, include/)        │
└─────────────────────────────────────────┘
```

**Secure Monitor Pattern:**
- BL31 acts as trusted intermediary between secure and normal worlds
- Exception Level 3 (EL3) is the highest privilege level
- Implements world switching via SMC (Secure Monitor Call)

**Service-Oriented Runtime:**
- Runtime services registered via descriptor tables
- SMC-based invocation (function ID routing)
- Modular service registration using `DECLARE_RT_SVC()` macro

### Code Design Patterns

**1. Registry Pattern (Runtime Services):**
```c
DECLARE_RT_SVC(
    std_svc,
    OEN_STD_START,
    OEN_STD_END,
    SMC_TYPE_FAST,
    std_svc_setup,
    std_svc_smc_handler
);
```
- Services self-register into `.rt_svc_descs` linker section
- Runtime dispatcher uses function ID to route calls
- Location: `include/common/runtime_svc.h`, `common/runtime_svc.c`

**2. Strategy Pattern (Platform Abstraction):**
- Platform API defined in `include/plat/common/platform.h`
- Each platform implements required interfaces
- Example: `plat_get_my_entrypoint()`, `plat_setup_psci_ops()`
- Allows platform-specific behavior without modifying core code

**3. Template Method (Boot Flow):**
```c
bl31_main() {
    bl31_lib_init();           // Common
    bl31_platform_setup();     // Platform-specific hook
    bl31_plat_arch_setup();    // Platform-specific hook
    runtime_svc_init();        // Common
    bl31_prepare_next_image(); // Common
}
```
- Defined in `bl31/bl31_main.c`
- Fixed algorithm with customizable steps

**4. Singleton Pattern (CPU Data):**
- Per-CPU data accessed via special register (TPIDR_EL3)
- Ensures single instance per CPU core
- Implementation: `lib/el3_runtime/cpu_data_array.S`

**5. Factory Pattern (Image Loading):**
- `load_auth_image()` in `bl2/bl2_image_load_v2.c`
- Creates appropriate image loaders based on type
- Handles authentication, decompression, placement

**6. Chain of Responsibility (Exception Handling):**
- Exception vectors → Synchronous/Async handlers → Service handlers
- Each level can handle or pass to next
- Implementation: `bl31/aarch64/runtime_exceptions.S`

**7. Facade Pattern (Translation Tables):**
- `lib/xlat_tables_v2/` provides simplified MMU interface
- Hides complexity of page table management
- API: `mmap_add_region()`, `init_xlat_tables()`

**8. Observer Pattern (Interrupt Management):**
- Interrupt sources register handlers
- GIC driver notifies registered handlers
- Implementation: `bl31/interrupt_mgmt.c`

### Code Organization and Naming Conventions

**File Naming:**
- Headers: `<component>.h` in `include/`
- Implementation: `<component>.c` or `<component>.S`
- Platform-specific: `<platform>_<function>.c`
- Makefiles: `<component>.mk`

**Function Naming:**
- Lowercase with underscores: `psci_cpu_on()`, `bl31_main()`
- Platform hooks prefixed: `plat_*()`, `bl31_plat_*()`
- Internal/static: often prefixed with `_` or component name

**Macro Naming:**
- Uppercase: `PSCI_E_SUCCESS`, `ENABLE_ASSERTIONS`
- Build options: `DEBUG`, `LOG_LEVEL`, `PLAT`

**Type Naming:**
- Suffixed with `_t`: `cpu_context_t`, `entry_point_info_t`
- Structures: `struct <name>` or typedef to `<name>_t`

**Directory Organization:**
- Architecture-specific subdirs: `aarch64/`, `aarch32/`
- Platform code: `plat/<vendor>/<platform>/`
- Common utilities: `common/`, `lib/`
- Public headers: `include/`
- Private headers: co-located with implementation

### Configuration Management Patterns

**Compile-Time Configuration:**
- Build options in `make_helpers/defaults.mk`
- Platform overrides in `plat/<vendor>/platform.mk`
- Preprocessor defines: `#if ENABLE_RME`, `#if DEBUG`

**Runtime Configuration:**
- Device Tree based: FCONF framework (`lib/fconf/`)
- `tb_fw_config.dts` → parsed at runtime → configure firmware
- Transfer List: boot parameter passing between stages

**Feature Detection:**
- Runtime CPU feature detection: `lib/arch_features.h`
- `is_armv8_2_feat_<name>_present()` functions
- Conditional code execution based on hardware capabilities

### Error Handling and Logging Strategies

**Error Handling:**
- Return codes: negative for errors (e.g., `PSCI_E_INVALID_PARAMS = -3`)
- Success: 0 or positive values
- Critical errors: `assert()` or `panic()`
- Assertion checking controlled by `ENABLE_ASSERTIONS` build option

**Panic Handling:**
- `panic()` function in `common/debug.c`
- Registers saved for crash analysis
- Platform can implement custom panic handler
- Crash reporting in `bl31/aarch64/crash_reporting.S`

**Logging:**
- Unified logging framework: `tf_log()` in `common/tf_log.c`
- Log levels: ERROR (10), NOTICE (20), WARNING (30), INFO (40), VERBOSE (50)
- Controlled via `LOG_LEVEL` build option
- Macros: `ERROR()`, `NOTICE()`, `WARN()`, `INFO()`, `VERBOSE()`
- Console abstraction allows multiple output devices

**Debug Support:**
- `DEBUG=1` build enables assertions and verbose logging
- Performance Measurement Framework (PMF): `lib/pmf/`
- Runtime instrumentation: timestamps for boot stages
- Memory map reporting: `make memmap`

## 5. Entry Points and Data Flow

### Application Entry Points

**Hardware Reset Sequence:**
```
CPU Reset
    ↓
[BL1 Entry] bl1/aarch64/bl1_entrypoint.S
    ↓
bl1_setup() - bl1/bl1_main.c
    ↓
bl1_main() - Load BL2, authenticate, transfer control
    ↓
[BL2 Entry] bl2/aarch64/bl2_entrypoint.S
    ↓
bl2_setup() - bl2/bl2_main.c
    ↓
bl2_main() - Load BL31/BL32/BL33, setup parameters
    ↓
[BL31 Entry] bl31/aarch64/bl31_entrypoint.S
    ↓
bl31_main() - bl31/bl31_main.c
    ↓
Runtime Service Loop (SMC handling)
```

**Assembly Entry Points:**
1. **BL1**: `bl1/aarch64/bl1_entrypoint.S:bl1_entrypoint`
   - Executed from reset vector
   - Sets up execution environment (stack, BSS)
   - Calls `bl1_setup()`, then `bl1_main()`

2. **BL2**: `bl2/aarch64/bl2_entrypoint.S:bl2_entrypoint`
   - Entered from BL1
   - Initializes execution state
   - Calls `bl2_setup()`, then `bl2_main()`

3. **BL31**: `bl31/aarch64/bl31_entrypoint.S:bl31_entrypoint`
   - Primary CPU entry point
   - Secondary CPU entry: `bl31_warm_entrypoint`
   - Prepares EL3 environment, calls `bl31_main()`

**Runtime Entry Points:**
- **Exception Vectors**: `bl31/aarch64/runtime_exceptions.S`
  - Synchronous exceptions (SMC calls)
  - IRQ, FIQ (interrupt handling)
  - SError (system error)
  - Each exception type has secure/non-secure variants

- **SMC Handler**: `common/runtime_svc.c:handle_runtime_svc()`
  - Dispatches SMC calls to registered services
  - Parses function ID, routes to appropriate handler

### Data Flow Between Major Components

**Boot-Time Data Flow:**
```
BL1 (ROM)
    ↓ [Authenticated Image + Entry Point]
BL2 (Trusted Boot)
    ↓ [BL31 Args, BL32 Args, BL33 Args]
BL31 (Runtime Monitor)
    ↓ [Entry Point Info]
BL32 (Secure Payload) / BL33 (OS Bootloader)
```

**Entry Point Information Structure:**
- Defined in `include/common/bl_common.h:entry_point_info_t`
- Contains: entry point address, execution state (AArch64/AArch32), security state
- Passed between boot stages via registers or memory

**Parameter Passing:**
- **Legacy**: `bl31_params_t` structure (deprecated)
- **Modern**: Transfer List (`contrib/libtl/`) - extensible tag-value format
- **Device Trees**: Hardware config (HW_CONFIG), firmware config (FW_CONFIG)

**Runtime Data Flow (SMC Call Example):**
```
Normal World Application
    ↓ [SMC instruction]
BL31 Exception Handler (runtime_exceptions.S)
    ↓ [Save context, identify call type]
Runtime Service Dispatcher (runtime_svc.c)
    ↓ [Parse Function ID, route to service]
Specific Service Handler (e.g., PSCI)
    ↓ [Execute request, prepare return values]
Return to Normal World
    ↓ [Restore context, ERET]
Normal World Application (resumes)
```

**Context Switching Flow:**
- Context save: Secure/Non-secure CPU state in `cpu_context_t`
- Functions: `cm_el1_sysregs_context_save()`, `cm_el1_sysregs_context_restore()`
- Location: `lib/el3_runtime/context_mgmt.c`
- Per-CPU storage: accessed via TPIDR_EL3

### External Integrations and API Endpoints

**SMC Calling Convention:**
- Function ID in X0/W0
- Parameters in X1-X7 / W1-W7
- Return values in X0-X3 / W0-W3
- Defined by ARM SMC Calling Convention (DEN0028)

**PSCI Interface:**
- Standard power management API
- Functions: `CPU_ON`, `CPU_OFF`, `CPU_SUSPEND`, `SYSTEM_OFF`, `SYSTEM_RESET`
- Entry: SMC calls with PSCI function IDs
- Implementation: `lib/psci/`

**Trusted OS Interface (BL32 ↔ BL31):**
- Secure Monitor Call (SMC) based
- Examples: OP-TEE, Trusty OS
- Dispatcher in `services/spd/<spd>/`

**Normal World OS Interface:**
- PSCI for power management
- SDEI for delegated exception handling
- SMCCC for general secure service calls

**Platform-Specific Interfaces:**
- Platform initialization hooks: `plat_*_setup()`
- Power management: `plat_setup_psci_ops()`
- Topology: `plat_get_power_domain_tree_desc()`
- Defined in: `include/plat/common/platform.h`

**Hardware Interfaces:**
- GIC (Generic Interrupt Controller): `drivers/arm/gic/`
- UART/Console: `drivers/console/`
- Timers: System counter, architectural timer
- Secure/Non-secure memory regions: TZC configuration

### Configuration Loading and Environment Setup

**BL1 Setup:**
1. Initialize architectural state (EL3, MMU off)
2. Setup exception vectors
3. Initialize stack, BSS
4. Platform-specific initialization: `bl1_platform_setup()`
5. Configure MMU: `bl1_plat_arch_setup()`
6. Load and authenticate BL2

**BL2 Setup:**
1. Architectural setup (similar to BL1)
2. Platform initialization: `bl2_platform_setup()`
3. Parse configuration DTBs (if using FCONF)
4. Load images: BL31, BL32, BL33, SCP firmware
5. Authenticate images (Trusted Boot)
6. Prepare handoff parameters (Transfer List or legacy)

**BL31 Setup (bl31_main.c:bl31_main()):**
1. Initialize libraries: `bl31_lib_init()` → context management
2. Platform setup: `bl31_platform_setup()` → GIC, console, topology
3. Architectural setup: `bl31_plat_arch_setup()` → MMU, caches
4. Initialize runtime services: `runtime_svc_init()`
5. Setup interrupt handling: GIC initialization
6. Initialize PSCI: CPU power state tables
7. Prepare next image entry point
8. Execute ERET to BL32 or BL33

**FCONF (Firmware Configuration) Flow:**
1. BL2 loads DTBs specified in FIP
2. Parse configuration: `fconf_populate()`
3. Configuration stored in global data structures
4. Later stages query configuration via getters
5. Example: `FCONF_GET_PROPERTY(dyn_cfg, dtb, TB_FW_CONFIG_ID)`

### Key Business Logic Workflows

**Trusted Boot Workflow:**
```
1. BL1 loads BL2 from flash/FIP
2. BL1 validates BL2 certificate chain
3. BL1 verifies BL2 signature (ROT Public Key)
4. BL1 transfers control to BL2
5. BL2 validates BL31, BL32, BL33 similarly
6. Chain of trust: ROT → BL2 Cert → Trusted Key Cert → BL3x Certs
```
- Implementation: `drivers/auth/`, `bl1/bl1_fwu.c`, `bl2/bl2_image_load_v2.c`

**CPU Power On (PSCI_CPU_ON) Workflow:**
```
1. Normal world calls PSCI_CPU_ON via SMC
2. BL31 routes SMC to PSCI handler (lib/psci/psci_on.c)
3. Validate target CPU MPIDR
4. Allocate context for target CPU
5. Call platform hook: plat_psci_ops->pwr_domain_on()
6. Platform performs low-level CPU power on
7. Target CPU starts at warm boot entry point
8. Target CPU initializes, enters Normal world
```

**Interrupt Handling Workflow:**
```
1. Interrupt occurs (IRQ/FIQ)
2. CPU traps to BL31 exception vector
3. Save current world context
4. Query GIC for interrupt ID
5. Route to appropriate handler:
   - Non-secure interrupt → return to NS world
   - Secure interrupt → call secure handler
6. Restore context, return (ERET)
```
- Implementation: `bl31/interrupt_mgmt.c`, `drivers/arm/gic/`

**Secure Partition Loading (BL2):**
```
1. Parse SP_LAYOUT_FILE (build-time generated)
2. For each Secure Partition:
   a. Load SP image from FIP
   b. Parse SP manifest (DTS)
   c. Allocate memory regions
   d. Setup MMU mappings
   e. Prepare SP entry point info
3. Package SP descriptors for BL31
4. BL31 initializes SPs via SPMC/SPMD
```

## 6. Build and Development Workflow

### Build System and Compilation Process

**Standard Build Invocation:**
```bash
export CROSS_COMPILE=aarch64-none-elf-
make PLAT=fvp DEBUG=1 LOG_LEVEL=40 all
```

**Build Process Flow:**
1. **Configuration Phase**:
   - Include `make_helpers/defaults.mk` - set default options
   - Parse command-line options (PLAT, DEBUG, etc.)
   - Include platform makefile: `plat/<PLAT>/platform.mk`
   - Platform can override defaults
   - Include feature makefiles based on options

2. **Source Collection Phase**:
   - Each BL stage defines `BL<N>_SOURCES`
   - Platform adds to sources: `BL<N>_SOURCES +=`
   - Libraries add sources via their makefiles
   - Sort and deduplicate source lists

3. **Compilation Phase**:
   - Compile C files: `$(CC) $(CFLAGS) $(DEFINES) -c`
   - Assemble .S files: `$(AS) $(ASFLAGS) $(DEFINES)`
   - Preprocess linker scripts: `$(CPP) *.ld.S → *.ld`
   - Build build tools (fiptool, cert_create)

4. **Linking Phase**:
   - Link each BL stage: `$(LD) $(LDFLAGS) -T bl<N>.ld`
   - Create ELF binaries: `build/<PLAT>/<BUILD_TYPE>/bl<N>/bl<N>.elf`
   - Generate binary images: `$(OBJCOPY) -O binary bl<N>.elf bl<N>.bin`

5. **Packaging Phase**:
   - Generate certificates (if `GENERATE_COT=1`): `cert_create`
   - Create FIP: `fiptool create --tb-fw bl2.bin --soc-fw bl31.bin ...`
   - Output: `build/<PLAT>/<BUILD_TYPE>/fip.bin`

**Key Build Artifacts:**
```
build/<PLAT>/<BUILD_TYPE>/
├── bl1/
│   └── bl1.bin              # Boot ROM firmware
├── bl2/
│   └── bl2.bin              # Trusted boot firmware
├── bl31/
│   └── bl31.bin             # EL3 runtime monitor
├── bl32/
│   └── bl32.bin             # Secure payload (optional)
├── fip.bin                  # Firmware Image Package
├── fdts/
│   └── *.dtb                # Compiled device trees
└── tools/
    ├── fiptool/fiptool      # FIP tool binary
    └── cert_create/cert_create
```

**Makefile Modularity:**
- `make_helpers/build_macros.mk` - Generic build rules
- `make_helpers/toolchain.mk` - Compiler detection, flags
- `lib/<component>/<component>.mk` - Library-specific sources
- `bl<N>/bl<N>.mk` - Bootloader stage sources
- `plat/<vendor>/platform.mk` - Platform configuration

**Build Optimization:**
- Release builds: `-Os` (optimize for size)
- Debug builds: `-O0 -g` (no optimization, debug symbols)
- Link-time optimization: `ENABLE_LTO=1`
- Position Independent Executables: `ENABLE_PIE=1`

### Testing Strategy and Execution Commands

**Unit Testing:**
- Limited unit testing within TF-A tree
- Test Secure Payload (TSP) for integration testing
- Build TSP: `make PLAT=fvp SPD=tspd all`

**Integration Testing:**
- External TF-A Tests repository
- Tests PSCI, TFTF (Trusted Firmware Test Framework)
- Runs on FVP, QEMU, or real hardware

**Validation Commands:**
```bash
# Build with Test Secure Payload
make PLAT=fvp SPD=tspd BL33=<u-boot.bin> all fip

# Run on ARM Fast Model
FVP_Base_RevC-2xAEMvA \
  -C bp.secureflashloader.fname=build/fvp/debug/bl1.bin \
  -C bp.flashloader0.fname=build/fvp/debug/fip.bin

# Style checking
make checkcodebase CHECKPATCH=<path>/checkpatch.pl

# Check specific commit range
make checkpatch BASE_COMMIT=origin/master

# Memory map analysis
make PLAT=fvp memmap
```

**Static Analysis:**
- Coverity Scan (automated daily)
- Results: https://scan.coverity.com/projects/arm-software-arm-trusted-firmware
- Address issues flagged by Coverity

**Platform Testing:**
- Minimum: Boot Linux on Foundation Platform (FVP)
- Recommended: Run TF-A Tests suite
- Validate on target hardware before production

### Deployment Procedures and Requirements

**Development Deployment:**
1. Build firmware images
2. Load via JTAG/SWD debugger
3. Or network boot (TFTP) if platform supports
4. Use ARM Fast Models for pre-silicon testing

**Production Deployment:**
1. Build release firmware: `DEBUG=0`
2. Enable Trusted Boot: `TRUSTED_BOARD_BOOT=1`
3. Generate production certificates
4. Package into FIP
5. Flash to device using platform-specific tools:
   - eMMC/SD card: `dd` or vendor tools
   - SPI flash: `flashrom`, vendor programmers
   - QSPI: Platform-specific utilities

**Update Mechanisms:**
- Firmware Update (FWU): BL2U stage for authenticated updates
- Build FWU FIP: `make fwu_fip`
- Platform must support FWU framework

**Deployment Requirements:**
- Correct platform selection (`PLAT=`)
- Matching BL33 (U-Boot, UEFI) for target
- Proper device tree (DTB) for hardware variant
- Production keys for Trusted Boot

### Environment Setup and Prerequisites

**Toolchain Setup:**
```bash
# For AArch64
export CROSS_COMPILE=aarch64-none-elf-
export PATH=/path/to/gcc-arm/bin:$PATH

# For AArch32
export CROSS_COMPILE=arm-none-eabi-
```

**Python Environment:**
```bash
# Install Poetry
curl -sSL https://install.python-poetry.org | python3 -

# Install dependencies
poetry install

# Optional: Install docs dependencies
poetry install --with docs
```

**Additional Tools:**
```bash
# Device Tree Compiler
sudo apt-get install device-tree-compiler

# OpenSSL (for certificate generation)
sudo apt-get install openssl libssl-dev

# checkpatch (from Linux kernel)
wget https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/plain/scripts/checkpatch.pl
chmod +x checkpatch.pl
```

**Fast Models (for testing):**
- Download from ARM Developer website
- Requires license for some models
- Foundation Platform is free

### Common Development Commands and Scripts

**Build Commands:**
```bash
# Basic build
make PLAT=fvp all

# Debug build with verbose logging
make PLAT=fvp DEBUG=1 LOG_LEVEL=50 all

# Clean specific platform
make PLAT=fvp clean

# Clean all
make realclean

# Build specific stages
make PLAT=fvp bl1
make PLAT=fvp bl2
make PLAT=fvp bl31
make PLAT=fvp fip

# Build with Trusted Boot
make PLAT=fvp TRUSTED_BOARD_BOOT=1 \
  GENERATE_COT=1 CREATE_KEYS=1 all fip

# Build with OP-TEE
make PLAT=fvp SPD=opteed \
  BL32=<optee_os>/out/arm/core/tee-pager_v2.bin \
  BL33=<u-boot.bin> all fip

# Build documentation
make doc
# Output: build/docs/html/index.html
```

**Analysis Commands:**
```bash
# Memory map
make PLAT=fvp all
make PLAT=fvp memmap

# Generate compile_commands.json (for IDEs)
bear -- make PLAT=fvp all

# Code navigation
make cscope
# Then use: cscope -d
```

**Debugging:**
```bash
# Build with symbols
make PLAT=fvp DEBUG=1 all

# Launch FVP with debugger
FVP_Base_RevC-2xAEMvA \
  -C bp.secureflashloader.fname=build/fvp/debug/bl1.bin \
  -C bp.flashloader0.fname=build/fvp/debug/fip.bin \
  --cadi-server

# Connect GDB
aarch64-none-elf-gdb build/fvp/debug/bl31/bl31.elf
(gdb) target remote localhost:7100
```

## 7. Key Features Analysis

### 1. Power State Coordination Interface (PSCI)

**Purpose:** Standard API for CPU and system power management

**Implementation:**
- **Location**: `lib/psci/`
- **Key files**:
  - `psci_main.c` - Entry point, function dispatch
  - `psci_on.c` - CPU power on
  - `psci_off.c` - CPU power off
  - `psci_suspend.c` - CPU/system suspend
  - `psci_setup.c` - Initialization
  - `psci_private.h` - Internal data structures

**Architecture:**
- **Power Domain Tree**: Hierarchical representation (CPU → Cluster → System)
- **State Machine**: Per-CPU and per-domain state tracking
- **Platform Hooks**: `plat_psci_ops` structure with platform callbacks
  - Defined in platform code: `plat/<vendor>/<platform>_pm.c`
  - Examples: `pwr_domain_on()`, `pwr_domain_off()`, `get_sys_suspend_power_state()`

**Workflow (CPU_ON example):**
1. Normal world SMC call: `PSCI_CPU_ON (0xC4000003)`
2. BL31 routes to `psci_cpu_on()` in `lib/psci/psci_on.c:159`
3. Validate target MPIDR, check CPU state
4. Allocate CPU context, set entry point
5. Call platform hook: `psci_plat_pm_ops->pwr_domain_on()`
6. Platform powers on CPU via SCP or direct register writes
7. Target CPU enters warm boot entry point: `bl31_warm_entrypoint`
8. Initialize CPU, enter Normal world at specified address

**Design Strategy:**
- Generic core implementation in `lib/psci/`
- Platform-specific power control delegated via callbacks
- State management ensures atomicity and ordering
- Supports both flat and hierarchical topologies

**Performance Optimizations:**
- Cached power domain tree for fast lookups
- Per-CPU data accessed via TPIDR_EL3 (no locking)
- Fast path for common operations (CPU_ON, CPU_OFF)

### 2. Trusted Board Boot Requirements (TBBR)

**Purpose:** Secure boot chain with cryptographic verification

**Implementation:**
- **Location**: `drivers/auth/`, `bl1/bl1_fwu.c`, `bl2/bl2_image_load_v2.c`
- **Key files**:
  - `drivers/auth/auth_mod.c` - Authentication framework
  - `drivers/auth/crypto_mod.c` - Crypto abstraction (mbedTLS)
  - `drivers/auth/img_parser_mod.c` - Image parsing (X.509)
  - `drivers/auth/tbbr/tbbr_cot_*.c` - Chain of Trust descriptors

**Chain of Trust:**
```
Root of Trust Public Key (ROT)
    ↓ [validates]
Trusted Boot Firmware Certificate (BL2)
    ↓ [validates]
Trusted Key Certificate
    ↓ [validates]
SoC Firmware Key Certificate (BL31)
SoC Firmware Content Certificate (BL31 image hash)
    ↓ [validates]
BL31 Image
```

**Workflow:**
1. BL1 has ROT public key hash (burned in ROM or OTP)
2. For each image (BL2, BL31, BL32, BL33):
   a. Load certificate from FIP
   b. Verify certificate chain back to ROT
   c. Extract image hash from content certificate
   d. Load image, compute hash
   e. Compare hashes
   f. If match, image is trusted; else panic

**Algorithms:**
- **Signature**: RSA-2048/3072/4096 or ECDSA
- **Hash**: SHA-256/384/512
- **Certificates**: X.509 v3 format
- Crypto provided by mbedTLS library

**Build Integration:**
```bash
make PLAT=fvp TRUSTED_BOARD_BOOT=1 \
  GENERATE_COT=1 CREATE_KEYS=1 \
  MBEDTLS_DIR=<mbedtls> all fip
```
- Generates development keys (or uses provided keys)
- Creates certificates: `build/fvp/release/*.crt`
- Packages into FIP with images

**Security Features:**
- Anti-rollback: Image versions tracked, downgrades prevented
- Dynamic authentication disable (DYN_DISABLE_AUTH) - for debugging only
- Measured boot: extend hashes into TPM (if enabled)

### 3. Exception Level 3 (EL3) Runtime Monitor

**Purpose:** Trusted intermediary for secure/normal world switching

**Implementation:**
- **Location**: `bl31/`
- **Key files**:
  - `bl31_main.c` - Initialization
  - `aarch64/runtime_exceptions.S` - Exception vectors (1024 bytes)
  - `interrupt_mgmt.c` - Interrupt routing framework
  - `bl31_traps.c` - System register access traps (RAS, MPAM, etc.)

**Exception Vector Table Layout:**
```
Offset | Exception Type              | Security State
-------+-----------------------------+----------------
0x000  | Sync (from Current EL, SP0) | -
0x080  | IRQ                         | -
0x100  | FIQ                         | -
0x180  | SError                      | -
0x200  | Sync (from Current EL, SPx) | -
0x280  | IRQ                         | -
0x300  | FIQ                         | -
0x380  | SError                      | -
0x400  | Sync (from Lower EL, AArch64)| Non-Secure
0x480  | IRQ                         | Non-Secure
0x500  | FIQ                         | Non-Secure
0x580  | SError                      | Non-Secure
0x600  | Sync (from Lower EL, AArch32)| Non-Secure
...
```
- Implementation: `bl31/aarch64/runtime_exceptions.S:runtime_exceptions`

**SMC Handling Flow:**
1. Normal world executes `SMC` instruction
2. CPU traps to BL31 at offset 0x400 (Sync from Lower AArch64)
3. Save Non-secure context: `el3_exit` → `save_gp_pmr_pauth_regs`
4. Determine SMC type (Standard, OEM, Arch, etc.)
5. Dispatch to runtime service: `handle_runtime_svc()`
6. Service executes, returns result in X0-X3
7. Restore Non-secure context: `el3_exit` → `restore_gp_pmr_pauth_regs`
8. Execute `ERET` to return to Normal world

**World Switching:**
- **Context Save/Restore**: `lib/el3_runtime/context_mgmt.c`
  - Structures: `cpu_context_t` (GP regs, system regs, floating-point, etc.)
  - Functions: `cm_el1_sysregs_context_save()`, `cm_el1_sysregs_context_restore()`
- **Secure/Non-secure Separation**:
  - Separate context for each world per CPU
  - SCR_EL3.NS bit controls current world
  - Switching ensures no leakage between worlds

**Interrupt Routing:**
- Group 0 (Secure): FIQ to Secure world
- Group 1 Non-secure: IRQ to Normal world
- Group 1 Secure: Configurable
- Routing tables: `interrupt_mgmt.c:intr_type_descs[]`

**Design Strategy:**
- Minimal Trusted Computing Base (TCB) in EL3
- Delegate as much as possible to Secure-EL1 (BL32)
- Fast SMC handling for performance-critical operations

### 4. Memory Management (Translation Tables)

**Purpose:** Configure MMU, manage virtual memory

**Implementation:**
- **Location**: `lib/xlat_tables_v2/` (preferred) or `lib/xlat_tables/` (legacy)
- **Key files**:
  - `xlat_tables_core.c` - Core mapping logic
  - `xlat_tables_context.c` - Context management
  - `xlat_tables_utils.c` - Helper functions

**API:**
```c
// Define memory regions
#define MAP_REGION_FLAT(adr, sz, attr) \
    MAP_REGION(adr, adr, sz, attr)

// Initialize tables
void init_xlat_tables(void);
void enable_mmu_el3(unsigned int flags);

// Dynamic mapping (xlat_tables_v2 only)
int mmap_add_dynamic_region(unsigned long long base_pa,
                             uintptr_t base_va,
                             size_t size,
                             mmap_attr_t attr);
int mmap_remove_dynamic_region(uintptr_t base_va, size_t size);
```

**Attributes:**
- **Memory Type**: Device, Normal (cacheable/non-cacheable)
- **Permissions**: RO, RW, Execute/No-Execute
- **Security**: Secure, Non-secure
- **Shareability**: Inner, Outer, Non-shareable

**Translation Table Levels:**
- Level 0: 512 GB per entry (rarely used)
- Level 1: 1 GB per entry (block or table)
- Level 2: 2 MB per entry (block or table)
- Level 3: 4 KB per entry (page)
- Automatic level selection based on region size/alignment

**Usage Pattern:**
1. Define static regions: `REGISTER_MMAP_REGION()` array in platform code
2. Initialize: `init_xlat_tables()` - allocates and populates tables
3. Enable MMU: `enable_mmu_el<N>()`
4. (Optional) Dynamic regions: `mmap_add_dynamic_region()` at runtime

**Optimizations:**
- Block mappings (2MB, 1GB) reduce table walks
- Minimal table allocation: only needed levels created
- Cache-friendly: translation table walks from L1/L2 cache

**Example (FVP platform):**
```c
const mmap_region_t plat_arm_mmap[] = {
    ARM_MAP_SHARED_RAM,          // Shared memory
    V2M_MAP_FLASH0_RO,           // Flash (Read-Only)
    V2M_MAP_IOFPGA,              // Device memory
    ARM_MAP_NS_DRAM1,            // Non-secure DRAM
    {0}                          // Terminator
};
```
- File: `plat/arm/board/fvp/fvp_common.c`

### 5. GIC (Generic Interrupt Controller) Driver

**Purpose:** Manage interrupt routing and handling

**Implementation:**
- **Location**: `drivers/arm/gic/`
- **Supported Versions**:
  - GICv2: `v2/gicv2_main.c`, `v2/gicv2_helpers.c`
  - GICv3: `v3/gicv3_main.c`, `v3/gicv3_helpers.c`
  - GIC-600: `v3/gic600.c`
  - GIC-700: `v3/gic700.c`

**Architecture (GICv3):**
```
Interrupt Sources
    ↓
Distributor (GICD) - routes SPIs to CPUs
    ↓
Redistributor (GICR) - per-CPU, routes PPIs/SGIs
    ↓
CPU Interface (ICC_* system registers)
    ↓
CPU
```

**Initialization Flow:**
1. **Distributor Init** (`gicv3_distif_init()`):
   - Disable distributor
   - Configure SPIs: priority, routing, edge/level
   - Enable distributor
   - File: `drivers/arm/gic/v3/gicv3_main.c:169`

2. **Redistributor Init** (`gicv3_rdistif_init()`):
   - Locate redistributor for current CPU
   - Configure SGIs, PPIs
   - Mark redistributor as awake
   - File: `drivers/arm/gic/v3/gicv3_main.c:268`

3. **CPU Interface Init** (`gicv3_cpuif_enable()`):
   - Enable system register interface
   - Set priority mask (allow all priorities)
   - Enable interrupt groups
   - File: `drivers/arm/gic/v3/gicv3_main.c:455`

**Interrupt Routing:**
- **Group 0**: Secure interrupts → FIQ
- **Group 1 Secure**: Secure interrupts → FIQ or IRQ (configurable)
- **Group 1 Non-secure**: Non-secure interrupts → IRQ
- Controlled by: GICD_IGROUPR, SCR_EL3.FIQ/IRQ routing bits

**BL31 Integration:**
- Platform provides: `GICD_BASE`, `GICR_BASE`, interrupt properties
- Initialized in: `bl31_platform_setup()` → `plat_arm_gic_init()`
- Interrupt handlers registered via `bl31/interrupt_mgmt.c`

**Performance Features:**
- Affinity routing: target specific CPUs for SPIs
- Priority-based preemption
- Fast EOI (End of Interrupt): split priority drop and deactivation

### 6. Secure Partition Manager (SPM)

**Purpose:** Manage and isolate Secure Partitions (S-EL0 or S-EL1 components)

**Implementation:**
- **Location**: `services/std_svc/spm/`
- **Variants**:
  - **SPM_MM** (deprecated): `spm_mm/` - Memory Management based
  - **SPMC at EL3**: `el3_spmc/` - Full SPM at EL3
  - **SPMD**: `spmd/` - Dispatcher to SPM at S-EL2

**Architecture (SPMD + Hafnium at S-EL2):**
```
Normal World (EL1)
    ↓ [FF-A calls via SMC]
BL31 / SPMD (EL3)
    ↓ [Route to S-EL2]
Hafnium SPM (S-EL2)
    ↓ [Manage, schedule]
Secure Partitions (S-EL1 / S-EL0)
```

**FF-A (Firmware Framework for Arm A-profile):**
- Standard interface for Secure Partition communication
- Functions: `FFA_VERSION`, `FFA_PARTITION_INFO_GET`, `FFA_MSG_SEND_DIRECT_REQ`
- Implemented by SPMD/SPMC

**Secure Partition Lifecycle:**
1. **BL2 Loading**:
   - Parse `SP_LAYOUT_FILE` (build time)
   - Load SP images from FIP
   - Parse SP manifests (DTS format)
   - Setup initial memory regions
   - File: `bl2/bl2_sp_load.c`

2. **BL31/SPMC Initialization**:
   - SPMD receives SP descriptors from BL2
   - Initialize SPM at S-EL2 (Hafnium) or EL3
   - Register SPs with SPM
   - File: `services/std_svc/spmd/spmd_main.c`

3. **Runtime**:
   - Normal world → SPMD (via SMC)
   - SPMD → SPMC (world switch to S-EL2)
   - SPMC schedules SP, delivers message
   - SP processes, returns result
   - Reverse path back to Normal world

**Use Cases:**
- Trusted services: Crypto, attestation, secure storage
- Platform security services: Firmware TPM (fTPM), DRM
- Isolation: Each SP runs in separate memory/execution context

**Files:**
- SPMD: `services/std_svc/spmd/spmd_main.c`
- EL3 SPMC: `services/std_svc/spm/el3_spmc/spmc_main.c`
- SP loading: `bl2/bl2_sp_load.c`

### 7. Realm Management Extension (RME)

**Purpose:** Arm Confidential Compute Architecture (CCA) support

**Implementation:**
- **Location**: `lib/gpt_rme/`, `services/std_svc/rmmd/`
- **Key Components**:
  - **GPT (Granule Protection Table)**: `lib/gpt_rme/gpt_rme.c`
  - **RMMD (RMM Dispatcher)**: `services/std_svc/rmmd/rmmd_main.c`
  - **TRP (Test Realm Payload)**: `services/std_svc/rmmd/trp/`

**Four-World Model:**
- **Root**: EL3 firmware (BL31) - highest privilege
- **Secure**: EL1 Secure world (BL32/TEE)
- **Realm**: EL1 Realm world (confidential VMs)
- **Non-secure**: EL1/EL2 Normal world (hypervisor, OS)

**Granule Protection:**
- 4KB granules (pages) tagged with security state
- GPT enforces access control in hardware
- Prevents Secure/Normal world from accessing Realm memory
- GPT managed by Root (BL31)

**Realm Management Monitor (RMM):**
- Runs at R-EL2 (Realm Exception Level 2)
- Manages Realm VMs
- Provided as separate binary or Test Realm Payload (TRP)
- Loaded by BL2, initialized by BL31/RMMD

**Workflow:**
1. **Boot**:
   - BL2 loads RMM image
   - BL31 initializes GPT: `gpt_init()` → `lib/gpt_rme/gpt_rme.c:307`
   - RMMD initializes RMM: `rmmd_setup()` → `services/std_svc/rmmd/rmmd_main.c:198`

2. **Realm Creation**:
   - Hypervisor requests Realm creation (via RMM ABI)
   - RMM allocates memory, sets up Realm context
   - GPT tags memory as Realm-accessible

3. **Realm Execution**:
   - Hypervisor schedules Realm VM
   - CPU enters Realm world (R-EL1)
   - Realm code runs in isolation
   - Exit via RMI (Realm Management Interface) calls

**Build:**
```bash
make PLAT=fvp ENABLE_RME=1 \
  ARM_ARCH_MAJOR=9 ARM_ARCH_MINOR=2 \
  RMM=<rmm_binary> all fip
```
- Requires ARMv9.2-A+ architecture
- Automatically sets `BL2_RUNS_AT_EL3=1`

**Files:**
- GPT: `lib/gpt_rme/gpt_rme.c`
- RMMD: `services/std_svc/rmmd/rmmd_main.c`
- TRP: `services/std_svc/rmmd/trp/trp_main.c`

### 8. Firmware Update (FWU)

**Purpose:** Authenticated firmware update mechanism

**Implementation:**
- **Location**: `bl1/bl1_fwu.c`, `bl2u/`, `drivers/fwu/`
- **Stages**:
  - **BL1**: Resident in ROM, coordinates update
  - **BL2U**: Update agent, loaded into RAM
  - **NS_BL1U**: Normal world update image (platform-specific)
  - **NS_BL2U**: Normal world update payload

**Update Flow:**
1. Normal world triggers update (platform-specific)
2. BL1 receives update request
3. BL1 authenticates BL2U image
4. BL1 loads and executes BL2U
5. BL2U authenticates and writes new firmware images
6. Reboot to new firmware

**Security:**
- All update images authenticated (part of CoT)
- Anti-rollback: version checks prevent downgrades
- Atomic updates: old firmware remains bootable if update fails

**Build:**
```bash
make PLAT=fvp TRUSTED_BOARD_BOOT=1 all fwu_fip
```
- Creates `fwu_fip.bin` with update images

**Files:**
- BL1 FWU: `bl1/bl1_fwu.c`
- BL2U: `bl2u/bl2u_main.c`
- FWU Metadata: `drivers/fwu/fwu.c` (PSA FWU specification)

## 8. Code Quality Assessment

### Documentation Coverage and Quality

**Comprehensive Documentation:**
- **Location**: `docs/` - 100+ reStructuredText files
- **Build System**: Sphinx 5.3.0+ with Read the Docs theme
- **Coverage**:
  - Getting started guides: build, prerequisites, initial setup
  - Design documents: architecture, boot flow, interrupt handling
  - Component docs: PSCI, SDEI, RME, SPM, etc.
  - Platform porting guides
  - Process docs: contributing, coding style, maintainers

**Quality Assessment:**
- ✅ **Excellent**: Architecture documentation (boot flow, security model)
- ✅ **Good**: API documentation for platform porting
- ⚠️ **Moderate**: Inline code comments (varies by component)
- ⚠️ **Sparse**: Some driver documentation

**Inline Comments:**
- Headers generally well-documented (Doxygen-style)
- Complex algorithms have explanatory comments
- Assembly code has instruction-level comments
- Some platform code lacks detailed comments

**Generated Documentation:**
- Build: `make doc` → `build/docs/html/`
- Published: https://trustedfirmware-a.readthedocs.io/
- Includes PlantUML diagrams for architecture

### Testing Coverage and Approaches

**Testing Levels:**

1. **Unit Testing**: Limited
   - No comprehensive unit test framework in tree
   - Test Secure Payload (TSP) provides basic integration tests
   - Location: `bl32/tsp/`

2. **Integration Testing**: Moderate
   - TF-A Tests (external repository)
   - TFTF (Trusted Firmware Test Framework)
   - Tests: PSCI compliance, SDEI, PMF, etc.
   - Runs on FVP, QEMU, real hardware

3. **System Testing**: Good
   - Boot testing on multiple platforms
   - Linux boot validation (minimum requirement)
   - OEM-specific test suites for production platforms

4. **Static Analysis**: Excellent
   - Daily Coverity Scan runs
   - Results publicly available
   - Active defect remediation
   - checkpatch.pl for coding style

**Test Execution:**
```bash
# Build with Test Secure Payload
make PLAT=fvp SPD=tspd all fip

# Run on FVP
FVP_Base_RevC-2xAEMvA \
  -C bp.secureflashloader.fname=build/fvp/debug/bl1.bin \
  -C bp.flashloader0.fname=build/fvp/debug/fip.bin \
  --run

# External: TF-A Tests
# Clone: https://git.trustedfirmware.org/TF-A/tf-a-tests.git
# Run test suite on target
```

**Coverage Gaps:**
- No code coverage metrics published
- Limited automated testing for platform-specific code
- RME, SPM features have less test coverage (newer features)

### Code Complexity and Technical Debt

**Complexity Indicators:**

**Low Complexity (Maintainable):**
- Build system (Makefiles): modular, well-structured
- PSCI library: clean abstraction, well-documented
- Translation tables v2: modern API, good design

**Moderate Complexity:**
- Context management: many architecture-specific variants
- Interrupt handling: complex routing logic
- GIC drivers: hardware complexity reflected in code

**High Complexity (Technical Debt):**
- Legacy translation tables v1: superseded by v2, still in use
- BL2 image loading: multiple code paths (v1 vs v2)
- Platform code duplication: similar code across vendors
- Build option sprawl: 100+ build options, complex interactions

**Technical Debt Areas:**

1. **Code Duplication**:
   - Platform code: many platforms duplicate initialization logic
   - Mitigation: ARM common platform code provides reusable components
   - Ongoing: refactoring to increase code reuse

2. **Legacy Code Paths**:
   - xlat_tables v1 still supported (should migrate to v2)
   - BL2 image loading v1 (v2 is preferred)
   - Old parameter passing (replaced by Transfer List)

3. **Build System**:
   - Complex Makefile dependencies (390+ .mk files)
   - Hard to trace which options affect which files
   - Improvement needed: dependency tracking

4. **Preprocessor Overuse**:
   - Heavy use of `#if` conditionals for features
   - Makes code harder to read and test
   - Alternatives: runtime configuration (FCONF) being adopted

**Positive Trends:**
- Active refactoring: xlat_tables v2, Transfer List, FCONF
- Deprecation process: old APIs marked, migration guides provided
- Continuous improvement: regular updates, community contributions

### Security Considerations and Practices

**Security Practices:**

1. **Secure Boot**: Mandatory for production
   - Cryptographic chain of trust
   - Anti-rollback protection
   - Development keys for testing, unique keys for production

2. **Memory Protection**:
   - MMU always enabled (after early boot)
   - Separate page tables for Secure/Non-secure
   - Stack protection: `ENABLE_STACK_PROTECTOR=all`
   - NX (No-Execute) for data regions

3. **Architectural Security**:
   - Pointer Authentication (ARMv8.3-A): `BRANCH_PROTECTION=1`
   - Branch Target Identification (ARMv8.5-A)
   - Speculation barriers where needed

4. **Code Safety**:
   - Assertions: `assert()` extensively used
   - Input validation: SMC parameters checked
   - Safe string functions: `strlcpy`, `strlcat` (custom libc)

5. **Cryptography**:
   - mbedTLS integration for Trusted Boot
   - True Random Number Generator (TRNG) support
   - PSA Crypto API support (emerging)

**Security Audits:**
- Regular security reviews by ARM and partners
- Vulnerability disclosure process
- Security advisories published for critical issues

**Known Limitations:**
- ⚠️ Default configuration may not be production-ready
- ⚠️ Platforms must enable Trusted Boot explicitly
- ⚠️ Production deployments must use unique keys
- ⚠️ Some features (e.g., DYN_DISABLE_AUTH) are debug-only

**Best Practices for Developers:**
- Always validate external inputs (SMC parameters, DTBs)
- Use safe memory functions (avoid `strcpy`, `memcpy` without bounds)
- Enable all security features: Trusted Boot, stack protector, pointer auth
- Perform penetration testing on production firmware
- Review security advisories: https://developer.trustedfirmware.org/

### Performance Considerations and Bottlenecks

**Performance-Critical Paths:**

1. **SMC Handling** (`bl31/aarch64/runtime_exceptions.S`):
   - Hot path: every secure service call
   - Optimizations:
     - Minimal context save/restore
     - Fast lookup of service descriptors
     - Inline assembly for critical sections
   - Typical latency: < 1 µs (depends on service)

2. **PSCI Operations** (`lib/psci/`):
   - CPU_ON, CPU_OFF: frequently used for power management
   - Optimizations:
     - Cached power domain tree
     - Per-CPU data via TPIDR_EL3 (no locks)
     - Platform hooks minimize EL3 work
   - Latency: dominated by platform hardware (SCP, power controllers)

3. **Interrupt Handling** (`bl31/interrupt_mgmt.c`):
   - Every interrupt routed through BL31
   - Optimizations:
     - Fast path for Non-secure interrupts (minimal overhead)
     - GIC EOI done quickly to minimize latency
   - Overhead: ~100-500 cycles for routing decisions

4. **MMU/Cache Operations** (`lib/xlat_tables_v2/`):
   - Dynamic mappings can impact performance
   - Optimizations:
     - Use block mappings (2MB, 1GB) to reduce TLB pressure
     - Minimal table walking (3-level max for 4KB granule)
     - Cache translation tables (Normal memory)
   - TLB maintenance: use targeted `TLBI` instructions

**Performance Measurement:**
- **PMF (Performance Measurement Framework)**: `lib/pmf/`
  - Timestamp critical operations
  - Build: `ENABLE_PMF=1`
  - Usage: `PMF_CAPTURE_TIMESTAMP()` macro
  - Report: `make memmap` shows timing data

- **Boot Time Optimization**:
  - Minimal BL1 (ROM constraints)
  - Parallel initialization where possible
  - Lazy initialization of unused components
  - Measured boot impact minimized (hash computation)

**Bottlenecks:**

1. **Cryptographic Operations**:
   - Trusted Boot: signature verification is slow (~10-100ms per image)
   - Mitigation: hardware crypto accelerators (if available)
   - Future: Measured boot (hash-only) as alternative

2. **Flash Access**:
   - Loading images from flash (NOR, QSPI)
   - Typically 10-50 MB/s
   - Mitigation: XIP (Execute In Place) for BL2 (`BL2_IN_XIP_MEM=1`)

3. **Serial Console**:
   - Logging can dominate boot time
   - Mitigation: `LOG_LEVEL=20` (NOTICE) for production, reduce verbosity

4. **DTB Parsing**:
   - FCONF parses DTBs at runtime
   - Can add 1-5ms to boot
   - Mitigation: minimize DTB size, parse only needed nodes

**Optimization Recommendations:**
- Enable compiler optimizations: `-Os` (default for release)
- Use LTO (Link-Time Optimization): `ENABLE_LTO=1` (reduces size, may improve speed)
- Profile with PMF to identify hotspots
- Minimize `LOG_LEVEL` for production builds
- Use hardware crypto accelerators for Trusted Boot
- Consider XIP for BL2 on flash-constrained platforms

## 9. Recommended Prompts for Deeper Investigation

### Architecture Deep Dives

**Boot Flow and Initialization:**
- "Trace the complete boot flow from BL1 reset to BL33 execution, including all initialization steps"
- "How does BL2 load and authenticate BL31? Show the complete workflow with file references"
- "Explain the warm boot path for secondary CPUs - how does PSCI_CPU_ON work end-to-end?"
- "What happens during a cold boot vs warm boot? Trace the differences in code paths"

**Memory Management:**
- "How does the xlat_tables_v2 library manage page tables? Explain the data structures and algorithms"
- "Trace a call to mmap_add_dynamic_region() - what happens at each step?"
- "How are secure and non-secure memory regions separated? Show the MMU configuration"
- "What is the memory layout for BL31? Where are stack, heap, code, and data regions located?"

**Security Architecture:**
- "Explain the world switching mechanism - how does BL31 transition between Secure and Normal worlds?"
- "How is the Chain of Trust implemented? Trace the certificate validation for a BL31 image"
- "What security features are enabled by BRANCH_PROTECTION=1? How are they implemented?"
- "Analyze the attack surface of BL31 - what are the entry points and how are they protected?"

**Interrupt and Exception Handling:**
- "How does the GIC driver route interrupts to the appropriate world (Secure vs Non-secure)?"
- "Trace an IRQ from hardware assertion to the interrupt handler in the Normal world"
- "How are FIQs and IRQs configured differently for Secure interrupts?"
- "What is the Exception Handling Framework (EHF) and how does it prioritize exceptions?"

### Feature Analysis

**PSCI Deep Dive:**
- "How is the power domain tree constructed and used by PSCI? Show data structures and algorithms"
- "Trace PSCI_CPU_SUSPEND from SMC call to platform hook - what happens at each layer?"
- "How does PSCI handle multi-cluster systems? Explain the hierarchical power management"
- "What are the PSCI state machines for CPU and system power states?"

**Trusted Boot:**
- "How does the authentication module work? Trace auth_mod_verify_img() with detailed steps"
- "What is the format of the Trusted Key Certificate? How is it parsed and validated?"
- "How are anti-rollback counters implemented? Where are they stored and checked?"
- "Compare TBBR (Trusted Board Boot) vs Measured Boot - what are the differences?"

**Secure Partition Management:**
- "How are Secure Partitions loaded by BL2? Show the complete lifecycle"
- "Explain the FF-A protocol - how do Normal world and Secure Partitions communicate?"
- "What is the difference between SPMC at EL3 vs SPMD with Hafnium at S-EL2?"
- "How does memory sharing work between Normal world and a Secure Partition?"

**Realm Management Extension:**
- "How does the Granule Protection Table (GPT) enforce memory isolation for Realms?"
- "Trace the initialization of RME - from GPT setup to RMM loading"
- "What is the four-world model? How does BL31 manage transitions between worlds?"
- "How are Realm VMs created and managed? Show the RMI (Realm Management Interface) calls"

### Code Quality and Maintenance

**Refactoring Opportunities:**
- "Identify duplicated code across platform implementations - what can be refactored into common/"
- "What are the main differences between xlat_tables v1 and v2? Create a migration guide"
- "Analyze the build system complexity - which .mk files are most complex and why?"
- "Which #if preprocessor conditionals could be replaced with runtime configuration (FCONF)?"

**Testing and Validation:**
- "What test coverage exists for the PSCI library? Identify gaps and recommend tests"
- "How is the Test Secure Payload (TSP) used for validation? What does it test?"
- "What static analysis tools are used on TF-A? How are issues tracked and resolved?"
- "Create a test plan for validating Trusted Boot on a new platform"

**Technical Debt:**
- "Identify deprecated APIs and code paths - what is the migration strategy?"
- "What are the oldest components in the codebase? Which need modernization?"
- "Analyze the complexity of bl2/bl2_image_load_v2.c - can it be simplified?"
- "Which build options interact in complex ways? Document the dependencies"

**Documentation Gaps:**
- "Which drivers lack comprehensive documentation? Prioritize by importance"
- "Create a detailed guide for porting TF-A to a new ARMv8-A SoC"
- "Document the runtime service registration mechanism with code examples"
- "Write a troubleshooting guide for common BL31 boot failures"

### Integration and Dependencies

**Platform Integration:**
- "How does a platform integrate a custom Secure Payload (BL32)? Show the complete process"
- "What hooks must a platform implement for PSCI? Create a checklist"
- "How can a platform add a new runtime service? Provide a step-by-step guide"
- "Explain the platform initialization sequence - what happens at each bl31_*_setup() call?"

**External System Integration:**
- "How does TF-A integrate with OP-TEE? Trace the communication between BL31 and OP-TEE"
- "What is the interface between BL31 and the System Control Processor (SCP)?"
- "How does TF-A pass information to U-Boot/UEFI (BL33)? What data structures are used?"
- "Explain the integration with Arm Firmware Framework (FF-A) compliant SPM"

**Dependency Analysis:**
- "What are the dependencies of the PSCI library? Create a dependency graph"
- "How does BL31 depend on platform-specific code? Identify the abstraction boundaries"
- "Trace the dependency chain from a build option (e.g., ENABLE_RME) to code changes"
- "Which components depend on mbedTLS? What happens if a different crypto library is used?"

**Impact Analysis:**
- "If the GIC driver is modified, what other components are affected?"
- "What is the impact of changing the translation table implementation (v1 → v2)?"
- "How does enabling RME affect the boot flow and memory layout?"
- "What breaks if a platform changes PLATFORM_CORE_COUNT after initial development?"

### Development Workflow

**Build and Deployment:**
- "Create a complete build and deployment guide for the FVP platform with Trusted Boot"
- "How are build options processed and propagated through the Makefile system?"
- "What is the FIP (Firmware Image Package) format? How is it created and parsed?"
- "How can I build TF-A with custom certificates for Trusted Boot?"

**Debugging Techniques:**
- "What are the best practices for debugging BL31 crashes? Show tools and techniques"
- "How can I use the Performance Measurement Framework (PMF) to profile boot time?"
- "Explain the crash reporting mechanism - what information is captured?"
- "How do I enable verbose logging for a specific component (e.g., PSCI)?"

**Configuration Management:**
- "How does the FCONF (Firmware Configuration) framework work? Provide usage examples"
- "What is the Transfer List format? How is it used to pass data between boot stages?"
- "How can a platform override default build options? Show the precedence order"
- "Create a guide for using Device Trees to configure TF-A firmware"

**Error Handling and Recovery:**
- "How are errors propagated from platform code back to the Normal world?"
- "What happens when a Trusted Boot authentication failure occurs? Trace the code path"
- "How does BL31 handle unexpected exceptions (e.g., undefined instruction)?"
- "What recovery mechanisms exist for firmware update failures?"

---

**End of Comprehensive Codebase Analysis**

This document serves as a foundation for understanding the Trusted Firmware-A codebase. Use the recommended prompts to dive deeper into specific areas of interest. For questions or clarifications, refer to the official documentation at https://trustedfirmware-a.readthedocs.io/ or consult the CLAUDE.md file in this repository.
