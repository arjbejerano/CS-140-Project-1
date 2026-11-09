Members:
202X-XXXXX  THY/XXXX    Bejerano, Arjay
202X-XXXXX  THY/FVW3    Esma, Georiza
202X-XXXXX  THY/FVW3    Patiño, Josh Rylie
202X-XXXXX  THY/XXXX    Rimo, Ace Zeus
202X-XXXXX  THY/FVW3    Tibay, John Michael

# Members
| Student Number | Name                | Section  |
|----------------|---------------------|----------|
| 202X-XXXXX     | Bejerano, Arjay     | THY/XXXX |
| 202X-XXXXX     | Esma, Georiza       | THY/FVW3 |
| 202X-XXXXX     | Patiño, Josh Rylie  | THY/FVW3 |
| 202X-XXXXX     | Rimo, Ace Zeus      | THY/XXXX |
| 202X-XXXXX     | Tibay, John Michael | THY/FVW3 |


How dhouls we document?
> By commit/Step?
    > By file?

> By todo list?
    - this for now

> as a whole?

TODO list
- [/] Add `SCHEDULING_POLICY` as make parameter
- [/] Add Constants in `stcf.h`
Phase 1
- [/] Add 2.2 attributes to struct
- [ ] `schedlog` implementation
- [ ] `setburst` implementation
- [ ] Decrement burst time of running process whenever a timer interrupt happens
- [ ] Set burst time correctly when a process is first created
- [ ] Modify scheduler to make it choose the process with the least total(which is also remaing) burst
- [ ] Somehow exempt the kernel/scheduler from the above rule
- [ ] test files
    - [ ] `sanity_check.{sh,py}` build incrementally?

# Phase 1
For now, STCF scheduler chooses the process with the shortest remaining  burst time left. This is implemented by linear searching the process table `proc[]` using a `for` loop.

## Necessary Constants
- `stcf.h` added
  - Defaults and constants

```c
#define DEFAULT_STCF_BURST 10
#define STCF_AT 10
#define STCF_AD 10
#define N_GROUPS 10
#define DEFAULT_SHARES 10
```
- make parameter SCHEDULING_POLICY
```c
#ifndef SCHEDULING_POLICY
#define SCHEDULING_POLICY "DEFAULT"
#endif
```
- to add a default value


## Necessary process fields for STCF scheduler
- The tick when the process was created
  - defined at the end of `allocproc`
- the total number of ticks that the process will run on the CPU for (a.k.a. total burst time)
  - defined at the end of `allocproc`
  - set as default
  - can be reset using burst time function
- how many CPU ticks the processor has remaining (a.k.a. remaining burst time)
  - defined at the end of `allocproc` and set initially to be equal to total burst time
  - currently decremented in `kernel/trap.c` whenever a timer interrupt occurs
    ```c
  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    if(--myproc()->burst_remaining == 0){
      yield();
    ```

- how many CPU ticks the process spent waiting to be scheduled (i.e. be in the `RUNNABLE` state)
  - defined at the end of `allocproc` and set initially as 0
  - updated in scheduler
  - for every context switch, a process time run is recorded and then added to other process waiting time by traversing linearly
  - ```c
        uint selected_start = ticks;
        swtch(&c->context, &selected->context);
        uint selected_time = ticks - selected_start;
  ```


## Linear search as Process-to-run selection process
Simply linear search
The schedular function was actually just changed by reusing the original for loop traversal as the linear search, and then moving the context switching part at the end of the function by making an else block

```c
        if(p->burst_remaining < shortest_time){
          selected = p;
          shortest_time = p->burst_remaining;
        }
```


## setburst function
syscall defined in sysproc.c
calling this function sets burst time of caller process?

```c
    myproc()->burst_total = burst_ticks; 
    myproc()->burst_remaining = burst_ticks;

    yield();

```

### Killing a process
This is the same to that one lab exercise. In scheduler, mark killed any process that has zero burst time remaining. They run again since they have the least time, but they are immediately killed because it is marked `killed#1`
```c
      if(p->state == RUNNABLE) {
        found = 1;
        if (p->burst_remaining == 0)
          p->killed = 1;
```




## schedlog function
This basically prints scheduler process table info and has a time limit where it only lasts some given ticks amount of time, i.e. limited active time

kapoy

something something
have a logs container thingy like 150

during context switch print these logs


# Setup
## Add `SCHEDULING_POLICY` as make parameter
John adds this


## Add Constants in `stcf.h`
John adds this

# Phase 1
## Add 2.2 attributes to struct
John adds this

`main` contains the base kernel and user programs. `phase1` adds:

- per-process burst tracking fields
- default burst initialization for new processes
- a custom `schedulerSTCF()` that chooses the runnable process with the smallest remaining burst time
- timer-interrupt decrement of remaining burst time
- `setburst` and `schedlog` system calls
- user programs for testing burst behavior and scheduling logs
- documentation for the phase1 implementation

## Code-level changes and what each one does

### 1) Build configuration and scheduler policy selection

In `Makefile`, phase1 adds the compile-time switch:

```make
CFLAGS += -DSCHEDULING_POLICY='"$(SCHEDULING_POLICY)"'
```

and:

```make
ifndef SCHEDULING_POLICY
SCHEDULING_POLICY := DEFAULT
endif
```

This allows the kernel to compile with a chosen scheduling policy. Later, the scheduler checks whether the policy string is `STCF` before running the custom scheduler.
### 2) STCF constants

Added in `kernel/stcf.h`:

```c
#define DEFAULT_STCF_BURST 10
#define STCF_AT 10
#define STCF_AD 10
#define N_GROUPS 10
#define DEFAULT_SHARES 10

#ifndef SCHEDULING_POLICY
#define SCHEDULING_POLICY "DEFAULT"
#endif
```

