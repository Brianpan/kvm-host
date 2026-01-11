# RSI Register and Boot Parameters in KVM

**Session:** rsi_boot_params_explanation
**Date:** 2026-01-10
**Topic:** Why RSI is set to 0x10000 in vm_init_regs()

---

## Question

In `vm_init_regs(vm_t *v)`, why do we set `regs.rsi` to `0x10000`? What's the purpose of the RSI register in KVM ecosystem? Why this specific value?

---

## Quick Answer (5 sentences)

**RSI is set to 0x10000 to point the Linux kernel to its boot parameters structure.** When the kernel starts executing at RIP=0x100000, it expects RSI to contain the memory address of `struct boot_params`. This follows the Linux x86-64 boot protocol specification. RSI is like a treasure map telling the kernel "hey, your configuration settings are stored at address 0x10000!" Without this pointer, the kernel wouldn't know where to find critical boot information like command line parameters, memory layout, and hardware details.

---

## The Metaphor 🎁

Imagine you're moving into a new house (the kernel starting up). The house is at address **0x100000** (RIP register). Before you arrive, someone places a **welcome package** at the mailbox address **0x10000** containing important info: WiFi password, thermostat instructions, emergency contacts.

**RSI is like a sticky note on your front door saying "Check mailbox at 0x10000 for your welcome package!"** Without that note, you'd wander around the house confused, not knowing where your setup instructions are.

---

## Technical Details

### Code Location
File: `src/arch/x86/vm.c`

Line 38 - Register initialization:
```c
regs.rip = 0x100000, regs.rsi = 0x10000;
```

Line 130 - Boot params structure placement:
```c
struct boot_params *boot =
    (struct boot_params *) ((uint8_t *) v->mem + 0x10000);
```

### Guest Memory Layout

```
0x10000   ┌─────────────────────────┐
          │  struct boot_params     │  ← RSI points here
          │  (kernel configuration) │
          └─────────────────────────┘
0x20000   ┌─────────────────────────┐
          │  Command line string    │
          └─────────────────────────┘
0x100000  ┌─────────────────────────┐
          │  Kernel code            │  ← RIP points here (execution starts)
          │                         │
          └─────────────────────────┘
```

### RSI's Role in x86-64 Architecture

1. **Calling Convention**: RSI is the **second parameter register** in System V AMD64 ABI (used by Linux)
2. **Boot Protocol**: Linux x86-64 boot protocol specifies that RSI must hold the **physical address** of `struct boot_params`
3. **Kernel Behavior**: When the kernel code at 0x100000 starts executing, it immediately reads RSI to locate its configuration data

### What's in boot_params?

The `struct boot_params` contains critical information the kernel needs to initialize:
- Command line arguments
- Memory map (E820 entries)
- Initrd/initramfs location and size
- Video mode information
- Setup header data
- Hardware configuration

### Why 0x10000 Specifically?

This address is part of the **Linux boot protocol specification**:
- Low enough to be in the first 1MB of memory (real mode accessible region)
- High enough to avoid BIOS data area and other reserved regions
- Well-documented and standardized location
- Follows the convention described in [Linux x86 Boot Protocol](https://www.kernel.org/doc/html/next/x86/boot.html)

### Complete Initialization Flow

1. **Host prepares guest memory**:
   - Places `boot_params` structure at 0x10000
   - Loads kernel image at 0x100000
   - Places command line at 0x20000

2. **vm_init_regs() configures vCPU**:
   - Sets RIP = 0x100000 (where to start executing)
   - Sets RSI = 0x10000 (where to find boot params)
   - Enables protected mode (CR0 |= 1)
   - Sets up flat memory model with segment registers

3. **Kernel starts execution**:
   - CPU jumps to 0x100000 (RIP)
   - Kernel reads RSI to find boot_params at 0x10000
   - Parses configuration and initializes subsystems

---

## Official Documentation Source

### Linux Kernel Documentation - Section 1.15

**Source:** [The Linux x86 Boot Protocol](https://www.kernel.org/doc/html/next/x86/boot.html)

**Section 1.15 - "THE 64-BIT KERNEL ENTRY"** states:

> "At entry, the CPU must be in 64-bit mode with paging enabled. The range with setup_header.init_size from start address of loaded kernel and zero page and command line buffer get ident mapping; a GDT must be loaded with the descriptors for selectors __BOOT_CS(0x10) and __BOOT_DS(0x18); both descriptors must be 4G flat segment; __BOOT_CS must have execute/read permission, and __BOOT_DS must have read/write permission; CS must be __BOOT_CS and DS, ES, SS must be __BOOT_DS; interrupt must be disabled; **%rsi must hold the base address of the struct boot_params.**"

**Key Requirement:**
```
%rsi must hold the base address of the struct boot_params
```

This is the **authoritative specification** from the Linux kernel documentation that mandates RSI register usage for boot parameter passing.

---

## Related References

- **Primary Source:** [Linux x86 Boot Protocol - Section 1.15](https://www.kernel.org/doc/html/next/x86/boot.html)
- Code: `src/arch/x86/vm.c:16-43` (vm_init_regs function)
- Code: `src/arch/x86/vm.c:127-144` (vm_arch_load_image function)
- Code Comment: `src/arch/x86/vm.c:134` (references the boot protocol documentation)
- Specification: System V AMD64 ABI (defines RSI as 2nd parameter register)

---

## Key Takeaways

✅ **RSI = pointer to boot configuration**
✅ **0x10000 = standardized location for boot_params**
✅ **Required by Linux x86-64 boot protocol**
✅ **Kernel depends on this to initialize correctly**
✅ **Without it, kernel boot would fail**
