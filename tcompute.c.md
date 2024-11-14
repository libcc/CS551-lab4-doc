### Section by Section Documentation

---

#### 1. **Includes and Global Variables**
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
#include <pthread.h>
```
This section imports the necessary libraries and headers:
- Standard libraries (`stdio.h`, `stdlib.h`, `string.h`, etc.) for basic I/O and memory management.
- System-specific libraries (`sys/ipc.h`, `sys/msg.h`, `sys/sem.h`, `sys/shm.h`) for handling Inter-Process Communication (IPC), message queues, semaphores, and shared memory.
- `pthread.h` for threading operations.
- `signal.h` to handle signals like SIGINT.

---

#### 2. **Global Variables**
```c
int Index;             // Index of the current process in shared memory.
int MsgQ_ID;           // ID of the message queue for IPC.
int Semaphore_ID;      // ID of the semaphore for synchronization.
int UNLOCK_SEM = 1;    // Constant value for unlocking the semaphore.
int LOCK_SEM = -1;     // Constant value for locking the semaphore.
int THREADS = 5;       // Number of threads used for computing.
int TERMINATED_OPT = 1;// Flag to check termination option.
long start = -1;       // Starting index for bitmap search.
long Current_Position_in_BitMap = 0; // Position in the bitmap for processing.
int N = 0;             // Holds the number being tested for perfection.
int N_Taken = 0;       // Holds the last processed number.
int ComputeFinished = 1; // Flag to indicate computation is still running.
SharedMemoryStruct *sharedMemoryInfoStruct; // Pointer to shared memory.
```
These variables manage the state of the program, including message queue and semaphore IDs, bitmap search position, synchronization flags, and shared memory access.

---

#### 3. **Thread and Mutex Initialization**
```c
pthread_cond_t NewNumProcessed_cond = PTHREAD_COND_INITIALIZER;
pthread_cond_t NewNumNotProcessed_cond = PTHREAD_COND_INITIALIZER;
pthread_mutex_t NewNum_mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t PNUpdate_mutex = PTHREAD_MUTEX_INITIALIZER;
```
These variables define condition variables and mutexes for thread synchronization, ensuring correct access to shared data.

---

#### 4. **Semaphore Operations**
```c
void Semaphore_Operation(int Semaphore_ID, int operation) {
    struct sembuf semaphoreBufferStruct = { .sem_num = 0, .sem_op = operation, .sem_flg = 0 };
    const char *operationName = (operation == LOCK_SEM) ? "lock" : "unlock";
    if (semop(Semaphore_ID, &semaphoreBufferStruct, 1) == -1) {
        fprintf(stderr, "ERROR: Failed to %s the semaphore!\n", operationName);
        exit(1);
    }
}
```
This function performs semaphore operations for locking and unlocking. It accepts the semaphore ID and the operation (lock or unlock) and ensures that the operation is successfully applied.

---

#### 5. **Semaphore Lock and Unlock Helpers**
```c
void Semaphore_Lock(int Semaphore_ID) {
    Semaphore_Operation(Semaphore_ID, LOCK_SEM);
}

void Semaphore_Unlock(int Semaphore_ID) {
    Semaphore_Operation(Semaphore_ID, UNLOCK_SEM);
}
```
These are helper functions to simplify locking and unlocking the semaphore using the `Semaphore_Operation` function.

---

#### 6. **Bitmap Search Thread**
```c
void BitMapSearchThread() {
    start %= MAX_NUMBER;
    Current_Position_in_BitMap = start;
    // Loops through bitmap to search for available positions.
    // If a position is available, it sets the bitmap bit, updates counters, 
    // and signals the `NewNumNotProcessed_cond` condition to notify other threads.
    // The loop continues until the starting position is reached again, at which point 
    // the program is marked as finished and an external report process is executed.
}
```
This function runs in a separate thread, searching for free positions in the bitmap. It locks the shared memory to update the number to be processed and signals other threads when a new number is available. It terminates after completing the search and runs an external program for reporting results.

---

#### 7. **Perfect Number Computation Thread**
```c
void ComputePerfectNumThreads() {
    int newNumber = 0;
    do {
        pthread_mutex_lock(&NewNum_mutex);
        while (N_Taken == N) {
            pthread_cond_wait(&NewNumNotProcessed_cond, &NewNum_mutex);
        }
        newNumber = N;
        N_Taken = N;
        pthread_mutex_unlock(&NewNum_mutex);
        pthread_cond_signal(&NewNumProcessed_cond);

        // Perfect number test: checks whether `newNumber` is a perfect number.
        // If it is, a message is sent via the message queue to the manager.
    } while (ComputeFinished != 0);
}
```
This function is responsible for checking if the given number is a perfect number. It processes numbers signaled by the bitmap search thread, computes the sum of divisors, and if a perfect number is found, it sends a message to the manager using the message queue.

---

#### 8. **Message Queue Creation**
```c
int Create_MsgQ() {
    int MsgQ_ID = msgget(KEY, IPC_CREAT | 0660);
    if (MsgQ_ID == -1) {
        fprintf(stderr, "ERROR: Creating message queue failed\n");
        return -1;
    }
    return MsgQ_ID;
}
```
This function creates a message queue and returns its ID. If the creation fails, an error is reported.

---

#### 9. **Signal Handler**
```c
void C_Signal_Handler(int signalNumber) {
    if (signalNumber == SIGINT || signalNumber == SIGHUP || signalNumber == SIGQUIT) {
        // Handles termination signals (SIGINT, SIGHUP, SIGQUIT).
        // Sends a termination message to the manager and updates the shared memory
        // before exiting the process.
    }
}
```
This function handles system signals like SIGINT and cleans up resources. It sends a termination message to the manager and updates the process information in shared memory.

---

#### 10. **Main Function**
```c
int main(int argc, char *argv[]) {
    int opt = 0;
    signal(SIGINT, C_Signal_Handler);
    signal(SIGHUP, C_Signal_Handler);
    signal(SIGQUIT, C_Signal_Handler);

    pthread_t BitMapSearch_thread;
    pthread_t ComputePerfectNum_threads[THREADS];

    // Argument parsing and message queue creation
    while ((opt = getopt(argc, argv, "t")) != -1) { /* handle options */ }

    MsgQ_ID = Create_MsgQ();

    // Semaphore and shared memory initialization
    if ((Semaphore_ID = semget(KEY, 1, IPC_CREAT | 0660)) == -1 ||
        (SharedMem_Id = shmget(KEY, sizeof(SharedMemoryStruct), IPC_CREAT | 0660)) == -1 ||
        (sharedMemoryInfoStruct = ((SharedMemoryStruct *)shmat(SharedMem_Id, 0, 0))) == (SharedMemoryStruct *)-1) {
        fprintf(stderr, "ERROR: Failed operation - Semaphore, Shared Memory Segment, or Attach.\n");
        exit(1);
    }

    // Start the bitmap search and computation threads
    pthread_create(&BitMapSearch_thread, NULL, (void *)&BitMapSearchThread, NULL);
    for (int i = 0; i < THREADS; i++) {
        pthread_create(&ComputePerfectNum_threads[i], NULL, (void *)&ComputePerfectNumThreads, NULL);
    }

    // Wait for threads to finish
    pthread_join(BitMapSearch_thread, NULL);
    for (int i = 0; i < THREADS; i++) {
        pthread_join(ComputePerfectNum_threads[i], NULL);
    }

    return 0;
}
```
This is the program's entry point. It sets up signal handling, parses command-line arguments, creates the message queue, semaphore, and shared memory, and launches the bitmap search and computation threads. The program waits for all threads to finish before terminating.