These constants define default values used by the STCF scheduler, including the default burst size for a process and the policy name.
### 3) Process fields for burst scheduling

In `kernel/proc.h`, phase1 adds the fields:

```c
// for Project 1
uint tick_start;            // The tick this process was created
uint burst_total;           // Total burst time
uint burst_remaining;       // Remaining burst time
uint ticks_waiting;         // Time this process spent in runnable (so far?)
```

These fields are required for STCF scheduling:

- `burst_total` stores the process's assigned CPU runtime
- `burst_remaining` is decremented every timer tick
- `tick_start` is used to break ties between equal bursts
- `ticks_waiting` tracks waiting time for scheduler accounting
### 4) Initializing new processes with a burst

In `kernel/proc.c`, `allocproc()` now includes:

```c
p->burst_total = DEFAULT_STCF_BURST;
p->burst_remaining = DEFAULT_STCF_BURST;
p->tick_start = ticks;
p->ticks_waiting = 0;
```

This means every newly created process starts with a default burst time and properly initialized timing metadata.
### 5) Custom STCF scheduler

`kernel/proc.c` adds a new scheduler function, `schedulerSTCF()`. The important part is the linear scan of the process table:

```c
struct proc *selected = p;
uint shortest_time = UINT_MAX;
uint selected_start = 0;
int selected_pid = 0;

for(p = proc; p < &proc[NPROC]; p++) {
  acquire(&p->lock);

  if(p->state == RUNNABLE) {
    found = 1;
    if (p->burst_remaining == 0)
      p->killed = 1;

    if(p->burst_remaining < shortest_time || 
      (p->burst_remaining == shortest_time && 
        (p->tick_start < selected_start || 
          (p->tick_start == selected_start && p->pid < selected_pid) ) ) ){
      selected = p;
      shortest_time = p->burst_remaining;
      selected_start = p->tick_start;
      selected_pid = p->pid;
    }
  }
  release(&p->lock);
}
```

This chooses the runnable process with the least remaining burst. If two processes have equal remaining burst, the older one (smaller `tick_start`) is preferred, and then the lower PID is used as a final tie-breaker.

Once picked, the scheduler switches to that process:

```c
selected->state = RUNNING;
c->proc = selected;
swtch(&c->context, &selected->context);
```

This completely changes the selection policy away from the default scheduler behavior.
### 6) Decrementing burst time on timer interrupt
+
In `kernel/trap.c`, phase1 modifies the trap logic to reduce the current process's remaining burst whenever a timer interrupt occurs:
+
```c
if(which_dev == 2)
  if(--myproc()->burst_remaining == 0){
    yield();
  }
```
+
This is the key runtime mechanism behind STCF. Each tick removes one unit of CPU time from the active process. When the remaining burst reaches zero, the process yields so the scheduler can select the next runnable job.
### 7) Scheduler waiting-time tracking
+
In `kernel/proc.c`, after a process runs and yields back to the scheduler, phase1 updates waiting time for other runnable processes:
+
```c
uint selected_time = ticks - selected_start;

for(pi = proc; pi < &proc[NPROC]; pi++) {
  if(pi->state == RUNNABLE && pi->pid != selected->pid) {
    pi->ticks_waiting += selected_time;
  }
}
```
+
This measures how long the selected process ran and adds that runtime to the waiting totals of other runnable processes.
### 8) New system calls: `schedlog` and `setburst`
+
In `kernel/sysproc.c`, phase1 adds:
+
#### `schedlog`
+
```c
uint64
sys_schedlog(void)
{
  int ticks_active;
  argint(0, &ticks_active);
  if (ticks_active <= 0)
    return -1;

  acquire(&tickslock);
  schedlog_start = ticks;
  schedlog_duration = ticks_active;
  release(&tickslock);

  return 0;
}
```

This starts a logging window. During the interval, the scheduler can print process state and burst information.

#### `setburst`

```c
uint64
sys_setburst(void)
{
  int burst_ticks;
  argint(0, &burst_ticks);

  if(burst_ticks <= 0){
    myproc()->killed = 1;
    return -1;
  } else {
    myproc()->burst_total = burst_ticks;
    myproc()->burst_remaining = burst_ticks;

    yield();
  };

  return 0;
}
```

This allows a user process to set its own burst duration and immediately yields so the scheduler can rebalance the ready queue based on the new values.
### 9) System call declarations and registration

The syscall API additions are wired through `kernel/syscall.c`, `kernel/syscall.h`, and `kernel/defs.h` so user programs can call the new functions safely.

This is necessary because without the syscall registration, user programs could not call `schedlog()` or `setburst()`.
### 10) Added user programs for testing

The phase1 branch also adds several testing programs in the `user/` folder:

- `a.c`
- `b.c`
- `selected.c`
- `schedlogTest.c`
- `setburstTest.c`

Examples:

`user/a.c`:

```c
setburst(10);
while (1) {
  printf("a");
}
```

`user/b.c`:

```c
setburst(9);
while (1) {
  printf("b");
}
```

`user/selected.c`:

```c
schedlog(20);
for (int i = 0; i < 3; i++) {
  if (fork() == 0) {
    char *argv[] = {"a", 0};
    exec("a", argv);
  }
}
```

These programs are designed to exercise STCF scheduling behavior, verify burst assignment, and confirm the scheduler logs correctly.


It documents the STCF implementation, the new process fields, the burst and scheduler logic, and how the new system calls fit into the kernel.

phase1` is essentially a custom scheduling implementation layered onto the base xv6 kernel. It adds the pieces needed for a shortest-remaining-burst-time scheduler:

- burst metadata in the process table
- timer-based burst decrement
- a scheduler that chooses the smallest remaining burst
- user-level burst control and logging
- testing programs to validate the behavior


