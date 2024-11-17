# Documentation for `manage.c`

## Overview
The `manage.c` program serves as a manager for multiple "compute" processes that work together to find perfect numbers. It utilizes **shared memory**, **message queues**, and **signals** to handle communication between the manager and compute processes. The manager is responsible for initializing compute processes, receiving and handling messages (e.g., found perfect numbers, process initialization), and managing the lifecycle of these processes (including termination).

### Key Responsibilities:
- **Process Management**: Initialize and manage the state of multiple compute processes.
- **Shared Memory**: Store process data and found perfect numbers.
- **Message Queue**: Facilitate communication between the manager and compute processes.
- **Signal Handling**: Manage process termination and resource cleanup through signal handling.

---

## Global Variables
### Process and IPC (Inter-Process Communication) Control
- **`smid`**: The ID of the shared memory segment.
- **`mqid`**: The ID of the message queue.
- **`cp_pid`**: Process ID of the current compute process.
- **`cp_count`**: Counter that tracks the number of compute processes initialized.
- **`result_count`**: Counter for the number of perfect numbers found.
- **`memory`**: Pointer to the shared memory segment, which holds process information and results.

---

## Function Definitions

### `mg_handler(int signum)`
This function handles termination signals (`SIGINT`, `SIGQUIT`, `SIGHUP`). The purpose is to gracefully shut down all compute processes and clean up shared resources (memory and message queue).

#### Steps:
1. Loop through all active compute processes stored in shared memory (`memory->process`), and send a `SIGINT` to each one to terminate them.
2. Wait for 5 seconds to ensure all processes have enough time to terminate.
3. Detach from the shared memory (`shmdt(memory)`), ensuring the manager no longer has access to the shared memory.
4. Remove the shared memory segment (`shmctl(smid, IPC_RMID, 0)`) and message queue (`msgctl(mqid, IPC_RMID, NULL)`).
5. Exit the program.

### `main(int argc, char *argv[])`
The main function sets up the shared memory, message queue, and signal handlers. It enters an infinite loop to handle messages from compute processes, initializing them and collecting perfect numbers as they are found.

#### Steps:

1. **Shared Memory Setup**:
   - Create a shared memory segment with `shmget()`. If successful, attach the shared memory to the `memory` pointer.
   - If attaching or creating shared memory fails, print an error message and exit.

2. **Message Queue Setup**:
   - Create a message queue with `msgget()`. If successful, store the ID in `mqid`.
   - If the message queue creation fails, print an error message and exit.

3. **Signal Handlers**:
   - Use `sigaction()` to set up custom signal handlers for `SIGINT`, `SIGQUIT`, and `SIGHUP`. When these signals are caught, the `mg_handler()` will be invoked to handle process termination and cleanup.

4. **Message Processing Loop**:
   - Enter an infinite loop where the manager waits for messages using `msgrcv()` to receive messages from the message queue.
   - The manager only processes messages with specific flags:
     - **INIT_FLAG**: Indicates that a new compute process has been started and its process ID (PID) has been sent to the manager.
       - The manager initializes the process state in shared memory (PID, number of tested/skipped/found numbers).
       - The manager sends back the index of the process in the shared memory for further communication with that process.
     - **FOUND_FLAG**: Indicates that a perfect number has been found by a compute process.
       - The manager stores the perfect number in the `result[]` array in shared memory.
       - If more than `MAX_PERFECT` numbers are found, a warning message is printed.
     - **KILL_FLAG**: Indicates that a specific compute process needs to be terminated.
       - The manager retrieves the PID of the target process from shared memory and sends a `SIGINT` signal to terminate it.

---

## Message Structure
The manager and compute processes communicate using a message structure defined in the `msg` structure:

```c
typedef struct {
    long flag;      // Message type: INIT_FLAG, FOUND_FLAG, KILL_FLAG, etc.
    int data;       // Process ID or perfect number found, depending on the message type
} msg;
```

- **flag**: Used to distinguish between different types of messages (initialization, found perfect number, or kill command).
- **data**: Stores the process ID (for initialization or termination) or the perfect number that was found.

---

## Signal Handling
Three signals are managed by the manager to handle termination:
- **`SIGINT`**: Typically triggered by pressing `Ctrl+C`. Used to initiate cleanup and termination of all compute processes.
- **`SIGQUIT`** and **`SIGHUP`**: These signals are also handled and perform the same cleanup procedure as `SIGINT`.

---

## Shared Memory Structure
The shared memory segment stores both process information and results:
- **Process Data**: Each process has an entry in the shared memory, which includes its PID and statistics (numbers tested, found, and skipped).
- **Perfect Numbers**: An array in shared memory (`result[]`) stores the perfect numbers found by compute processes.

---

## Error Handling
The manager program includes comprehensive error handling:
- If any system calls for creating or attaching shared memory, creating message queues, or sending messages fail, the program prints detailed error messages and terminates.
- If there are too many compute processes (`cp_count > MAX_PROCESS`), the program will exit with an error message.
- If too many perfect numbers are found (`result_count > MAX_PERFECT`), a warning message is printed, suggesting an unusual scenario (since finding more than 20 perfect numbers is highly improbable).
