# Operating System Concepts

This document explores core OS abstractions that underpin system design, focusing on processes, memory management, I/O operations, and scheduling.

## Processes

A process is an instance of a running program with its own memory space and resources.

### Process States
- **New**: Being created
- **Ready**: Waiting for CPU
- **Running**: Executing on CPU
- **Waiting**: Waiting for I/O or event
- **Terminated**: Finished execution

### Process Control Block (PCB)
Contains:
- Process ID
- Program counter
- CPU registers
- Memory management info
- I/O status

### System Design Implications
- Process isolation for security and stability
- Context switching overhead
- Inter-process communication (IPC) mechanisms

## Memory Management

OS manages physical and virtual memory to provide abstraction and protection.

### Virtual Memory
- **Address Space**: Each process has its own virtual address space
- **Paging**: Divides memory into fixed-size pages
- **Page Tables**: Map virtual to physical addresses
- **TLB**: Translation Lookaside Buffer for fast lookups

### Memory Allocation
- **Stack**: Automatic allocation/deallocation (local variables)
- **Heap**: Dynamic allocation (malloc/free in C, new/delete in C++)
- **Static**: Global variables, compile-time allocation

### System Design Implications
- Memory leaks and fragmentation
- Cache locality and performance
- Memory-mapped files for efficient I/O
- NUMA (Non-Uniform Memory Access) in multi-socket systems

## I/O Models

Different ways applications interact with I/O devices.

### Blocking I/O
- Thread waits for I/O completion
- Simple but inefficient for concurrent operations
- Used in traditional synchronous programming

### Non-Blocking I/O
- Thread continues execution, polls for completion
- Efficient for single-threaded applications
- Can lead to busy-waiting

### Asynchronous I/O
- I/O operations run in background
- Callback or event-driven completion
- Scales well for high concurrency

### I/O Multiplexing (select/poll/epoll)
- Monitor multiple file descriptors
- Efficient for handling many connections
- Used in event-driven servers (nginx, Node.js)

### System Design Implications
- Blocking I/O limits scalability
- Async I/O enables high-throughput servers
- Kernel bypass (DPDK, io_uring) for ultra-low latency

## Scheduling

OS decides which process runs when on available CPUs.

### Scheduling Algorithms
- **FCFS (First Come First Served)**: Simple, but convoy effect
- **SJF (Shortest Job First)**: Optimal for minimizing wait time
- **Round Robin**: Fair time slicing
- **Priority Scheduling**: Higher priority processes first
- **Multilevel Queue**: Different queues for different priorities

### Modern Schedulers
- **CFS (Completely Fair Scheduler)**: Linux default, proportional fairness
- **Real-time Schedulers**: For time-critical tasks
- **NUMA-aware Scheduling**: Considers memory locality

### System Design Implications
- CPU affinity and cache effects
- Real-time requirements for low-latency systems
- Oversubscription and thread contention
- Container scheduling (Kubernetes, Docker)

## Key Takeaways

- Processes provide isolation but incur context switch costs
- Virtual memory enables efficient memory usage and protection
- I/O model choice dramatically affects application scalability
- Scheduling policies balance fairness, efficiency, and responsiveness

## Code Examples

### Process Creation (Java)
```java
ProcessBuilder pb = new ProcessBuilder("ls", "-l");
Process p = pb.start();
int exitCode = p.waitFor();
```

### Memory Mapped File (Java)
```java
FileChannel fc = FileChannel.open(Paths.get("file.txt"));
MappedByteBuffer buffer = fc.map(MapMode.READ_ONLY, 0, fc.size());
```

### Non-Blocking I/O (Java NIO)
```java
SocketChannel channel = SocketChannel.open();
channel.configureBlocking(false);
channel.connect(new InetSocketAddress("example.com", 80));
```

## Resources

- [Operating System Concepts by Silberschatz](https://www.os-book.com/)
- [Linux Kernel Documentation](https://www.kernel.org/doc/)
- [Java NIO Tutorial](https://docs.oracle.com/javase/tutorial/essential/io/)