# `compute.c` Program Documentation

## Overview
The `compute.c` program is designed to search for perfect numbers within a specified range using shared memory and inter-process communication. It is part of a larger system that includes multiple processes, each of which tests candidate numbers for perfection. The program leverages shared memory to keep track of which numbers have been tested and communicates results to a manager process using message queues. Signals are used to handle termination and clean up the process state.

### Key Features:
- **Shared Memory**: Used to track the status of tested numbers and store process statistics.
- **Message Queues**: For inter-process communication (IPC) with a manager process, such as sending perfect number discoveries and handling initialization.
- **Signal Handling**: Custom signal handling is used to gracefully clean up the process state upon termination.
- **Perfect Number Testing**: Each candidate number is tested to determine whether it is a perfect number (i.e., the sum of its divisors equals the number itself).

## Dependencies
- `hw4.h`: This header file defines necessary constants, structures, and IPC keys like `SM_KEY`, `MQ_KEY`, and other project-specific macros.
- **System Libraries**:
  - `<signal.h>`: For signal handling.
  - `<sys/shm.h>`: For shared memory management.
  - `<sys/msg.h>`: For message queue management.
  - `<errno.h>`: For handling system errors.
  - `<stdlib.h>`, `<stdio.h>`, `<string.h>`, `<stdbool.h>`: For standard I/O, memory allocation, string manipulation, and boolean support.

## Data Structures
- `segment *memory`: A pointer to the shared memory segment that stores process information and a bitmap for tracking tested numbers.
- `msg`: A structure used for message communication between this program and the manager process.
- Various integer variables are used for process ID (`pid`), message queue ID (`mqid`), shared memory ID (`smid`), and the candidate index (`cp_index`).

## Functions

### `void cp_handler(int signum)`
This function handles termination signals (`SIGINT`, `SIGQUIT`, `SIGHUP`). When the program receives any of these signals, this handler is invoked to reset the process information in shared memory and terminate the process cleanly.
- **Parameters**: 
  - `int signum`: The signal number.
- **Effect**: Resets the process's shared memory slot, clearing its PID and statistics, then exits the program.

### `int whichSeg(int cand)`
Calculates the index of the shared memory segment corresponding to a candidate number.
- **Parameters**: 
  - `int cand`: The candidate number being tested.
- **Returns**: The index of the shared memory segment containing the bitmap for this candidate.

### `int whichInt(int cand)`
Calculates the index of the integer in the bitmap array that contains the bit corresponding to the candidate number.
- **Parameters**: 
  - `int cand`: The candidate number being tested.
- **Returns**: The index of the integer in the bitmap that contains the bit for this candidate.

### `int whichBit(int cand)`
Calculates the bit position within an integer that corresponds to a candidate number.
- **Parameters**: 
  - `int cand`: The candidate number.
- **Returns**: The bit position within the integer that represents the candidate.

### `int test(int cand)`
Tests if a given candidate number is a perfect number. A perfect number is a number that is equal to the sum of its proper divisors.
- **Parameters**: 
  - `int cand`: The candidate number to test.
- **Returns**: 
  - `1` if the candidate is a perfect number.
  - `0` otherwise.

### `int tested(int cand)`
Checks whether a candidate number has already been tested by inspecting the corresponding bit in the shared memory bitmap.
- **Parameters**: 
  - `int cand`: The candidate number.
- **Returns**: 
  - Non-zero if the candidate has already been tested.
  - `0` if the candidate has not been tested.

## Main Program Flow
1. **Initialization**: 
   - The program begins by verifying that the correct number of arguments is provided (only one argument is expected—the starting number for candidate testing). 
   - It checks that the starting number is within a valid range (`2 <= start <= 33554432`).
   
2. **Shared Memory and Message Queue Setup**:
   - The program attempts to attach to shared memory and the message queue, both of which are essential for inter-process communication.
   - Upon successful connection, the program sends an initialization message to the manager process, which assigns a unique index (`cp_index`) for this process.

3. **Signal Handling**:
   - Custom signal handling is set up using `sigaction`, where signals (`SIGINT`, `SIGQUIT`, `SIGHUP`) are caught and processed by `cp_handler` to ensure graceful cleanup.

4. **Candidate Testing Loop**:
   - The program enters a loop where it tests candidate numbers starting from the specified `start` value.
   - For each candidate, it checks whether the number has been tested by inspecting the bitmap in shared memory.
     - If the candidate has not been tested:
       - It marks the candidate as tested in shared memory.
       - It tests whether the candidate is a perfect number using the `test` function.
       - If the candidate is a perfect number, it sends a message to the manager process indicating the discovery.
     - If the candidate has already been tested, it increments the skipped count for this process in shared memory.
   - The loop continues testing numbers until the maximum range (`33554432`) is reached, after which the process terminates.

5. **Termination**:
   - When the candidate reaches the maximum range, the program resets to the starting position and calls `cp_handler` to handle cleanup and exit.

## Error Handling
- The program checks for various errors during execution, such as:
  - Invalid argument count or starting number.
  - Failure to attach to shared memory or message queues, which triggers an error message and program exit.
  - Failure to send or receive messages, which also triggers an error message and program exit.

## Usage
```bash
./compute start_number
```
- `start_number`: The number from which the candidate testing begins.
- Example:
```bash
./compute 5000
```

## Limits
- The program tests numbers in the range `2 <= start <= 33554432`.
- If the `start_number` is outside this range, the program exits with an error message.
