Here is a detailed section-by-section documentation for the provided program:

---

### **Includes and Global Declarations**
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
- **Purpose**: This section includes necessary system libraries and headers for performing inter-process communication (IPC), synchronization, and signal handling. These libraries provide access to low-level system operations:
  - `stdio.h`: Standard input/output functions (e.g., `printf`, `fprintf`).
  - `stdlib.h`: General utilities, including dynamic memory management and process control.
  - `string.h`: String manipulation functions.
  - `unistd.h`: Provides access to the POSIX API, such as file operations and process control.
  - `signal.h`: Signal handling functions for asynchronous event processing.
  - `sys/ipc.h`, `sys/msg.h`, `sys/sem.h`, `sys/shm.h`, `sys/types.h`: System headers to support IPC mechanisms like message queues, semaphores, and shared memory.
  
- **Global Variables**:
  - `Index`: This variable stores the index of the current process in the shared memory's process structure.
  - `MsgQ_ID`: Message queue identifier used for communication between processes.
  - `Semaphore_ID`: Semaphore identifier used for controlling access to shared resources.
  - `UNLOCK_SEM`, `LOCK_SEM`: Constants defining semaphore operations for unlocking and locking respectively.
  - `TERMINATED_OPT`: A flag that determines if the process should terminate gracefully when signaled.
  
- **Data Structures**:
  - `messageQueueStruct`: A structure for message queue communication. Contains:
    - `type`: Indicates the type of message (e.g., 1 for PID, 2 for a perfect number, 3 for termination).
    - `data`: Holds the actual data to be transmitted, such as a PID or a number.
  - `sharedMemoryInfoStruct`: A pointer to a `SharedMemoryStruct` which represents the shared memory used for inter-process communication (defined elsewhere in the `defs.h` file).

---

### **Semaphore_Operation Function**
```c
void Semaphore_Operation(int Semaphore_ID, int operation)
```
- **Purpose**: This function performs semaphore operations like locking or unlocking, which are essential for coordinating access to shared resources (e.g., shared memory) among multiple processes. Semaphores ensure that only one process can modify shared data at a time, preventing race conditions.
  
- **Parameters**:
  - `Semaphore_ID`: The unique identifier of the semaphore set used for synchronization.
  - `operation`: Specifies the semaphore operation, either `LOCK_SEM` (-1) for locking or `UNLOCK_SEM` (1) for unlocking.
  
- **Details**:
  - A `sembuf` structure is used to define the semaphore operation. The operation is passed through `semop`, a system call that performs atomic changes on the semaphore set.
  - If the semaphore operation fails (returns -1), an error message is printed, and the program exits. This ensures proper error handling for critical synchronization tasks.
  - The operation's name (lock/unlock) is stored in `operationName` for more informative error messages.

---

### **Semaphore_Lock & Semaphore_Unlock Functions**
```c
void Semaphore_Lock(int Semaphore_ID)
void Semaphore_Unlock(int Semaphore_ID)
```
- **Purpose**: These are wrapper functions that simplify semaphore locking and unlocking. They call `Semaphore_Operation` with predefined constants (`LOCK_SEM` or `UNLOCK_SEM`).
  
- **Details**:
  - `Semaphore_Lock` calls `Semaphore_Operation` with `LOCK_SEM`, ensuring exclusive access to the shared resource.
  - `Semaphore_Unlock` calls `Semaphore_Operation` with `UNLOCK_SEM`, allowing other processes to access the shared resource.
  - By encapsulating semaphore operations in these functions, the code is more readable and easier to maintain.

---

### **C_Signal_Handler Function**
```c
void C_Signal_Handler(int signalNumber)
```
- **Purpose**: Handles termination signals (`SIGINT`, `SIGHUP`, `SIGQUIT`) that may be sent to the process. This function ensures that the process can terminate cleanly and that shared resources (e.g., shared memory and semaphore) are properly released or updated.

