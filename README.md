# Operating Systems Notes
These are some notes about how operating systems work. I've collected from all over the internet and added my own thoughts and words too. It is based on the Tanenbaum's Modern Operating Systems book.

Here is the table of contents:
1) [Processes, Threads, and Scheduling](#1-processes-threads-and-scheduling)
2) [Memory Management](#2-memory-management)


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
- Multiple queues: This method is similar to the previous method with the slight difference that the processes inside each queue gets different quantums. So the shortest gets (high priority) out first.
- Fair-share scheduling: In this algorithm, the CPU usage is equally distributed among system users or groups, as opposed to equal distribution among processes. One common method of logically implementing the fair-share scheduling strategy is to recursively apply the round-robin scheduling strategy at each level of abstraction (processes, users, groups, etc.) The time quantum required by round-robin is arbitrary, as any equal division of time will produce the same results.
- Shortest process next: It works if we know the remaining times of processes.
- Lottery scheduling: Hold lottery for cpu time several times a second.

**Other aglorithms:**
- Guaranteed scheduling

### Thread scheduling vs. process scheduling
As mentioned before, a process is consisted of a couple of threads. If the kernel has to schedule processes, then it has no choice other than picking the threads inside a process all together. But there is another way too; the scheduler, is able to access and pick threads (instead of a whole process) amongst all runnable processes. Think of it as a picture  with higher resolution where kernel has more to work with.


## 2) Memory management
So here's the problem: Computers don't have infinite memory and even if we had 2048TB of RAM, it still wouldn't be enough.
We do have different types of memory:
- Cache (which is fast)
- Memory (which has a good enough speed)
- Disk (which is slow)
Memory manager has the job of using this hierarchy to create an abstraction (illusion) of easily accessible memory.

### Static relocation
We have more than one program at a time to run. We have to think of a way to use the limited memory to run multiple programs. The most basic idea would be static relocation.
Here's how it works:
- Divide memory into (for example) 2 KB blocks, and associate a 4 bit protection key. Keep keys in registers and hardware prevents program from accessing block
with another protection key.
- Load first instruction of program at address x, and add x to every subsequent address during loading.

Two problems with static recloation:
- Let's say there is an `ADD` instruction in the 25th block. We run another program that has a `JMP 25` instruction and in the 25th block it has a `CMP` (or any other instruction). In this case our `ADD` instruction that previously existed, will be lost.
- Also, static reloctaion is too slow.

Instead of having our programs access the memory directly, we create abstract memory space for program to exist in. In this case:
- Each program has its own set of addresses
- The addresses are different for each program
- Call it the address space of the program

This is called dynamic relocation.

### Base and limit registers
- It's a form of dynamic relocation
- Base contains beginning address of program
- Limit contains length of program
- Program references memory, adds base address to address generated by process. Checks to see if address is larger then limit. If so, it generates fault.

Disadvantage: Addition and comparison have to be done on every instruction.

### How to run more programs and fit them in the main memory at once?
We can't keep all of processes in the main memory; they might be too much (hundreds) and too big (e.g. 200MB program). We have two approaches to solve this problem:
- Swap: Bring program in and run it for awhile then put it back and bring another program.
- Virtual memory: Allow program to run even if only part of it is in main memory

## Swapping
![Swapping](/photos/swapping.png)

Programs grow as they execute; to handle the growth we have some solutions:
- Using stack (return addresses and local variables)
- Data segmentation (heap for variables which are dynamically allocated and released)

It's a good idea to allocate extra memory for both of these solutions. Also, when program goes back to disk, don’t bring holes along with it!!!

#### Two ways to allocate space for growth
![Two ways](/photos/twoways.png)

We can:
a) Just add some extra space (have some room for growth).
b) Have our stack grow downwards, and data grow upwards.


### Managing free memory
Two techniques to keep track of free memory:
- Bitmaps
- Linked lists
- Grouping
- Counting

### Bitmaps
A Bitmap or Bit Vector is series or collection of bits where each bit corresponds to a disk block. The bit can take two values: 0 and 1: 0 indicates that the block is allocated and 1 indicates a free block. The given instance of disk blocks on the disk in Figure 1 (where green blocks are allocated) can be represented by a bitmap of 16 bits as: 0000111000000110.

