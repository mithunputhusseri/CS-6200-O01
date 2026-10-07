

The paper presents the design of a new web server architecture: Asymmetric multi-process event driven architecture. 


Web server must be structured such that it can overlap the serving of requests for cached content with concurrent disk operations that fetch content not currently cached in main memory


Ibe workloads that exceed capacity of server cache, servers with multi process or multi threaded architectures usually performs best. 

AMPED nearly matched performance of SPED on cached workloads as well matching MP and MPT servers on disk intensive workloads. Flash uses only standard APIs and is therefore
easily portable

In Multi process each server process handles on request at a time. Processes execute the processing stages sequentially
In Multi Threaded model, Server uses a single address space with multiplr oncurrent threads of execution. Each threads handles a request.

Since each process has its own private address space, no sync is necessary to handle the processing of different HTTP requests. However, it may be more difficult to perform
optimization in this architecture that rely on global information, such as shared cache of valid URLs.


The primary difference between the MP and MT architecture, however is that all threads can share gloval variable.  The use of a single shared address space lends itself
to optimization that rely on shared state. However the threads must use some form og sync to control access to shared data.

The MT model requires that operating system supports kernel threads, that is when one thread is blocked on I/O other runnable threads withhin same address space must remain eligible for execution. 


Single process event-driven:

In each iteration, the server performs a select to check for completed I/O events. WHen an I/O event is ready, it completed the corresponding basic step and inititated
the next step associated with the HTTP request, if appropriate.

In priciviple a SPED server is able to overlap CPI, disk and network operations associated with the servering of many HTTP requests, in the context of single process and single thread of control. 


Assymetric multi-process event driven:


AMPED combines the 