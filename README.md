# Operating Systems Notes
These are some notes about how operating systems work. I've collected from all over the internet and added my own thoughts and words too.


## 1) Processes, Threads, and Scheduling

### Events which cause process creation:
- System initialization. (userinit and main)
- Execution of a process creation system call by a running process. (allocproc)
- A user request to create a new process. (allocproc)
- Initiation of a batch job.

### Events which cause process termination:
- Normal exit (voluntary).
- Error exit (voluntary).
- Fatal error (involuntary).
- Killed by another process (involuntary).

### Process states
A process can be in running, blocked, or ready state. (these are the basic states. There are other states too; like runnable, zombie, etc.)


### Process table entries for process management system calls:
Registers, program counter, program status word, stack pointer, process state, priority, scheduling parameters, process ID, parent process, process group, signals, time when process started, CPU time used, child's CPU time, time of next alarm.

(process table is basically an array of process structs. These variables are whats inside that process struct)


### What happens when an interrupt occurs?
1. Hardware stacks program counter, etc.
2. Hardware loads new PC from interrupt vector.
3. Assembly language procedure saves registers.
4. Assembly language procedure sets up new stack.
5. C interrupt service runs (typically reads and buffers input).
6. Scheduler decideds which process is to run next.
7. C procedure retursn to the assembly code.
8. Assembly language procedure starts up the new process.


### CPU intensive vs. I/O intensive at CPU utilization
Processes are either CPU intensive or I/O intensive. If the majority of the processes are I/O intensive (CPU has to wait for I/O to bring info), it takes more time for the CPU to use its max capacity or in other words, less CPU utilization. On the other hand, if the majority of process are CPU intensive, it's easier for CPU to get to 100% utilization.

### The classical thread model
Instead of having multiple process where each has one thread, we make one process with has multiple threads.
In this case, the process structure holds information about address space, global variables, child processes, and signals which is shared by all the threads inside that process and each thread has its own **stack**, program counter, and registers and also has a state value.
The process also has a thread table too.

### Race conditions
It happens when two processes want to access the same resource (aka shared memory) at the same time. For example, when both processes want to use the same global variable.
In this case, the shared memory is considered a critical region.

### Conditions required to avoid race condition:
- No two processes may be simultaneously inside their critical regions.
- No process should have to wait **forever** to enter its critical region.
- No assumptions may be made about speeds or the number of CPUs.
- No process running outside its critical region may block other processes.

The critical region should be **mutualy exclusive**.

### Mutual exclusion with busy waiting
We can achieve mutual exclusion by:
- Strict alternation
- Peterson's solution
- The TSL instruction
- Disabling interrupts
- Lock variables

#### Strict alternation:
It's mostly usable for two processes. Algorithm: Wait until the other process is out of the critical region and it's your turn.
#### Peterson's solution:
A process is allowed to be inside a critical section if:
1) It's its turn
2) The other process gives the priority to the process who wants to enter the critical section (`flag[other_process] == TRUE && turn == this_process`).
#### The TSL instruction:
Here's a simple assembly code for TSL algorithm:
```
enter_region:
    TSL REGISTER, FLAG ; copy flag to register and set flag to 1
    CMP REGISTER, #0 ; was flag zero?
    JNE enter_region ; if flag was non zero, lock was set, so loop
    RET ; return (and enter critical region)

leave_region:
    MOV FLAG, #0 ; store zero in flag
    RET ; return
```
#### Disabling interrupts:
Allow a process to disable interrupts before it enters its critical section and then enable interrupts after it leaves its critical section. There is a huge risk because the process might not ever finish its job or take a long time. The disadvantages of this solution is way more than the advantages.

### The producer-consumer problem
Here's the problem: The producer and the consumer both use a shared fixed-size buffer. The producer process adds data to the buffer and the consumer fetches that data.
We have to make sure that:
1) The producer doesn't produce data when the buffer is full.
2) The consumer doesn't fetch data when the buffer is empty.
**A simple solution** would be for the producer to go to sleep or discard the data if the buffer is full and for the consumer to go to sleep or discard the data if the buffer is empty. But there is another solution too; to use semaphores.

### Semaphores
It was suggested by Dijkstra in 1965! The alrogirthm saves the number of sleeps and wake ups. These variables are called semaphores.
- The sleep operation (aka DOWN or wait), checks the semaphore to see if it's greater than zero. If so, it decrements the value (using up a stored wakeup) and continues. If the semaphore is zero the process sleeps.
- The wakeup operation (aka UP or signal) increments the value of the semaphore. If one or more processes were sleeping on that semaphore then one of the processes is chosen and allowed to complete its DOWN.
To avoid the race conditions while updating the semaphores, we have to update them using atomic actions.
Note that semaphores are variable types.
Here's the producer-consumer problem solution using sempahores:
```
int BUFFER_SIZE = 100;
typedef int semaphore;

semaphore mutex = 1;
semaphore empty = BUFFER_SIZE;
semaphore full = 0;

void producer(void) {
    int item;
    while(TRUE) {
        produce_item(&item); // generate next item
        down(&empty); // decrement empty count
        down(&mutex); // enter critical region
        enter_item(item); // put item in buffer
        up(&mutex); // leave critical region
        up(&full); // increment count of full slots
    }
}

void consumer(void) {
    int item;
    while(TRUE) {
        down(&full); // decrement full count
        down(&mutex); // enter critical region
        remove_item(&item); // remove item from buffer
        up(&mutex); // leave critical region
        up(&empty); // increment count of empty slots
        consume_item(&item); // print item
    }
}
```
Description: When the full sempahore reaches to zero, the consumer sleeps until producer makes a new slot and it becomes non zero. Also, when the empty semaphore reaches to zero, the producer sleeps until the consumer uses one slot and it becomes non zero.