![Bitmaps](/photos/bitmap.png)


**Advantage:** Finding the first free block is efficient. It requires scanning the words (a group of 8 bits) in a bitmap for a non-zero word. (A 0-valued word has all bits 0). The first free block is then found by scanning for the first 1 bit in the non-zero word.


### Linked lists
In this approach, the free disk blocks are linked together i.e. a free block contains a pointer to the next free block. The block number of the very first disk block is stored at a separate location on disk and is also cached in memory.
- We might want to use doubly linked lists to merge holes more easily.
- Algorithms to fill in the holes in memory:
    - Next fit
    - Best fit: Smallest hole that fits (it's slow)
    - Worst fit: Largest hole that fits (not usable)
    - Quick fit: keep list of common sizes (it's quick, but it can’t find neighbors to merge with)

![Linkedlist](/photos/linkedlist.png)

**Conclusion:** the fits couldn’t out-smart the unknowable distribution of hole sizes
**A drawback** of this method is the I/O required for free space list traversal.

### Tradeoff between bitmaps and linkedlists:
If the disk is almost full, it might make sense to use a linked list, as it will need less blocks than the bitmap. However, most of the time the bitmap will be store in main memory, which will make it more efficient than the linked list. I guess that if the disk is almost full, and you can store the linked list in main memory, then it's a good alternative too.

## Virtual Memory
- Program’s address space is broken up into fixed size pages
- Pages are mapped to physical memory
- If instruction refers to a page in memory, fine
- Otherwise, page fault happens; OS gets the page, reads it in, and re-starts the instruction
- While page is being read in, another process gets the CPU

**Memory Management Unit (MMU)** generates physical address from virtual address provided by the program and puts them on memory bus.

### Pages and page frames
- Virtual addresses are divided into pages (e.g. 512 bytes-64 KB range)
- Transfer between RAM and disk is in whole pages

![Virtual Memory](/photos/virtualmemory.jpg)

(obviously, the number of virtual pages will be more than physical pages)

### Page fault processing
There is a present/absent bit which tells whether a page is in memory.
If address is not in memorya "trap" to OS happens:
- OS picks page to write to disk
- Brings page with (needed) address into memory
- Re-starts instruction

### Page Table
![Virtual Address](/photos/virtualaddress.png)

- Virtual Address = (virtual page number, offset)
- Virtual page number helps us find the index of the virtual address inside the page table
- After the index is calculated, the present/absent bit is checked to find out whether the page already exists.
- If present/absent bit is set to 1, attach page frame number to the front of the offset to create the physical address which is sent on the memory bus.


#### Page table entry
![Page Table Entry](/photos/pagetableentry.png)
- Modified (dirty) bit: 1 means it has to written it to disk. 0 means the opposite.
- Referenced bit: 1 means it was either read or written. Used to pick page to evict. Don’t want to get rid of page which is being used.
- Present: 1 and Absent: 0
- Protection bits: r, w, r/w

### Paging problems
- Virtual to physical mapping is done on every memory reference so mapping must be fast.
- If the virtual address space is large, the page table will be large.

### Solution for slow paging: TLB
There are some naive solutions for the slow speed of paging but they are not really useful:
- Bring page table for a process into MMU when it is started up and store it in registers.
- Keep page table in main memory

The better solution is this:
### Translation Lookaside Buffers (TLB)
Adding TLB to MMU, speeds up the address translation by storing frequently accessed frames. If we want to use TLB, beside the refrence bit, present/absent bit, protection bit, etc, we should add a **valid bit** which indicates whether a page is in use or not.
If the address is inside MMU, we don't check page table at all. If not, it refers to page table and finds it. It also puts it in TLB.

#### TLB managmement
It can be done both in hardware and software. If it gets done in software, OS has to handle TLB faults whereas if it's done by hardware, it has to be handled by MMU.
Software can figure out which pages to pre-load into TLB (e.g. Load server after client request) and it also keeps cache of frequently used pages

### Solution for large page table: Multi-level tables
We want to avoid keeping the entire page table in memory because it is too big. We use multiple page tables with different hierarchies.

![Page Table Hierarchy](/photos/hierarchy.png)

- The 32-bit address contains two bits for two page table fields and other bits as offset.
- Top level of page table contains
    - Entry 0 points to pages for program text
    - Entry 1 points to pages for data
    - Entry 1023 points to pages for stack

**There's still another problem:** Multi-level page table works for 32-bit memory Doesn’t work for 64-bit memory because it still gets too big.

### Inverted page table
- Keep one entry per (real) page frame in the “inverted” table.
- Entries keep track of (process, virtual page) associated with page frame.
- Need to find frame associated with (process, virtual page) for **each** memory reference.

![Inverted Table](/photos/invertedtable.png)

#### Searching through page frames efficiently
- Keep heavily used frames in TLB
- If miss, then can use and associative search to find virtual page to frame mapping
- Use a hash table

### Page replacement algorithms
- If a new page is brought in, we need to choose a page to evict but we don't want to remove heavily used pages.
- If page has been written to, we need to copy it to disk. Otherwise, it gets overwritten.

There are many algorithms for page replacement:
- Optimal page replacement algorithm
- Not recently used page replacement
- First-in, first-out page (FIFO) replacement
- Second chance page replacement
- Clock page replacement
- Least recently used page (LRU) replacement
- Working set page replacement
- WSClock page replacement

### Optimal page replacement algorithm
- Pick the one which will be not used for the longest time
- Not possible unless know when pages will be referenced (crystal ball)
- Used as ideal reference algorithm

### Not recently used algorithm
- Use R and M bits
- Periodically clear R bit
    - Class 0: not referenced, not modified
    - Class 1: not referenced, modified (this never happens)
    - Class 2: referenced, not modified
    - Class 3: referenced, modified
- Pick lowest priority page to evict

### FIFO algorithm
- Keep list ordered by time (latest to arrive at the end of the list)
- Evict the oldest (head of the line)
It is easy to implement but the oldest might be most heavily used.

### Second chance algorithm
- Pages are still sorted in FIFO order by arrival time.
- Examine R bit. If it was 0, evict. If it was 1, put the page at end of list and set R to zero.

**But** it might still evict a heavily used page.

### Clock page replacement algorithm
![Clock Page Replacement Algorithm](/photos/clock.png)
When a page fault occurs, the page that the hand is pointing to is inspected. The action taken depends on the R bit;
- If R=0, evict the page
- If R=1, Clear R and advance hand

This algorithm:
- Doesn't use age as a reason to evict page
- Doesn’t distinguish between how long pages have not been referenced


### LRU algorithm
- Approximate LRU by *assuming* that recent page usage approximates long term page usage.
- Could associate counters with each page and examine them but this is expensive.

#### Implementing LRU with hardware:
Associate counter with each page. At each reference increment counter and evict the page with the lowest counter. Implementing LRU with hardware is quite easy.

**How it works:**
Keep a n*n array for n pages. When a page frame, k, is referenced then **all the bits of the k row are set to 1 and all the bits of the k column are set to zero**. At any time the row with the lowest binary value is the row that is the least recently used (where row number = page frame number). The next lowest entry is the next recently used; and so on.

If we have four page frames and access them as follows: 0123210323, it leads to the algorithm operating as follows:

![LRU using hardware](/photos/lruhardware.png)

#### Implementing LRU with software:
We can use software counter instead of harware ones. It implements a system of aging.
![LRU using software](/photos/lrusoftware.png)

**How it works:**
- Consider the (a) column. After clock tick zero, the R flags for the six pages are set to 1, 0, 1, 0, 1 and 1. This indicates that pages 0, 2, 4 and 5 were referenced. This results in the counters being set as shown. We assume they all started at zero so that the shift right, in effect, did nothing and the reference bit was added to the leftmost bit.
This process continues similarly for each clock tick.
- When a page fault occurs, the counter with the lowest value is removed. It is obvious that a page that has not been referenced for, say, four clocks ticks will have four zeroes in the leftmost positions and will have a lower value that a page that has not been referenced for three clock ticks.

