# Page Tables Project Report

## Overview
This project implements two main features for the xv6 operating system:
1.  **Page Table Visualization (`vmprint`)**: A utility to print the hierarchical page table structure of a process, aiding in debugging and understanding virtual memory mapping.
2.  **Page Access Detection (`pgaccess`)**: A system call that detects and reports which pages have been accessed by the user, useful for garbage collection algorithms.

## Feature 1: Print Page Table (`vmprint`)
-   **Functionality**: Recursively traverses the 3-level page table (L2, L1, L0) and prints valid Page Table Entries (PTEs) along with their physical addresses.
-   **Trigger**: Called in `exec.c` specifically for the first process (`init`, pid 1) to demonstrate functionality without spamming output for every process.
-   **Implementation**: `kernel/vm.c`

### Sequence Diagram
```mermaid
sequenceDiagram
    participant User as User Process (init)
    participant Exec as kernel/exec.c
    participant VM as kernel/vm.c (vmprint)
    
    User->>Exec: sys_exec()
    Exec->>Exec: Load program segments
    Exec->>Exec: Allocate/Map stack
    
    Note over Exec, VM: Just before return argc (for pid==1)
    Exec->>VM: vmprint(pagetable)
    
    loop Recursively Walk Page Table
        VM->>VM: vmprint_walk(pagetable, 0)
        VM-->>VM: Print L2 PTEs
        VM->>VM: vmprint_walk(child_pt, 1)
        VM-->>VM: Print L1 PTEs
        VM->>VM: vmprint_walk(child_pt, 2)
        VM-->>VM: Print L0 (Leaf) PTEs
    end
    
    Exec-->>User: returns argc
```

### Testing Command
Execute in terminal:
```bash
make clean
make qemu
```
**Observation**: You will see the page table printed (with `..` indentation) immediately after `xv6 kernel is booting`.


## Feature 2: Page Access Detection (`pgaccess`)
-   **Functionality**: Checks the "Accessed" bit (`PTE_A`) in the RISC-V PTEs for a specified range of user pages.
-   **Logic**:
    1.  Walks the page table for the given virtual addresses.
    2.  If `PTE_V` (Valid) and `PTE_A` (Accessed) are set, marks the corresponding bit in the result mask.
    3.  Clears the `PTE_A` bit to reset the "accessed" state for future checks.
    4.  Returns the bitmask to the user.
-   **Implementation**: `kernel/sysproc.c` (system call logic), `kernel/riscv.h` (bit definition).

### Sequence Diagram
```mermaid
sequenceDiagram
    participant Test as User Program (pgtbltest)
    participant Syscall as kernel/sysproc.c (sys_pgaccess)
    participant Walk as kernel/vm.c (walk)
    
    Test->>Syscall: pgaccess(base_va, len, &buffer)
    
    loop For each page in len
        Syscall->>Walk: walk(pagetable, va, 0)
        Walk-->>Syscall: return PTE pointer
        
        alt PTE Valid and Accessed (PTE_A set)
            Syscall->>Syscall: Set bit 'i' in result
            Syscall->>Syscall: Clear PTE_A bit in PTE
        end
    end
    
    Syscall->>Test: copyout result mask
    Syscall-->>Test: return 0 (Success)
```

### Testing Command
Inside xv6 shell, run:
```bash
pgtbltest
```
**Observation**:
```
pgaccess_test starting
pgaccess_test: OK
```

