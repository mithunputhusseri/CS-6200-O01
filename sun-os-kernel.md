

1. Kernel threads use synchronization primitives taht support protocl for preveneting priority inversion, so a threads priority is determined by which activities it is impeding
by holding locks as well as by the service it is performing

SunOS kernel uses kernel threads to provide aynchronous kernel activity, such as async writes to disk, serviving STREAMS queues and callout

Even interrupts are handled by kernel threads, If an interrupt thread encounters a locked sync variable it blocks and allows the critical section to clear


A major feature of the new kernel is its support of multiple kernel-suuported threads of control called lightweight processes(LWPs), in 

Separating user-levelt threads from LWP allows the user thread library to quickly switch between user threads without entering kernel.


LWP strucutre contains per-LWP data such as PCB for storing user lvel prcoessero registers, system call arguments, signal handling masks, resource usafe infromation and profiling pointers.
 It also container pointer to the associated kernel threads and process strcutures.


 Kernel thread structures contiains kernel registers, scheduling class, dispatch quue link and pointers to the stack and the assicuated LWP, process and CPU structures.


 The thread strucutre is not swapped, so it aslo contains some data associated with the LWP that i sneeded when the LWP strcuture is swapped.

 Thread strcutures are linked on a list of threads for the process and also on a list of all existing threads in the sysytem.


 Per-processor data is kept in the cpi structure, which has pointers to the currently executing thread, the idle thread for that CPI and current dispatching and interrupt handling information.


 Kernel thread scheduling

 With addition of multi threading, the scheduling classes and dispatcher operate on threads instead of processes. The scheduling classess currently supported are system time sharng and real-time(fixed-priority)


 The dispatcher chooses the thread with the greates global priority to tun on the CPU. If more than one threwad has the same priortiy they are dispatched in round-robin order.

 Kernedl has been made preemptible to better support real-time class and interrupt threads, preemption is disabled only in a small number of bounded sections of code.


 This means that a runnable threads runs as soon as is practical afterr its priority become high enough


 System threads.

 System threads can be created for short or long-term activitirs. They are scheduled like any other thread, but usually belong to the system scheduling class. These threads have not need for LWP structures. 


 Kernel implements the same synchronization objects for internal use as are provided by user-level libraries for use in amulithreaded application programs. 

 Kernel thread synchronization interfaces are similar(mutexes, condition variable, signal, broadcast, multi reader, single writer locks, counting semapshores)

 These are all implemented such that the behavious of the synchromnization object is spcoefied when it is initialized. 

 Most of sych objected has types that enable collecting statistcs such as blocking counts or times. A patchable kernel variable can also set the default types to enable statistics gathering. 

 THe semantics of most of the sync primitives cause the calling thread to be prevented from progressing pas the primitive until some condition is satisified. The way in which further progress is impeded is a function of the initialization. By default, the kernel thread sync primitives that can logically block can poterntially sleep.

 Some of the syn primitives are strictly bracketing( eg, threads that locks a mutex mus be thr threads that unlocks it)


 Default blocking policy for mutexes called adaptive (spins while the owner of the lck reaming running on a processoe) This is done ny polling the owners status in the spin wait loop. If the owner ceases to run, the caller stops spinning and sleeps. This gives fast response and low overhead for simple contentions. 


 MUTEX_SPIN: WHich takes as its type specific argumentm the interrupt level to be disabled while the mutex is hld. 

 MUTEX_DRIVER: Device drivers are restricted to using this type,  Which takses a sub-DDI defined opaque values as an argument,




 Each sync object requires a way of finding threads that are suspended waiting for that object. It has to be small. Because many system structure contain sync objects. So the

 queue header is not directly in th eobject. Instead two bytes of sync objects are used find a turnstile structure contaiining the sleep queue  header and priority inheritence  info.


 Turnstiles are preallocated such that there are always more turnstiles than the number of threads active.

 Another approach is to have a hashmap storing, but turnstile approach is favoured for more predictable real-time behavious since they are neved shared by other locks as hashed sleep queues sometiem are.


 INterrupts as threads:

 Drawbacks of traditional interrupt handling
 1. Deadlock caused i the interrupt handled using the same sync object
 2. Raising and loweing or priority for fixing the above is expensive.
 3. In modular kernel such as sunOS, these request can come from interrupt handler can involve many kernel subsystem. This in turn means that the mutexes used in many 



 To avoid the dwabacks. The sunOS 5.0 kenel treats most interrupts as asyn cteaed and dispatched high-priorityn threads. This enabled these interrupt handlers to skeep, if required and to use the standard sync primitives.

 For practically, Interrupts threads are preallocated and partially initializalized. Hence when an interrupt occurs only minimum amount of work needs to be done. At this point it not yet a fully fledged thread is pinned until the interrupt thread reutnes or block and cannot proceed on another CPI. When interrupt return, we restore the state pf the interrupted thread and return.

 If an interrupt thread block on a sync variableit saves state to make it a full fledged thread capable of being run by a CPU. And then retuens to the pinned threads. Thus most of the overhead of creating a full thread is only done when the interrupt must block due to contention.

 While an interrupt thread is in progress the interrupt level it is handling and all lower priority interrupts must be blocked. This is handled by the normal interrupt priority mechanism unless th thread vlocks. 


 Clock interrupts:

 1. There is only one clock interrupt thread in the system.(not one per CPU)
 2. ANd the clock interrupt handler invokes the clock thread only ig it is not already active.

This occurs rarely in practice, when it occurs it is usally due to heavy activity at higher interrupt levels. It can also occur during debugging.


Kernel Locking strategy. Every piece of shared data is protected by a sync object. 

Non-MT driver support:  Some drivers have been modified to protect themsleces against concurreny in a MT env. So a wrapper is added this ensures only one such driver will be active at any one time.

Implementation technology

1. Kernel time slicing: Added code to the clock interrupt handler to preempt whatever thread was interrupted. This allows even a uniprocessero to have almost arbitrary code interleaving.
Increasing clock interrupt rate made this even more valuable.

2. Lock hierarchy violation detection: Intead of establishing a system lock hierarchy a priori. They developed a static analysis tool that would check lock order violation in the ysttem. Its called locknest.


Deadlcok delection

1. A side benefit of the priority inheritance mechanis, is that deadlocks caused by hierarchy violations are usually deletected at run time as well. It does a good job on mutexes and readers/writer lock held for write.  But since there isn't a complete lust of threads holding a read lock. It cant always find deadlocks involving reader/writer locks. there are other deadlock possible with CVs. That are  detected
