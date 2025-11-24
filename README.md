# xv6 Process State System Call Implementation

## Overview
This project implements a new system call `pstate` in the xv6 operating system that displays the state of current user processes, similar to the Unix `ps` command. The implementation includes kernel-level system call handling and user-level programs for testing.

## Features
- **Process State Display**: Shows process ID, name, state, and parent name
- **CPU Status Monitoring**: Displays CPU utilization (idle or running process)
- **Process Filtering**: Only shows processes in SLEEPING, RUNNING, or RUNNABLE states
- **Total Process Count**: Provides count of active processes
- **Testing Utilities**: Includes `pi.c` CPU-intensive program for testing

## Requirements
- Implement `pstate` system call without unnecessary xv6 modifications
- Do not modify `sh.c`
- Display process information in specified format
- Show CPU status for all processors
- Include testing programs like `pi.c`

## System Architecture

### Files Modified
- **Kernel Level**:
  - `kernel/proc.c` - Main `pstate` system call implementation
  - `kernel/syscall.c` - System call table registration
  - `kernel/syscall.h` - System call number definition
  - `kernel/defs.h` - Function prototype declaration

- **User Level**:
  - `user/user.h` - User-space system call declaration
  - `user/usys.pl` - System call stub generation
  - `user/pstate.c` - User program to invoke system call
  - `user/pi.c` - CPU-intensive test program

## Implementation Details

### System Call Implementation (`kernel/proc.c`)
The core `sys_pstate()` function:
- Iterates through all processes in the process table
- Filters processes in SLEEPING, RUNNING, or RUNNABLE states
- Safely accesses process data using proper locking
- Handles special case for init process parent name
- Displays CPU status for all processors

### Key Features
- **Process State Mapping**: Converts internal state codes to human-readable strings
- **Parent Process Resolution**: Safely retrieves parent process names
- **CPU Utilization Tracking**: Monitors which process each CPU is executing
- **Thread-Safe Operations**: Uses proper locking to prevent race conditions

## Installation & Usage

### 1. Add Files to xv6
Copy all modified and new files to your xv6 source directory and update the Makefile:

```makefile
# In UPROGS add:
_pstate\
_pi\
```

### 2. Compile and Run
```bash
make
make qemu
```

### 3. Usage Examples
```bash
pstate
pi &
pstate
```

### 4. Sample Output
```
$ pstate
pid     state           name    parent
1       RUNNABLE        init    (init)
2       SLEEPING        sh      1
3       RUNNING         pi      2
total: 3
cpu 0: running process 3
cpu 1: idle
cpu 2: idle
cpu 3: idle
```

## Code Structure

### System Call Handler (`kernel/proc.c`)
```c
uint64 sys_pstate(void) {
  struct proc *p;
  int total_processes = 0;

  printf("pid\tstate\t\tname\tparent\n");
  
  for(p = proc; p < &proc[NPROC]; p++){
    acquire(&p->lock);
    if(p->state == SLEEPING || p->state == RUNNING || p->state == RUNNABLE) {
      total_processes++;
      char *parent_name = "???";
      
      if (p->pid == 1) {
        parent_name = "(init)";
      } else if (p->parent) {
        parent_name = p->parent->name;
      }
      
      printf("%d\t%s\t\t%s\t%s\n", p->pid, pstate_states[p->state], p->name, parent_name);
    }
    release(&p->lock);
  }
  
  printf("total: %d\n", total_processes);
  
  for (int i = 0; i < NCPU; i++) {
    struct proc *proc_on_cpu = cpus[i].proc;
    if(proc_on_cpu){
      printf("cpu %d: running process %d\n", i, proc_on_cpu->pid);
    } else {
      printf("cpu %d: idle\n", i);
    }
  }
  
  return 0;
}
```

### Test Program (`user/pi.c`)
```c
#include "kernel/types.h"
#include "user/user.h"

int main(int argc, char *argv[]) {
    volatile int i;
    for (i = 0; i < 1000000000; i++) {
        // CPU-intensive loop for testing
    }
    exit(0);
}
```

### User Interface (`user/pstate.c`)
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(void) {
    pstate();
    exit(0);
}
```

## System Call Registration

### 1. System Call Number (`kernel/syscall.h`)
```c
#define SYS_pstate  22
```

### 2. System Call Table (`kernel/syscall.c`)
```c
extern uint64 sys_pstate(void);
static uint64 (*syscalls[])(void) = {
    // ... other syscalls
    [SYS_pstate]  sys_pstate,
};
```

### 3. User Space Declaration (`user/user.h`)
```c
int pstate(void);
```

### 4. Stub Generation (`user/usys.pl`)
```c
entry("pstate");
```

## Technical Details

### Process States Displayed
- **SLEEPING**: Process waiting for an event
- **RUNNABLE**: Process ready to run but waiting for CPU
- **RUNNING**: Process currently executing on a CPU

### Process Information
- **Process ID**: Unique identifier for each process
- **Process Name**: Executable name or process label
- **Process State**: Current execution state
- **Parent Name**: Name of the parent process

### CPU Status
- **Running**: CPU is executing a specific process
- **Idle**: CPU is not currently running any process

## Testing Strategy

### 1. Basic Functionality
```bash
pstate
```
Verify init process and shell are displayed correctly.

### 2. Process Creation
```bash
pi &
pstate
```
Check if background process appears in process list.

### 3. Multiple Processes
```bash
pi &
pi &
pstate
```
Verify multiple processes and CPU status updates.

### 4. Process States
Monitor state transitions between RUNNING, RUNNABLE, and SLEEPING.

## Files Included

### Modified Files
- `kernel/proc.c` - System call implementation
- `kernel/syscall.c` - System call table
- `kernel/syscall.h` - System call numbers
- `kernel/defs.h` - Function prototypes
- `user/user.h` - User declarations
- `user/usys.pl` - System call stubs

### New Files
- `user/pstate.c` - User interface program
- `user/pi.c` - CPU-intensive test program

## Limitations
- Only displays processes in SLEEPING, RUNNING, or RUNNABLE states
- No command-line arguments for filtering or formatting
- Process information is a snapshot (may change during display)
- Limited to xv6's process table size (NPROC)

## Performance Considerations
- Uses process locking to ensure data consistency
- Minimal performance impact on system operation
- Efficient process table iteration
- Clean output formatting for readability

## Notes
- The implementation maintains xv6 coding conventions
- No unnecessary modifications to existing xv6 functionality
- Proper error handling and resource management
- Thread-safe operations throughout
