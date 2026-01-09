# CPU Architecture Initialization in vm_init

**Date**: 2026-01-09
**Topic**: What CPU architecture is initialized during vm_init?

## Question
What does cpu arch is initialized during vm_init?

## Answer Summary (5 sentences)

During `vm_init`, the code initializes the CPU architecture by calling `vm_arch_cpu_init()`. **For x86-64 systems**, it sets up protected mode with flat memory segments, configures CPUID to advertise KVM features, and sets MSRs for fast string operations. **For ARM64 systems**, it asks KVM what CPU type the host prefers (like Cortex-A series), then initializes the virtual CPU with those settings. Think of it like setting up a puppet theater - x86 needs detailed costume instructions (registers, segments), while ARM just picks the best puppet model and lets it go. The architecture is auto-detected at compile time using `#if` preprocessor checks.

## The Metaphor: Building a Robot Actor

Imagine you're building a robot actor for a play:

- **x86-64 way**: You manually dress the robot - put on the hat (segment registers), set the walking style to "modern mode" (protected mode with CR0), teach it specific tricks (CPUID features), and give it speed settings (MSRs for fast operations). See `src/arch/x86/vm.c:108-114`

- **ARM64 way**: You ask the factory "what's your best robot model?" (KVM_ARM_PREFERRED_TARGET), and the factory says "use this one!" Then you just turn it on with those settings (KVM_ARM_VCPU_INIT). See `src/arch/arm64/vm.c:89-99`

## Code Flow

### Main initialization in src/vm.c

The `vm_init` function calls architecture-specific initialization:

```c
// Line 24-25
if (vm_arch_init(v) < 0)
    return -1;

// Line 45-46
if (vm_arch_cpu_init(v) < 0)
    return -1;
```

### x86-64 Implementation (src/arch/x86/vm.c)

```c
int vm_arch_cpu_init(vm_t *v)
{
    vm_init_regs(v);      // Set registers, enable protected mode
    vm_init_cpu_id(v);    // Configure CPUID (CPU capabilities)
    vm_init_msrs(v);      // Set MSRs (performance features)
    return 0;
}
```

Key x86-64 initialization steps:
1. **vm_init_regs** (line 16-43): Sets up segment registers (CS, DS, FS, GS, ES, SS) with flat memory model, enables protected mode in CR0, sets instruction pointer to 0x100000
2. **vm_init_cpu_id** (line 46-65): Configures CPUID to advertise KVM paravirtualization features with signature "KVMKVMKVM"
3. **vm_init_msrs** (line 74-87): Sets Model-Specific Registers for performance optimizations like fast string operations

### ARM64 Implementation (src/arch/arm64/vm.c)

```c
int vm_arch_cpu_init(vm_t *v)
{
    struct kvm_vcpu_init vcpu_init;
    if (ioctl(v->vm_fd, KVM_ARM_PREFERRED_TARGET, &vcpu_init) < 0)
        return throw_err("Failed to find perferred CPU type\n");

    if (ioctl(v->vcpu_fd, KVM_ARM_VCPU_INIT, &vcpu_init))
        return throw_err("Failed to initialize vCPU\n");

    return 0;
}
```

Key ARM64 initialization steps:
1. **Query preferred CPU type**: Uses `KVM_ARM_PREFERRED_TARGET` ioctl to ask KVM what CPU model works best on this host
2. **Initialize vCPU**: Uses `KVM_ARM_VCPU_INIT` ioctl to configure the virtual CPU with the recommended settings

## Key Differences

| Aspect | x86-64 | ARM64 |
|--------|--------|-------|
| Approach | Manual configuration | Host-recommended configuration |
| Registers | Explicitly set flat segments, protected mode | Let KVM set based on preferred target |
| CPUID/Features | Manually configure CPUID entries | Automatically configured by KVM |
| Complexity | More detailed setup required | Simpler, delegation to KVM |

## Architecture Detection

The codebase uses preprocessor directives to select the correct implementation at compile time:

```c
// In src/arch/x86/vm.c
#if !defined(__x86_64__) || !defined(__linux__)
#error "This implementation is dedicated to Linux/x86-64."
#endif

// In src/arch/arm64/vm.c
#if !defined(__aarch64__) || !defined(__linux__)
#error "This implementation is dedicated to Linux/aarch64."
#endif
```

This ensures you get the right initialization code for your CPU architecture!