- **Details**:
  - **Signal Handling**: Upon receiving a signal, the function first ignores further occurrences of the signal using `signal(signalNumber, SIG_IGN)` to prevent recursive signal handling.
  - **Termination Message**: If `TERMINATED_OPT` is set and the process has not yet terminated, the handler sends a termination message (message type 3) with the current process's index (`Index`) to the manager through the message queue.
  - **Shared Memory Cleanup**: If the process PID matches the entry in the shared memory's process structure, it resets its counters (`FoundCounter`, `SkippedCounter`, `TestedCounter`) and PID to indicate that the process is no longer active.
  - **Error Handling**: If the PID does not match, an error is printed, indicating a mismatch in the process index.
  - The function calls `exit(0)` to terminate the process gracefully.

---

### **Find_PerfectNums Function**
```c
int Find_PerfectNums(long start, int MsgQ_ID)
```
- **Purpose**: This function finds and reports perfect numbers starting from a given number (`start`). A perfect number is a number equal to the sum of its proper divisors (excluding itself). The function uses shared memory to store checked numbers and processes and communicates perfect numbers to a manager via the message queue.

- **Parameters**:
  - `start`: The number to start checking for perfect numbers.
  - `MsgQ_ID`: The message queue identifier used to send perfect numbers to the manager process.
  
- **Details**:
  - **Bitmap Checking**: Uses a bitmap stored in shared memory to track whether a number has already been checked. If the number is marked, it is skipped.
  - **Perfect Number Calculation**: For each unmarked number, the function calculates the sum of its divisors. If the sum equals the number, it is identified as a perfect number.
  - **Message Queue Communication**: Found perfect numbers are sent to the manager by constructing a `messageQueueStruct` with type 2 (for perfect numbers) and sending it via the message queue.
  - **Bitmap Update**: Once a number is checked, it is marked in the bitmap to avoid rechecking.
  - **Counters**: The function updates the `FoundCounter` for perfect numbers found and the `TestedCounter` for numbers checked.
  - **Execl Call**: When all numbers are checked, the program replaces itself with the `report` program using `execl`. If this fails, the program returns `-1`.

---

### **main Function**
```c
int main(int argc, char *argv[])
```
- **Purpose**: This is the entry point of the program. It handles the initialization of IPC mechanisms (message queue, semaphore, shared memory), argument parsing, and process indexing. The main function eventually calls `Find_PerfectNums` to start searching for perfect numbers.

- **Details**:
  1. **Message Queue Initialization**: 
     - Creates a message queue using `msgget`. If it fails, an error message is printed, and the program exits.
  2. **Signal Registration**: 
     - Registers `C_Signal_Handler` for `SIGINT`, `SIGHUP`, and `SIGQUIT` to handle termination requests.
  3. **Argument Parsing**:
     - Uses `getopt` to parse command-line options. If the arguments are incorrect, it prints the correct usage (`SYNOPSIS`) and exits. The program expects exactly one argument, which is the starting number for checking perfect numbers.
  4. **PID Communication**:
     - Sends the process's PID to the manager through the message queue, indicating that this process is active and ready to compute perfect numbers.
  5. **Shared Memory and Semaphore Setup**:
     - Initializes the semaphore using `semget` and attaches the shared memory segment with `shmat`. If any of these operations fail, the program prints an error message and exits.
  6. **Semaphore Locking and Process Indexing**:
     - Locks the semaphore to prevent race conditions and then searches for the current process's PID in the shared memory's process structure. If no matching PID is found, the program exits with an error.
  7. **Perfect Number Search**:
     - Calls `Find_PerfectNums` with the starting number and message queue ID to begin searching for perfect numbers.
  
- **Error Handling**:
  - The program provides comprehensive error handling at every stage, ensuring that IPC mechanisms (semaphore, message queue, shared memory) are correctly initialized, and that proper resources are released upon termination.

---

### **Inter-Process Communication and Synchronization**
- **Message Queue**: The program uses a message queue to communicate with a manager process. Messages include the process's PID (for registration) and any found perfect numbers.
- **Shared Memory**: A shared memory segment is used to store global information such as the bitmap of checked numbers and process-specific counters (e.g., `FoundCounter`, `SkippedCounter`, `TestedCounter`).
- **Semaphores**: Semaphores are used to control access to shared memory, ensuring that only one process can modify shared memory structures at a time, preventing data races.

---
