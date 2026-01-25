## Overview

This document describes the clock gating implementation I did for the OSCAR parallelizing compiler (Owned by Kasahara Lab) on Intel Xeon Gold 6326 CPUs, including why my original approach failed and the working solution with nanosleep().

---

## The Problem We Were Trying to Solve

During parallel execution, some CPU cores wait idle while others finish their work (synchronization barriers). These idle cores waste power spinning in busy-wait loops:

```c
// Original OSCAR-generated busy-wait
while (sync_flag != expected) {
    #pragma omp flush
}
```

**Goal**: Put idle cores into low-power sleep states (C-states) during these waits.

---

## Original Approach: MWAIT (Failed)

### What I Tried

Intel provides `MONITOR`/`MWAIT` instructions specifically designed for power-efficient waiting:

```c
// Monitor an address for changes
__asm__ volatile ("monitor" : : "a"(addr), "c"(0), "d"(0));

// Wait until the monitored address changes (enters C-state)
__asm__ volatile ("mwait" : : "a"(hint), "c"(0));
```

### Why It Failed: "Illegal Instruction"

```
$ ./ep.A.8.clockgate
Illegal instruction (core dumped)
```

### Technical Reason: CPU Privilege Levels (Ring 0 vs Ring 3)

```
┌─────────────────────────────────────────────────────────┐
│  Ring 3 (User Space)  ←── Our program runs here         │
│    - Normal applications                                │
│    - Even with sudo (root = admin, still Ring 3)        │
│    - CANNOT execute MWAIT                               │
├─────────────────────────────────────────────────────────┤
│  Ring 0 (Kernel Space)                                  │
│    - Linux kernel                                       │
│    - Device drivers                                     │
│    - CAN execute MWAIT                                  │
└─────────────────────────────────────────────────────────┘
```

**An insight**: `sudo` gives you root *privileges* (admin rights), but you're still in Ring 3 (user space). MWAIT is a **privileged instruction** that can only execute in Ring 0 (kernel space).

On older Linux kernels (pre-2.6), MWAIT sometimes worked from user space. Modern kernels block this for security reasons.

---

## Governor Confusion

### What is a CPU Governor?

The governor controls how the kernel manages CPU frequency:

| Governor | Frequency Control | Turbo Boost | Use Case |
|----------|------------------|-------------|----------|
| `performance` | Kernel manages, max freq | Yes (3.5 GHz) | Best performance |
| `userspace` | Manual via sysfs | No (max 2.9 GHz) | Fine-grained control |
| `powersave` | Kernel manages, min freq | No | Battery saving |

### The Problem We Hit

Our original script used `userspace` governor for frequency changes:

```bash
sudo cpupower frequency-set -g userspace
sudo cpupower frequency-set -f 2900000  # Set to 2.9 GHz
```

**Result**: Baseline ran 3x slower (~69s instead of ~21s) because:
- `userspace` governor caps frequency at 2.9 GHz
- No turbo boost (which goes up to 3.5 GHz)

### Why We Thought We Needed Userspace Governor

The OSCAR `fvcontrol` pragma supports frequency scaling:

```c
#pragma oscar fvcontrol(pe, (FV_LOGIC, 50))  // Set CPU to 50% frequency
```

This requires writing to sysfs files:
```
/sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed
```

Which only works with `userspace` governor.

**But clock gating (rate -1) doesn't need frequency control** - it needs C-states, which work with ANY governor.

---

## New Approach: nanosleep() (Working so far...)

### How It Works

Instead of using MWAIT directly, we call `nanosleep()`:

```c
// Our code (Ring 3 - user space)
nanosleep(&ts, NULL);  // Sleep for 1 microsecond

// Internally, the kernel (Ring 0) does:
// 1. Marks thread as sleeping
// 2. Runs idle loop on that CPU
// 3. Idle loop uses MWAIT to enter C-state
// 4. CPU enters C1/C6 = power savings!
```

