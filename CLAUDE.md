# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **📚 For comprehensive codebase analysis**: See `code_analysis.md` for in-depth architecture details, design patterns, feature implementations, and investigative prompts for deeper exploration.

## Project Overview

Trusted Firmware-A (TF-A) is a reference implementation of secure world software for Arm A-Profile architectures (Armv8-A and Armv7-A). It serves as secure bootloader and runtime firmware, implementing the EL3 Secure Monitor and providing the foundation for Trusted Execution Environment (TEE) support.

Current version: **2.13**

### Key Standards Implemented

- Power State Coordination Interface (PSCI)
- Trusted Board Boot Requirements CLIENT (TBBR-CLIENT)
- SMC Calling Convention
- System Control and Management Interface (SCMI)
- Software Delegated Exception Interface (SDEI)
- Realm Management Extension (RME) - for Arm confidential computing

## Build System

### Basic Build Commands

The build system is Make-based and highly configurable through build options.

```bash
# Set cross-compiler (required before building)
export CROSS_COMPILE=<path-to-aarch64-gcc>/bin/aarch64-none-elf-

# Basic build (defaults to FVP platform, release mode, AArch64)
make PLAT=<platform> all

# Debug build
make PLAT=<platform> DEBUG=1 all

# AArch32 build
make PLAT=<platform> ARCH=aarch32 AARCH32_SP=sp_min all

# Build specific bootloader stages
make PLAT=<platform> bl1
make PLAT=<platform> bl2
make PLAT=<platform> bl31

# Build Firmware Image Package (FIP)
make PLAT=<platform> fip

# Clean build
make PLAT=<platform> clean

# Clean all platforms
make realclean
```

### Common Build Options

- `PLAT=<platform>` - Target platform (required, defaults to `fvp`)
- `ARCH=aarch64|aarch32` - Target architecture (defaults to `aarch64`)
- `DEBUG=0|1` - Debug build (0=release, 1=debug, defaults to 0)
- `LOG_LEVEL=<0-50>` - Logging level (release: 20=NOTICE, debug: 40=INFO)
- `TRUSTED_BOARD_BOOT=1` - Enable secure boot chain
- `GENERATE_COT=1` - Generate Chain of Trust certificates
- `SPD=<spd>` - Secure Payload Dispatcher (e.g., `opteed`, `trusty`, `spmd`)
- `BL32=<path>` - Path to pre-built BL32 image (Trusted OS)
- `BL33=<path>` - Path to BL33 image (normal world bootloader like U-Boot/UEFI)
- `ENABLE_RME=1` - Enable Realm Management Extension support
- `BRANCH_PROTECTION=<0-5>` - Enable ARM pointer authentication and BTI

### Build Outputs

Build artifacts are placed in: `build/<platform>/<build-type>/`
- `bl1.bin` - Boot ROM firmware
- `bl2.bin` - Trusted boot firmware (RAM-based)
- `bl31.bin` - EL3 runtime monitor (AArch64 only)
- `bl32.bin` - Secure-EL1 payload (TEE/Trusted OS)
- `fip.bin` - Firmware Image Package containing all images

### Testing

```bash
# Run checkpatch on entire codebase
make checkcodebase CHECKPATCH=<path-to-checkpatch.pl>

# Check style on current branch against base
make checkpatch BASE_COMMIT=origin/master

# Generate memory map of built binaries
make PLAT=<platform> memmap

# Build documentation
make doc
```

## Code Architecture

### Boot Flow Stages

TF-A implements a multi-stage secure boot architecture:

1. **BL1** (`bl1/`) - Boot ROM firmware at EL3
   - First code executed on Application Processor
   - Minimal initialization, loads and authenticates BL2
   - Typically ROM-based, cannot be modified

2. **BL2** (`bl2/`) - Trusted boot firmware at Secure-EL1/EL3
   - Runs in RAM, performs additional hardware initialization
   - Loads and authenticates subsequent boot stages (BL31, BL32, BL33)
   - Can run at EL3 when `RESET_TO_BL2=1` or `ENABLE_RME=1`

3. **BL31** (`bl31/`) - EL3 Runtime Monitor (AArch64 only)
   - Secure Monitor running at Exception Level 3
   - Handles world switching between secure and normal worlds
   - Implements PSCI, SMC handling, interrupt management
   - Key files:
     - `bl31_main.c` - Main initialization
     - `bl31_traps.c` - Exception/trap handling
     - `interrupt_mgmt.c` - Interrupt routing

4. **BL32** (`bl32/`) - Secure-EL1 Payload
   - Trusted OS or minimal secure payload
   - Provides secure services to normal world
   - Options: OP-TEE (`bl32/optee/`), SP_MIN (`bl32/sp_min/`), TSP (`bl32/tsp/`)

5. **BL33** - Normal world bootloader (external, e.g., U-Boot, UEFI)

### Directory Structure

```
bl1/, bl2/, bl2u/, bl31/, bl32/  - Bootloader stage implementations
common/                          - Shared bootloader code
lib/                             - Core libraries
  ├── cpus/                      - CPU-specific code and errata workarounds
  ├── el3_runtime/               - EL3 context management
  ├── extensions/                - ARM architecture extensions (PAAuth, SVE, etc.)
  ├── psci/                      - PSCI implementation
  ├── xlat_tables_v2/            - MMU translation table library
  ├── fconf/                     - Firmware Configuration framework
  └── gpt_rme/                   - Granule Protection Tables (RME)
plat/                            - Platform-specific implementations
  ├── arm/                       - ARM reference platforms (FVP, Juno, etc.)
  ├── common/                    - Common platform code
  └── <vendor>/                  - Vendor-specific platforms
drivers/                         - Hardware drivers (console, MMC, USB, crypto, etc.)
services/                        - Runtime services
  ├── std_svc/                   - Standard services (PSCI, TRNG, etc.)
  ├── spd/                       - Secure Payload Dispatchers
  └── arm_arch_svc/              - ARM architecture services
tools/                           - Build tools
  ├── fiptool/                   - FIP creation tool
  ├── cert_create/               - Certificate generation
  └── sptool/                    - Secure partition packaging
include/                         - Public headers
fdts/                            - Device tree files
docs/                            - Documentation (Sphinx-based)
make_helpers/                    - Build system infrastructure
```