### Mutexes (locks)
```
mutex_lock:
    TSL REGISTER, MUTEX ; copy mutex to register and set mutex to 1
    CMP REGISTER, #0 ; was mutex zero?
    JZE ok ; if so, mutex was unlocked, so return
    CALL thread_yield ; if not, mutex is busy; so schedule another thread
    JMP mutex_lock ; try this again until the mutex is not busy

mutex_unlock:
    MOV MUTEX, #0 ; store 0 in the mutex
    RET ; return to the caller

ok:
    RET ; return to the caller
```
With `pthread`, we can use different functions for using mutexes.

### Monitors
A monitor consists of a mutex (lock) object and condition variables. A condition variable is a container of threads that are waiting for a certain condition. (It's really similar to other synchronization algorithms.)
Here's the producer consumer problem psuedo solution using monitors:
```
monitor ProducerConsumer {
    int itemCount = 0;
    condition full;
    condition empty;

    procedure add(item) {
        if (itemCount == BUFFER_SIZE) {
            wait(full);
        }
        putItemIntoBuffer(item);
        itemCount = itemCount + 1;
        if (itemCount == 1) {
            notify(empty);
        }
    }

    procedure remove() {
        if (itemCount == 0) {
            wait(empty);
        }
        item = removeItemFromBuffer();
        itemCount = itemCount - 1;
        if (itemCount == BUFFER_SIZE-1) {
            notify(full);
        }
        return item;
    }
}

procedure producer() {
    while (true) {
        item = produceItem();
        ProducerConsumer.add(item);
    }
}
procedure consumer() {
    while (true) {
        item = ProducerConsumer.remove();
        consumeItem(item); // print the item
    }
}
```
### Message passing
In a message-passing model, the sending and receiving processes need to coordinate sending and receiving messages with each other so that they make sure messages sent are eventually received and that messages received have actually been sent. They synchronize access to the shared channel. It's similar to previous algorithms. The only difference is that the producer and consumer, instead of using up and down, they send and recieve messages. Example: `recieve(consumer, &message)`


### Barriers
Barriers help us make sure that all of the processes will start/continue doing their procedure starting from the same time. This helps us synchronize shared or dependant data between processes.
When processes get to the barrier, they will be blocked until every other process arrives to the barrier to and then the processes will get through the barrier and continue running.

### Famous inter-process communication problems (IPC)
- The producer-consumer prblem
- The dining philisophers problem
- The readers writers problem
- The sleeping barber problem

## Scheduling
As I mentioned before, a process is either CPU intensive or I/O intensive. A CPU intensive process has a longer CPU burst time compared to an I/O intensive process.
Operating system is a mechanism provider that should provide different ways to schedule these processes with different burst times.

### Categories for scheduling algorithms:
- Batch
- Interactive
- Real time

### Scheduling algorithm goals:
**For all systems:**
– Fairness: giving each process a fair share of the CPU
– Policy enforcement: seeing that stated policy is carried out
– Balance: keeping all parts of the system busy
**For batch systems (like banks):**
– Maximize jobs per hour
– Minimize time between submission and termination of a task
– CPU utilization: keep the CPU busy at all times
**For interactive systems (like a normal PC):**
– Respond to requests quickly
– Meet users’ expectations (a task that is supposed to take a short time should finish quickly, and not surprise the user in terms of his/her expectations)
**For Real-time systems (like satellites or fire alarms):**
– Meeting deadlines: avoid losing data
– Predictability: avoid quality degradation in multimedia systems

### Scheduling in batch systems:
- First-come first-served
- Shortest job first: Maximize the throughput
- Shortest remaining Time next

### Scheduling in interactive systems:
- Round-robin scheduling: Each process has a certain period in which it's allowed to do it's job (this is called quantum.) When that period passes, the scheduler will automatically choose the next runnable process and the previous process will be rescheduled to contiune its job.
- Priority scheduling: In this method, CPU has multiple queues with different priorities. The scheduler has to choose a process from the higher-priority queues first (while being fair).
- Multiple queues: This method is similar to the previous method with the slight difference that the processes inside each queue can have different priorities.
- Fair-share scheduling: In this algorithm, the CPU usage is equally distributed among system users or groups, as opposed to equal distribution among processes. One common method of logically implementing the fair-share scheduling strategy is to recursively apply the round-robin scheduling strategy at each level of abstraction (processes, users, groups, etc.) The time quantum required by round-robin is arbitrary, as any equal division of time will produce the same results.

**Other aglorithms:**
- Shortest process next
- Guaranteed scheduling
- Lottery scheduling

### Thread scheduling vs. process scheduling
As mentioned before, a process is consisted of a couple of threads. If the kernel has to schedule processes, then it has no choice other than picking the threads inside a process all together. But there is another way too; the scheduler, is able to access and pick threads (instead of a whole process) amongst all runnable processes. Think of it as a picture  with higher resolution where kernel has more to work with.
