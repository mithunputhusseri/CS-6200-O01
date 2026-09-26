

Which model is better.

1. Boss worker model
2. Pipeline model

Ppeline has advantage in case of shorted total execution time.

Avg execution time: is worse for pipeline stage.

Threads are useful because og
1. Parellilzation:
2. Specialization -> hot cache
3. Efficiency -> lower memory req and cheaper aync

Threads hide latency of i/o operations.


Thread performance considerations:


Performance metrics intro:

metrics = a measurament standard.
measurable and/or quantifiable property.

exampleS: execution time,throughput, request rate, cpu utilization, wait time, throughput, platform efficiency, performance/$, performance/w. 
percentage of SLA violation. 
Metrics cab be used to evaluate the system behaviour.


Performanc emetrics:

Measurable Quantirty:
Obtain from experiments with real software deploymnt, real machines, real workloads, if its not possivle, toy experiments , representative of realistic setting(simulation).

We refer these experiments settings as TESTBED


Are threads really useful.

1. Depends on metrics
2. Depends on workload.


DIfferent number of toy orders -> different implementation of toy shop

different type of graph -> different shortest path algorithm.
different file patterns -> diffrent file system.

Multi process vs Multi threaded:

1. How to best provide concurrency. 

MUlti threads vs multi process. 

In context of a web server(it is important for concurrent)

Steps in simple webs erver
1. Client/browser send request
2. Web server accepts request
3. server processing steps.
4. respond by sending file.

![Diagrams](images/Screenshot%202026-09-17%20at%205.14.20 PM.png)

Process
Acept conn -> read request -> parse request -> find file -> computer header -> send header -> read file send data.

One way to spd is to run the above in multi process

1. SImple programming
2. Many processes, high mmory usage, costly context statge, hard/cost to maintain shared state. Tricky port setup.


Alternative multi threaded web server (Multi threaded.)

[Diagems](images/Screenshot%202026-09-17%20at%205.17.37 PM.png)

1. SHared adress space
2. SHared state
2. cheap context switch

1. Not simple implementation
2. Requires sync
3. Underlying OS need support for threads (not an issue today)


Event -driven model:

1. single adress space
2. single process
3. single thread of control.

[Diagrams 3](images/Screenshot%202026-09-17%20at%205.20.12 PM.png)

Dispatcher == state machine call handler == jump to code. Events can be 
1. receipt of request
2. completion of send
3. COmpletion of disk read.

Handler 
- run to completion
- If they need to block (initiation blocking operation and pass control to dispath loop)


Concurrent execution in evet dirver model. ( MP and MT ( 1 request per execution context)

Event -driver (many request interleaved in a execution contextt)

WHy does this work: (event driven model)
1 CPU ( threads hide latency ) If there are no idle time. context switching just waste cycles.

In event driver
1. Process requedt until wait neceassary, then switch to another request.

Multi CPUs => mutli event driver process.


HO does this works

sockets(network) 
files (disk)

Internally both of the above are described by file descriptor(fd)

event  ==  input on file descriptor

which file descriptor. Below api can be used to find the FD
    - select() 
    - poll()
    - epoll()

- SIngle address space
- single flow of controll
- smaller memory requirements, no context switching
- no syncs.

Problem with event-driven model

1. A blocking requrst/handler will block the entire process.
2. Async I/O operations.
 -process/thread make system call
 -OS obtains all releavant info from stack and wither learns where to return results or tells caller where to get results later.
 -process/thread can continue

 requires support from kernel (eg threads)(and/or device)
 Fits nicely with event-driven model.

 What is sync calls are not available

 Helpers
  -designated for blocking ii/o operations only
  - pip/socker based comm w/ event dispatched
  -> select/poll still ok
  ![Diagram 7](images/Screenshot%202026-09-17%20at%205.57.17 PM.png)
  Asymmetric multi-process event-driven model(AMPED)
  Asymmetric multi-thread event-driven model(AMTED)

  1. Resolved portability limitation of basic even-driven model
  2. SMaller footprint than regular worker thread.

  1. Applicability to certain classed of applications
  2. Event routing on multi CPU system.


Event-driver model requires least amount of memory (only require extra memory for helper threads for concurrent blocking I/O)


Flash web server:
