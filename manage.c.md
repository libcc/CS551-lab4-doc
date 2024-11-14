

#### 1. **Include Directives**
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
This section includes header files for system and standard libraries required for inter-process communication (IPC), signal handling, and shared memory management:
- `"defs.h"`: Likely a custom header defining constants, structures, or functions used across the program.
- `<stdio.h>`: Provides functions for input/output operations.
- `<stdlib.h>`: Supports memory allocation, process control, and conversions.
- `<string.h>`: Contains string manipulation functions.
- `<unistd.h>`: Declares access to the POSIX operating system API (e.g., `fork`, `getpid`).
- `<signal.h>`: Defines signal handling functions, such as `signal()`.
- `<sys/ipc.h>`, `<sys/msg.h>`, `<sys/sem.h>`, `<sys/shm.h>`, `<sys/types.h>`: Allow IPC mechanisms, including message queues, semaphores, and shared memory.

#### 2. **Global Variables and Structures**
```c
int Semaphore_ID;
int MsgQ_ID;
int SharedMem_ID;
int UNLOCK_SEM = 1;
int LOCK_SEM = -1;
int TERMINATED_OPT = 1;
typedef struct {
    long type; // 1: PID, 2: Number, 3: Terminate
    long data; 
} messageQueueStruct;
SharedMemoryStruct *SharedMem_Struct;
```
- **`Semaphore_ID`, `MsgQ_ID`, `SharedMem_ID`**: Global variables storing identifiers for the semaphore, message queue, and shared memory, respectively.
- **`UNLOCK_SEM` and `LOCK_SEM`**: Constants used to represent semaphore operations (unlock and lock).
- **`TERMINATED_OPT`**: A flag for process termination handling (1 for active).
- **`messageQueueStruct`**: Defines the structure for the messages sent through the message queue, containing a `type` field (indicating the kind of message) and a `data` field (used for process PID or data).

#### 3. **Semaphore Operations**
```c
void Semaphore_Operation(int Semaphore_ID, int operation) {
    ...
}

void Semaphore_Lock(int Semaphore_ID) {
    Semaphore_Operation(Semaphore_ID, LOCK_SEM);
}

void Semaphore_Unlock(int Semaphore_ID) {
    Semaphore_Operation(Semaphore_ID, UNLOCK_SEM);
}
```
- **`Semaphore_Operation`**: Handles semaphore locking or unlocking by creating and configuring a `sembuf` structure and performing the `semop` system call to change the semaphore state. It handles errors and prints appropriate messages on failure.
- **`Semaphore_Lock`** and **`Semaphore_Unlock`**: Helper functions to lock and unlock semaphores by calling `Semaphore_Operation` with the corresponding operation.

#### 4. **Signal Handling**
```c
void Signal_Ignore(int signalNumber) {
    signal(signalNumber, SIG_IGN);
}

void Kill_All_Processes(void) {
    ...
}

void Cleanup_Resources(void) {
    ...
}

void M_Signal_Handler(int signalNumber) {
    ...
}
```
- **`Signal_Ignore`**: Sets a signal to be ignored by the program using `SIG_IGN`.
- **`Kill_All_Processes`**: Iterates over all processes stored in shared memory and sends a `SIGINT` signal to terminate them.
- **`Cleanup_Resources`**: Cleans up the IPC resources (message queue, semaphore, shared memory) by removing them via system calls `msgctl`, `semctl`, and `shmctl`.
- **`M_Signal_Handler`**: Custom signal handler for terminating the program. It sets the termination flag in shared memory, kills all processes, cleans up resources, and exits.

#### 5. **Message Handlers**
```c
void PIDMessage_Handler(messageQueueStruct RCVDmsg_compute, SharedMemoryStruct *sharedMemStruct, int semaphoreID) {
    ...
}

void NumMessage_Handler(messageQueueStruct RCVDmsg_compute, SharedMemoryStruct *sharedMemStruct) {
    ...
}

void TerminationMessage_Handler(messageQueueStruct RCVDmsg_compute, SharedMemoryStruct *sharedMemStruct, int semaphoreID, int terminatedOption) {
    ...
}
```
- **`PIDMessage_Handler`**: Handles messages of type `1` (PID). It finds an empty slot in the `ProcessessStruct` array in shared memory and stores the received process ID. The semaphore is unlocked once the PID is registered.
- **`NumMessage_Handler`**: Handles messages of type `2` (numbers). It stores the number in the `PerfectNumber` array in shared memory. If the number is new, it is inserted and the array is sorted.
- **`TerminationMessage_Handler`**: Handles messages of type `3` (termination). It finds an empty slot in the `TerminatedProcessessStruct` and updates its counters (e.g., `FoundCounter`, `SkippedCounter`, `TestedCounter`) with the data from the terminated process.

#### 6. **Main Program Execution**
```c
int main(void) {
    ...
}
```
- **Signal Setup**: Registers the custom signal handler `M_Signal_Handler` for signals `SIGINT`, `SIGHUP`, and `SIGQUIT` to handle process termination gracefully.
- **IPC Initialization**: The program attempts to create and attach IPC resources (message queue, semaphore, shared memory). If any operation fails, an error message is displayed, and the program exits.
- **Process ID Management**: The current process ID is saved in shared memory as `manage_PID`.
- **Message Handling Loop**: The program enters an infinite loop to receive messages (`msgrcv`). Depending on the message type:
  - Type `1`: Calls `PIDMessage_Handler` to manage process IDs.
  - Type `2`: Calls `NumMessage_Handler` to store and sort numbers.
  - Type `3`: Calls `TerminationMessage_Handler` to update the termination structure.
  - Unknown types trigger an error message.
- **Resource Cleanup**: When the loop ends, the message queue, semaphore, and shared memory are cleaned up using their respective system calls.

#### 7. **Exit and Cleanup**
At the end of the `main` function, the program cleans up IPC resources explicitly before exiting. This ensures no IPC resources are left in the system after termination. 

### Conclusion
This program implements a shared-memory based process management system using IPC mechanisms. It handles multiple processes through message queues, semaphores, and shared memory, with robust signal handling for clean termination.