```
┌──────────────────────────────────────────────────────────┐
│  User Space (Ring 3)                                     │
│    nanosleep() ─────────────────┐                        │
│                                 │ syscall                │
├─────────────────────────────────┼────────────────────────┤
│  Kernel Space (Ring 0)          ▼                        │
│    schedule() → idle_loop() → MWAIT → C-state            │
└──────────────────────────────────────────────────────────┘
```

### Why nanosleep Works with Any Governor

- nanosleep is a **syscall** - it works regardless of governor
- The kernel manages C-states internally during idle
- We don't need manual frequency control for clock gating

### Implementation Details

OSCAR uses `-nostdinc` flag (no standard includes), so we can't use `<time.h>`. Solution: inline syscall assembly.

```c
static inline void oscar_nanosleep_syscall(long nsec) {
    long ts[2] = {0, nsec};  // struct timespec {sec, nsec}

    __asm__ volatile (
        "syscall"
        : /* no outputs */
        : "a"(35),      // syscall number: nanosleep = 35 on x86_64
          "D"(ts),      // rdi = &timespec
          "S"((long)0)  // rsi = NULL
        : "rcx", "r11", "memory"
    );
}
```

---

## C-States Explained

When a CPU core has nothing to do, it can enter progressively deeper sleep states:

| C-State | Name | Power Reduction | Wake-up Latency | What Happens |
|---------|------|-----------------|-----------------|--------------|
| C0 | Active | 0% | 0 | Running code |
| C1 | Halt | ~50% | ~1 µs | Clock stopped |
| C1E | Enhanced Halt | ~60% | ~10 µs | Clock + voltage reduced |
| C6 | Deep Sleep | ~90% | ~100 µs | Most circuits off |

### How to Verify Clock Gating Works

Check C-state residency before and after running:

```bash
# Read C-state times (in microseconds)
cat /sys/devices/system/cpu/cpu0/cpuidle/state*/name
cat /sys/devices/system/cpu/cpu0/cpuidle/state*/time
```

**Expected results**:
- Baseline (no clock gating): Minimal C-state increase during test
- Clock gated version: Significant C1/C6 increase = cores actually sleeping

---

## Summary: What Changed

| Aspect | Original (Broken) | New (Working) |
|--------|-------------------|---------------|
| Clock gating method | MWAIT instruction | nanosleep() syscall |
| Why it failed/works | Ring 3 can't execute MWAIT | Kernel handles MWAIT internally |
| Governor | `userspace` (no turbo) | `performance` (turbo enabled) |
| Frequency scaling | Manual sysfs writes | Not used (clock gating only) |
| Baseline speed | ~69s (slow, no turbo) | ~21s (fast, turbo enabled) |

---

## Files Modified

1. **`oscar_clockgate_runtime.h`** - Complete rewrite
   - Removed MWAIT-based code
   - Added nanosleep via inline syscall
   - Works with `-nostdinc` flag

2. **`Makefile.clockgate`** - Updated flags
   - Removed: `MWAIT_ENABLED`, `MWAIT_HINT`
   - Added: `SLEEP_NS`, `USE_YIELD`

3. **`full_eval_clockgate.sh`** - Updated evaluation script
   - Changed to `performance` governor
   - Added C-state monitoring
   - Removed MWAIT references

---

## Usage

```bash
# Run evaluation (example: Class A, 8 threads, 4 iterations, EP benchmark)
./full_eval_clockgate.sh A 8 4 EP

# Check results
cat results_clockgate_*.txt
```

---

## Configuration Options

In `oscar_clockgate_runtime.h`:

```c
#define OSCAR_SLEEP_NS 1000        // Sleep duration (1µs default)
#define OSCAR_USE_YIELD 0          // Use sched_yield instead of nanosleep
#define OSCAR_CLOCKGATE_DEBUG 0    // Enable debug output
#define OSCAR_NUM_CPUS 16          // Number of CPUs
```

---

## References

- Intel 64 and IA-32 Architectures Software Developer's Manual (MWAIT)
- Linux kernel CPU idle documentation
- ACPI C-states specification
