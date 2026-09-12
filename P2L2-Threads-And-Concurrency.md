1. SIngle process, single execution contex.t

Threads and concrrency:

1. Threads are active unity, threads executre a unit of process.
2. Many threads execute simultaneously
3. Require coordination.
 (sharinf of IO devices, CPUs, memory)

 Process vs Thread
 [Diagram 1](images/Screenshot%202026-09-08%20at%207.22.41 PM.png)

 Benefits of Multi Threading
 1. Parallization speeds up.
 2. Threads may execute completely different portions of code.
 3. Specialization -> hot cache
 4. Better than Multi process alternative, because it has less memory requirements. Synchrinizing data between process is difficult. but its easier in threads.


 Benefits of MUlti Threading: Single CPU
 1. If t_idls > 2*t_ctx_switch (Then make sense to have multi threads.)
 2. Time to context between threads is faster than time to context between process.
 3. Hence multi thread is useful if you want to hide the idle time. Useful even in a single CPU.



 Benefits to OS as well:
 1. Threds working on behlaf og apps.
 2. OS-level services like daemon or drivers.


 What do we need to support threads:
 1. Data strcture to track of resource usage
 2. Mecahnis to create and manage threads.
 3. MEchanis to safely coordinate among threads. Running concurrentlyy in the same address space.

 In case of process, OS ensure no memory is shared between process.


 But in case threads, both threads t1 and t2 use same virtual memory and shared addreess space. If both threads hence try to access same data
 it causes problems. 

 1. Mutual exclusions:
    1.  Exclusive access to only one thread at a time. 
    2. Mutex
    3. Conditional variables.


Thread creation:

Thread type data strucutre (Thread ID, PC, SP, registers, stack, attributes)

Thread creation:
   Fork (proc,args) (not same as UNIX fork)
    proc: procedure that the created threads will execute
    args: Argumens of the procedure

   Join(thread): Join will send back the result of the child process back to parent.


Thread Creation examples


Thread thread1:
shared_list list;
thread1 = fork(safe_insert,4)
safe_insert(6)
join(thread1);
 in this case the order can be either 4,6 or 6,4.


 Mutexes

 How is the list updates

 Create new list elemeent e
 set e.value = X
 read list and list.p_next
 set e.pointer = list.p_next
 set list.p_next = e

 What happens if 2 threads try to execute at the same time, and try to set different values in the p_next field. Only one will be succesful and other will be lost


Mutual Exclusion:

Mutex: Mutext is like a lock, when a thread tries to use a shared data. Mutex should contian:  status, owners, blocked threads

only one thread will have access. Portion of thread protected is called critical section. It could be a counter, list. Or any king of operation that requires
mutual exclusion. Critical sections can only be accessed one by one.
[Diagram 2](images/Screenshot%202026-09-08%20at%208.02.44 PM.png)


Produces/Consumer Example:
WHat is the procesing you wish to perform with Mutex needs to occur onky under certain conditions


Condition Variable:
[Diagram 3](images/Screenshot%202026-09-08%20at%208.02.44 PM.png)

Condition variable API

wait(mutex,cond)
-mutex is automtically release and re-acquired on wait

signal(cond)
 -notify only one thread waiting on condition
broadcast(cond)
 -notify all waiting threads
 Condition vairable (list of waiting threads, mutex ref associated with the condition)

 wait(mutex,cond){
    //automatically release mutex
    //and go on wait queue

    //wait
    //remove from queue
    //re-acquire mutex
    //exit the wait operation.
 }

 Reader/Writer Problem

 1. at any point o or more threads can read a resource
 2. o or 1 writer can write on a resource.

 One solution to use Mutex lock. But this is too restrictive.
They have a binary state 1,0. Perfect for writers, but for readers can be restricted.

state of shared file/resource

. free: resource counter = 0
. reading: resource counter >0
. writing: resource counter = -1


READERS 

Lock(counter_mutex){
    while(resource_counter == -1)
        Wait(counter_mutex,read_phase)
    resource_counter++
}//unlock

// read data
Lock(counter_mutex){
    resource_counter--;
    if(resource_counter ==0)
        Signal(write_phase)
}//unlock


// WRITER
Lock(counter_mutex){
    while(resource_counter !=0)
        Wait(counter_mutex,write_phase)
    resouce_counter = -1
}
 
 // write data
Lock(counter_mutex){
    resource_counter=0
    Broadcast(read_phase)
    Signal(write_phase)

}


General format

Lock(mutex){
    while(!predicate_indicating_Access_ok){
        Wait(mutex,cond_var)
    }
    updaed state => update predicate
    signal and/or broadcast
    (cond_Var with correct waiting threds)
}//unlock


Critical Section strcuture with procy variable




Avoid common mistake
- Keep track of mutex/cond variables used witha resource
- CHeck that you are always using the lock and unlock.
    . Some compilters give error
-Use a single mutext to access a single resource.
- Check that you are signaling the correct conditions.
- CHeck that you are not using signal when broadcast is needed.
- Signal only 1 thread will procees, remaining threads might continue to wait indefinitelly
- THreads excutions order not controlled by order of giveing signals to condition variable.



Spurious wake ups:
eg:
//WRITER
Lock(counter_mutex){
    resource_counter = 0;
    Broadcast(read_phase)
    Signal(write_phase)
}

//READER
Wait(counter_mutex, write/read_phase)
spurios wake uo: when we wake up threads up knowing they may not be able to proceed.


to avoid spurious wake up

//NEW WRITED
Lock(counter_mutex){
    resource_counter = 0;
}
Broadcast(read_phase)
Signal(write_phase)


DeadLock introduction:

1. Two or more competing threads are wiating on each other to complete, but none of them ever do.
[Diagram 4](images/Screenshot%202026-09-09%20at%204.14.10 PM.png)


1. Get all locks upront, then release at the end. Use one mega lock (too restricitve)

1, Maintain lock order. (If there is a cyclic dependedncy, And it caises deadlock) (potential challenge is in complex program it is hard to find.)

In summary:

1. A cycle in wait graph is neceassary and sufficient for a deadlock ot occuer,

How to peresent.
    1. Deadlocl preveention (expensive)
    2. Deadlock detection and recovery: Rollback
    3. Apply the ostric algorithm.. (DO nothing)


Kernel vs user level threads.

1. Kernel level
    1. kernel level threads implies OS is multi threaded.
    2. Manage by kernesl level components like scheduler
2. User level
    1. Process itslef is multi threased.
    2. It should be associated with a kernel level thread.


One ot one model
1. Each user thread has a kernel level thread. (OS sees, understand, synchronize, blocking)
2. Downside, must go to OS for all operations, OS may have loimits on policies, and portability is a prbem.

Many to one model
1. Many threads are handled by the OS. There will be a OS level thread manager that managers it
2. Totally portable, everything will be done at user level thread manager. (dont have to make system calls, hence less expensive)
3. OS has no insights into application needs
4. OS may block entire process, if onse user level thread blocks on IO.

Many to manys model
1. It can be best of both models.
2. It can havebound ot unbounded threads.

1. Requires coordination between user and kernel level thread managers.


Kernel vs user level threads

1. Kernesl level
    1. System scope:
    system wide thread management by OS=-level thread managers ()

2. User level
    1. Process scope
    user level libarwary managers threads within a single process.



Multi threading patterns:

Boss-worker patern: 