### Section by Section Documentation

#### Header Inclusions:
```c
#include "defs.h"
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#include <sys/ipc.h>
#include <sys/msg.h>
#include <sys/sem.h>
#include <sys/shm.h>
#include <sys/types.h>
```
- **Purpose:** These headers include necessary libraries for the program’s functionality:
  - `stdio.h` and `stdlib.h` for standard I/O and memory allocation.
  - `string.h` for string manipulation functions.
  - `unistd.h` for system calls like `getopt()` and `kill()`.
  - `signal.h` for signal handling.
  - `sys/ipc.h`, `sys/msg.h`, `sys/sem.h`, `sys/shm.h`, and `sys/types.h` for inter-process communication (IPC), including shared memory, semaphores, and message queues.

---

#### Main Function Declaration:
```c
int main(int argc, char *argv[])
```
- **Purpose:** The main function starts here. It processes command-line arguments and manages inter-process communication to output the shared memory status.

---

#### Variable Initialization:
```c
int k_flag = 0;
int opt = 0;
int SharedMem_ID;
int Total_found = 0;
int Total_tested = 0;
int Total_skipped = 0;
int PerfectNumbersDiscoveredCounter = 0;
SharedMemoryStruct *SharedMem_Struct;
```
- **k_flag:** Used to track if the `-k` option (kill signal) is specified.
- **opt:** Stores the current option parsed by `getopt()`.
- **SharedMem_ID:** Stores the ID of the shared memory segment.
- **Total_found, Total_tested, Total_skipped:** Accumulators for the statistics.
- **PerfectNumbersDiscoveredCounter:** Tracks the count of perfect numbers discovered.
- **SharedMem_Struct:** Pointer to the shared memory structure.

---

#### Argument Parsing with `getopt()`:
```c
while ((opt = getopt(argc, argv, "k")) != -1) {
    switch (opt) {
        case 'k':
            k_flag = 1;
            break;
        default:
            fprintf(stderr, "SYNOPSIS: report -k\n");
            exit(1);
            break;
    }
}
```
- **Purpose:** This block processes command-line arguments using `getopt()`:
  - If the `-k` option is present, it sets `k_flag` to `1` (indicating the need to send a signal to terminate all processes).
  - If an unknown option is provided, the program prints a synopsis message and exits.

---

#### Shared Memory Setup:
```c
if ((SharedMem_ID = shmget(KEY, sizeof(SharedMemoryStruct), IPC_CREAT | 0660)) == -1 |
    (SharedMem_Struct = (SharedMemoryStruct *)shmat(SharedMem_ID, 0, 0)) == (SharedMemoryStruct *)-1) {

    fprintf(stderr, "ERROR: Failed to create or attach the shared memory segment\n");
    exit(1);
}
```
- **Purpose:** This section attaches the program to a shared memory segment:
  - `shmget()` creates or accesses a shared memory segment using `KEY` and assigns it to `SharedMem_ID`.
  - `shmat()` attaches the shared memory to the process and casts it to the custom structure `SharedMemoryStruct`.
  - If either call fails, an error message is printed, and the program exits.

---

#### Perfect Number Reporting:
```c
fprintf(stdout, "Perfect Number Found: ");

for (int i = 0; SharedMem_Struct->PerfectNumber[i] != 0; i++) {
    fprintf(stdout, "%ld ", SharedMem_Struct->PerfectNumber[i]);
}

fprintf(stdout, "\n");
```
- **Purpose:** This loop prints all the perfect numbers stored in the shared memory segment.
  - It iterates through the `PerfectNumber` array in the shared memory, printing each non-zero value.

---

#### Process Statistics Reporting:
```c
for (int i = 0; i < MAX_PROCESSES; i++) {

    if (SharedMem_Struct->PerfectNumber[i] != 0) {
        PerfectNumbersDiscoveredCounter++;
    }

    if (SharedMem_Struct->ProcessessStruct[i].pid != 0) {
        fprintf(stdout, "pid(%d): ", SharedMem_Struct->ProcessessStruct[i].pid);
        Total_found += SharedMem_Struct->ProcessessStruct[i].FoundCounter;
        fprintf(stdout, "found: %d, ", SharedMem_Struct->ProcessessStruct[i].FoundCounter);
        Total_tested += SharedMem_Struct->ProcessessStruct[i].TestedCounter;
        fprintf(stdout, "tested: %d, ", SharedMem_Struct->ProcessessStruct[i].TestedCounter);
        Total_skipped += SharedMem_Struct->ProcessessStruct[i].SkippedCounter;
        fprintf(stdout, "skipped: %d\n", SharedMem_Struct->ProcessessStruct[i].SkippedCounter);
    }
}
```
- **Purpose:** This loop iterates through the `ProcessessStruct` array to print statistics for each active process:
  - **Found Counter:** Number of perfect numbers found by the process.
  - **Tested Counter:** Number of numbers tested by the process.
  - **Skipped Counter:** Number of skipped numbers.
  - It accumulates totals for all counters and prints them.

---

#### Total Statistics Output:
```c
fprintf(stdout, "Statistics:\n");
fprintf(stdout, "Total found: %d\n", Total_found);
fprintf(stdout, "Total tested: %d\n", Total_tested);
fprintf(stdout, "Total skipped: %d\n", Total_skipped);
```
- **Purpose:** After processing all processes, the program prints the cumulative statistics for:
  - Total numbers found.
  - Total numbers tested.
  - Total numbers skipped.

---

#### Optional Kill Signal (`-k` option):
```c
if (k_flag == 1) {
    kill(SharedMem_Struct->manage_PID, SIGHUP);
}
```
- **Purpose:** If the `-k` option was specified, this section sends a `SIGHUP` signal to the process managing the shared memory (`manage_PID`), instructing it to terminate gracefully.

---

#### Program Exit:
```c
return 0;
```
- **Purpose:** The program terminates successfully after completing its operations.

---

### Overall Purpose:
This program is designed to report statistics on perfect numbers and process activity stored in a shared memory segment. If the `-k` option is provided, it sends a signal to the process managing the shared memory, requesting termination.
