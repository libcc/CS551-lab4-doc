# `hw4.h` - Header File Documentation

## Overview

The `hw4.h` header file defines the constants, data structures, and function prototypes necessary for the operation of a system that finds perfect numbers using multiple processes. The file is essential for inter-process communication and shared memory management between the processes involved in the system, including `manage.c`, `compute.c`, and `report.c`. It includes key definitions for shared memory, message queues, and flags used to signal between processes.

The file contains:
1. Library imports required for system calls and data management.
2. Constant definitions for shared memory, message queue keys, permissions, and other system settings.
3. Data structures to store process statistics, computation results, and inter-process messages.

## Library Imports

The following libraries are imported for use in the program:

- `<stdio.h>`: Standard input/output functions such as `printf()`, `scanf()`, etc.
- `<stdlib.h>`: Standard utility functions, including memory allocation and process control.
- `<sys/shm.h>`: Defines the shared memory functions (`shmget()`, `shmat()`, `shmdt()`, `shmctl()`).
- `<sys/msg.h>`: Defines the message queue functions (`msgget()`, `msgsnd()`, `msgrcv()`, `msgctl()`).
- `<string.h>`: Provides string manipulation functions (`strcmp()`, `strerror()`, etc.).
- `<unistd.h>`: Contains standard symbolic constants and types, and declarations for functions like `fork()` and `exec()`.
- `<signal.h>`: Declares functions for handling signals, such as `kill()`.
- `<stdbool.h>`: Provides support for boolean data types (`true` and `false`).
- `<errno.h>`: Defines macros for reporting and retrieving error codes (e.g., `errno`).

## Constant Definitions

### Message Queue and Shared Memory Keys

- `MQ_KEY`: The key used to create or access the message queue. This key must be consistent across processes that communicate via this queue.
- `SM_KEY`: The key used to create or access the shared memory segment. This key ensures all processes access the same shared memory.
- `PERM`: Permissions used for both shared memory and message queue, set to `0666`, which gives read and write permissions to the owner, group, and others.

### Message Flags

Flags are used to indicate the type of message being sent between processes via the message queue.

- `INIT_FLAG`: Signals a process initialization.
- `KILL_FLAG`: Signals a termination request for a process.
- `FOUND_FLAG`: Indicates that a perfect number has been found and should be communicated.
- `INIT_SUCCESS_FLAG`: Confirms the successful initialization of a process.

### Memory and Process Constraints

- `RANGE`: Defines the upper limit of the range within which perfect numbers are computed (set to \(2^{25} = 33,554,432\)).
- `SEG_NUM`: Defines the number of segments in the bitmap (set to 32).
- `INT_PER_SEG`: Defines the number of integers in each segment (set to 32,768). This value, combined with the number of segments and the size of an integer, determines the total size of the bitmap.
  - \( \text{INT\_PER\_SEG} \times \text{SEG\_NUM} \times 8 \times \text{sizeof(int)} = \text{RANGE} \).
- `MAX_PERFECT`: Specifies the maximum number of perfect numbers that can be stored in the `result` array (set to 20).
- `MAX_PROCESS`: Defines the maximum number of processes that can be tracked in the `process` array (set to 20).

## Data Structures

### `stats`

This structure stores statistics for individual processes involved in finding perfect numbers.

- `pid`: Process ID of the compute process.
- `found`: The number of perfect numbers found by the process.
- `tested`: The number of numbers tested by the process.
- `skipped`: The number of numbers skipped by the process.

This structure is used to track and report each process's performance, including how many numbers were tested, how many were skipped, and how many perfect numbers were found.

### `segment`

This structure represents the shared memory segment used by all processes to store and share data.

- `bitMap[SEG_NUM][INT_PER_SEG]`: A 2D array used to store a bitmap, where each bit represents whether a number in the range has been tested or not. It can represent up to \(2^{25}\) bits (approximately 33.5 million bits).
- `result[MAX_PERFECT]`: An array that stores the perfect numbers found by all compute processes. The array can store up to 20 perfect numbers.
- `process[MAX_PROCESS]`: An array of `stats` structures, each representing the statistics of an active process. The array can store data for up to 20 processes.

This structure is central to the system's operation, allowing multiple processes to share the state of the computation and their progress through shared memory.

### `msg`

This structure defines the format of messages passed between processes via the message queue.

- `flag`: A long integer that stores the type of message being sent (e.g., `INIT_FLAG`, `KILL_FLAG`, `FOUND_FLAG`, `INIT_SUCCESS_FLAG`).
- `data`: An integer that carries additional data related to the message (e.g., the process ID of the sender or a perfect number that was found).

Messages are used to signal various events between the control and compute processes, such as starting a process, terminating a process, or reporting a found perfect number.

## External Variables

- `extern int errno`: Declares `errno`, an external integer variable that stores error codes produced by system calls and library functions. It is used throughout the program to print error messages when a system call fails (e.g., if shared memory cannot be accessed, or if sending a message fails).

## Usage Example

This header file would typically be included in multiple C source files that manage the processes, including:

```c
#include "hw4.h"
```

For example, the `manage.c` file might include logic to set up shared memory and the message queue, while the `compute.c` file handles the computation of perfect numbers and communicates progress via messages.

### Memory Allocation Example

In `report.c`, shared memory is accessed and mapped to a `segment` structure:

```c
smid = shmget(SM_KEY, sizeof(segment), 0);
memory = shmat(smid, NULL, 0);
```

### Message Passing Example

A process might send a message using the message queue to report that it has found a perfect number:

```c
msg message;
message.flag = FOUND_FLAG;
message.data = perfect_number;
msgsnd(mqid, &message, sizeof(message.data), 0);
```