### Platform Porting

Each platform implementation is in `plat/<vendor>/`:
- `platform.mk` - Platform-specific build configuration
- `platform_def.h` - Platform definitions (memory maps, limits)
- `<platform>_bl<N>_setup.c` - BL stage-specific setup
- `<platform>_topology.c` - CPU topology
- `<platform>_pm.c` - Power management (PSCI implementation)

Platforms must implement mandatory interfaces defined in:
- `include/plat/common/platform.h` - Core platform API
- `include/plat/arm/common/plat_arm.h` - ARM platform helpers (if using ARM common code)

### Key Libraries and Services

**Translation Tables** (`lib/xlat_tables_v2/`):
- Memory management unit (MMU) configuration
- Use `xlat_tables_v2` for new code (supports dynamic mappings)

**PSCI** (`lib/psci/`):
- Power state management
- CPU on/off, system suspend/resume
- Platform hooks in `plat/<platform>/<platform>_pm.c`

**CPU Support** (`lib/cpus/`):
- CPU-specific initialization
- Errata workarounds (critical for production)
- File naming: `lib/cpus/aarch64/<cpu_name>.S`

**Exception Handling** (`bl31/bl31_traps.c`, `lib/el3_runtime/`):
- SMC handling
- Synchronous/asynchronous exception routing
- Context save/restore for world switching

## Development Workflow

### Code Style

- Follow Linux kernel coding style
- Use `make checkcodebase` with Linux's `checkpatch.pl`
- 8-space tabs for indentation, 80-column limit (some exceptions allowed)
- Function/variable names: lowercase with underscores

### Patch Submission

- Patches are submitted via Gerrit at `review.trustedfirmware.org`
- Use `git review` to submit patches (rebases onto `integration` branch)
- Each patch needs:
  - `Code-Owner-Review+1` for each modified module
  - `Maintainer-Review+1`
  - `Verified+1` from CI
- Patches merge to `integration` branch, then to `master` after integration testing

### Testing Requirements

- Minimum: Linux boots on Foundation FVP
- Run TF-A Tests suite for comprehensive validation
- All CI tests must pass
- Test both debug and release builds

### Important Files for Changes

**When modifying boot flow:**
- Update linker scripts (`bl*/bl*.ld.S`)
- Check image size constraints in `plat/<platform>/platform_def.h`
- Verify `BL*_BASE`, `BL*_LIMIT` definitions

**When adding platform support:**
- Create `plat/<vendor>/<platform>/` directory
- Provide `platform.mk` and required setup files
- Update `docs/plat/` with porting guide
- Add platform to CI configuration

**When adding features:**
- Consider backward compatibility
- Add build option in `make_helpers/defaults.mk`
- Document in `docs/getting_started/build-options.rst`
- Update `docs/design/` for architecture changes

## Important Constraints

- **BL31 is AArch64 only** - AArch32 uses BL32 (sp_min) for EL3 runtime
- **Image size limits** - Each BL stage has strict size limits defined by platform
- **Build dependency** - Build system doesn't track dependencies for build options; use clean build when changing options
- **Security validation** - All production code should undergo security validation and penetration testing

## Debugging

### Debug Build

```bash
make PLAT=<platform> DEBUG=1 LOG_LEVEL=40 all
```

### Common Debug Options

- `LOG_LEVEL=<0-50>` - Control verbosity (0=none, 50=verbose)
- `ENABLE_ASSERTIONS=1` - Enable assert() statements
- `ENABLE_PMF=1` - Performance Measurement Framework
- `CRASH_REPORTING=1` - Enhanced crash reporting
- `CFLAGS="-O0"` - Disable optimization for debugging

### Useful Debugging Techniques

- Use `tf_log()` for logging (defined in `common/tf_log.c`)
- Check console output via UART (driver in `drivers/console/`)
- Use ARM Fast Models or QEMU for emulation/debugging
- Enable stack protector: `ENABLE_STACK_PROTECTOR=all`

## Learning and Documentation

### Personal Notebook

This repository includes a `notebook.md` file for capturing personal learning notes and insights about TF-A.

**Important**: When answering questions about TF-A concepts or implementation details, always ask the user if they would like you to capture the key insights and understandings in their `notebook.md` file. This helps build a personal reference guide over time.

Example workflow:
1. User asks: "What is MPIDR?"
2. You provide a detailed answer
3. You ask: "Would you like me to add this explanation to your notebook.md for future reference?"
4. If yes, append the explanation to notebook.md with proper formatting and date

### Additional Documentation

**For deeper analysis, see:**
- `code_analysis.md` - Comprehensive codebase analysis including:
  - Complete technology stack and dependencies
  - Design pattern analysis and code organization
  - Detailed feature implementations with workflows
  - Performance analysis and optimization strategies
  - 50+ recommended prompts for deeper investigation

## Resources

- Full documentation: https://trustedfirmware-a.readthedocs.io/
- Mailing list: TF-A mailing list (for design discussions)
- Code review: https://review.trustedfirmware.org/
- Issue tracking: https://developer.trustedfirmware.org/
