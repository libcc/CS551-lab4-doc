# `report.c` - Documentation

## Overview

This C program, `report.c`, is part of a system that manages processes involved in finding perfect numbers. It connects to shared memory and a message queue, both set up by another program (`manage.c`), to monitor and report the status of running processes. The program also has the ability to send termination signals to all active processes.

The program performs the following main tasks:
1. Retrieves and displays the perfect numbers found so far.
2. Summarizes the performance of currently running processes (i.e., how many numbers they have tested, perfect numbers they have found, and numbers they have skipped).
3. Optionally, it can send a termination signal to all running processes if the `-k` argument is provided.

## Global Variables

- `smid`: Shared memory ID obtained through `shmget()`.
- `mqid`: Message queue ID obtained through `msgget()`.
- `total_tested`: Accumulates the total number of numbers tested by all running processes.
- `total_found`: Accumulates the total number of perfect numbers found by all running processes.
- `total_skipped`: Accumulates the total number of skipped numbers by all running processes.
- `pid`: Stores the process ID of each running process when iterating over processes in shared memory.
- `p_tested`, `p_found`, `p_skipped`: Store the number of tested, found, and skipped numbers for each individual process.
- `memory`: Pointer to the shared memory segment, which contains process statistics and perfect numbers.

## Functions

### `main(int argc, char *argv[])`

#### Purpose

The main function performs the following tasks:
1. **Accesses shared memory and message queue**: The program attaches to the shared memory segment and the message queue created by `manage.c`. These contain the statistics of processes and perfect numbers found.
2. **Kill processes (if `-k` flag is used)**: If the program is executed with the `-k` flag, it sends a termination signal to all running processes.
3. **Displays perfect numbers**: The program prints all the perfect numbers found so far from shared memory.
4. **Displays running process statistics**: For each active process, the program displays its PID, how many numbers it has tested, how many perfect numbers it has found, and how many numbers it has skipped.
5. **Summarizes total statistics**: It also computes and prints the total numbers tested, perfect numbers found, and numbers skipped across all running processes.

#### Detailed Steps

1. **Memory and Queue Initialization**:
    - The program retrieves the shared memory segment ID using `shmget(SM_KEY, sizeof(segment), 0)`. If it fails, an error message is printed, and the program exits.
    - The shared memory is then attached using `shmat()`. If this fails, an error message is printed, and the program exits.
    - The program retrieves the message queue ID using `msgget(MQ_KEY, 0)`. If this fails, an error message is printed, and the program exits.

2. **Handling the `-k` Flag**:
    - If the `-k` flag is passed as an argument (`strcmp(argv[1], "-k") == 0`), the program sends a termination message (`KILL_FLAG`) to all active processes.
    - The program iterates over the shared memory process table (`memory->process`), checking each process's PID. For each active process, it sends a `KILL_FLAG` message using `msgsnd()`. The loop breaks when it encounters a process with a PID of 0 (indicating no more active processes).

3. **Displaying Perfect Numbers**:
    - The program prints the perfect numbers found so far by iterating through `memory->result`. It stops printing once it encounters a `0` in the array, which indicates the end of the list of perfect numbers.

4. **Displaying Process Statistics**:
    - The program iterates through the process table in shared memory, printing the statistics (tested, found, skipped numbers) for each active process. It also accumulates these values into `total_tested`, `total_found`, and `total_skipped` to provide a summary.
    - For each process:
      - `memory->process[i].pid`: The process ID.
      - `memory->process[i].tested`: The number of numbers tested.
      - `memory->process[i].found`: The number of perfect numbers found.
      - `memory->process[i].skipped`: The number of numbers skipped.
    - The loop terminates when a process with a PID of 0 is encountered (indicating no more active processes).

5. **Displaying Total Statistics**:
    - After printing the statistics for each process, the program prints the total number of numbers tested, perfect numbers found, and numbers skipped across all running processes.

#### Error Handling

- **Shared Memory Access**: If `shmget()` or `shmat()` fail to access or attach to the shared memory, an error message is printed, and the program exits.
- **Message Queue Access**: If `msgget()` fails to access the message queue, an error message is printed, and the program exits.
- **Kill Signal Failure**: If `msgsnd()` fails to send the kill signal, an error message is printed, and the program exits.

## Usage

```
./report              # Displays the status of processes and the perfect numbers found
./report -k           # Sends a termination signal to all active processes
```

- Without any arguments, the program prints the perfect numbers and the statistics of the currently running processes.
- With the `-k` flag, the program sends a termination signal to all active processes.

## Data Structures

### `segment`

This structure, defined in `hw4.h`, stores information about the processes and the perfect numbers found. It is accessed through shared memory.

- `process[]`: An array that stores information about each process:
  - `pid`: Process ID of each compute process.
  - `tested`: Number of numbers tested by the process.
  - `found`: Number of perfect numbers found by the process.
  - `skipped`: Number of numbers skipped by the process.
  
- `result[]`: An array that stores the perfect numbers found by all compute processes.

### `msg`

The `msg` structure, also defined in `hw4.h`, is used for message passing between the `manage.c` program and the compute processes.

- `flag`: Indicates the type of message (e.g., `KILL_FLAG`).
- `data`: Carries additional data (e.g., process ID or index).
