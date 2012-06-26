!SLIDE title-page
## CPU Caches
.notes More and more of our runtime environment is becoming virtual, but it is important as engineers for us to understand the low level details of hardware for our software to run optimally.  My goal is help everyone understand how each physical level of caching within a CPU works.  I've capped the talk below virtual memory (memory management for multitasking kernels), as that is specific to the operating system running on the machine.

Jamie Allen

jamie.allen@typesafe.com

@jamie_allen

<img src="typesafe-logo-081111.png" class="illustration" note="final slash needed"/>

!SLIDE transition=blindY
# Data Locality
.notes All algorithms are dominated by that - you have to send data somewhere to have an operation performed on it, and then you have to send it back to where it can be used.  With spatial, what's important is that data which is required together is located together.  The JVM does not guarantee this for object instances, such as fields in a class.

* Spatial - reused over and over in a loop, data accessed in small regions
* Temporal - high probability it will be reused before long


!SLIDE transition=blindY
# The Old and Busted Architecture
.notes Used to be that every socket had its own infrastructure that communicates with RAM via the frontside bus and northbridge.

* Frontside Bus
* Northbridge
* Image by Alexander Taubenkorb, Wikimedia Commons

<img src="300px-Schema_chipsatz.png" class="illustration" note="final slash needed"/>

!SLIDE transition=blindY
# The New Hotness Architecture
.notes New machines today are SMP, a multiprocessor architecture where two or more identical processors are connected to a single shared main memory and controlled by a single OS instance.  Since Intel's Nehalem architecture, each CPU socket controls a portion of RAM, and no other socket has access to it directly.  2011 Sandy Bridge and AMD Fusion integrated Northbridge functions into CPUs, along with processor cores, memory controller and graphics processing unit. So components are closer together today than depicted in this picture.

* Symmetric Multiprocessor (SMP)
* No more Frontside Bus or Northbridge
* Image by Khazadum, Wikimedia Commons

<img src="smp.png" class="illustration" note="final slash needed"/>


!SLIDE transition=blindY
# Memory Wall
.notes Gil Tene, CTO of Azul has a good presentation about this concept from a paper in 1994 on their website.  Each Intel core has 6 execution units able to perform work during a cycle (loading memory, storing memory, arithmetic, branch logic, shifting, etc), but they need to be fed data very fast to take advantage of it. Note that it didn't happen because of RAM, but the Application Memory Wall does exist in other forms, like GC pauses - applications designed to minimize GC can hit the memory wall, but those are highly specialized (like those using the Disruptor).

* CPUs are getting faster
* Memory isn't, and bandwidth is increasing at a much slower rate
* It was predicted that applications would become memory-bound by now
* Didn't happen

!SLIDE transition=blindY
# Non-Uniform Memory Access (NUMA)
.notes NUMA architectures have more trouble migrating processes across cores.  Initially, it has to look for a core with the same speed accessing memory resources and enough resources for the process.  If none are found, it then allows for degradation.  NUMA is not the same as multiple commodity machines because they don't have a shared address space.  Thread migration across sockets is extremely expensive because the shared L3 cache isn't warmed.

* Access time is dependent on the memory locality to a processor
* Memory local to a processor can be accessed faster than memory farther away
* The organization of processors reflect the time to access data in RAM, called the NUMA factor.

!SLIDE transition=blindY
# Latency Numbers Everyone Should Know
.notes The problem with these numbers is that while they're not far off, the lowest level cache interactions (registers, store buffers, L0, L1) are truly measured in cycles, not time.

	L1 cache reference ......................... 0.5 ns
	Branch mispredict ............................ 5 ns
	L2 cache reference ........................... 7 ns
	Mutex lock/unlock ........................... 25 ns
	Main memory reference ...................... 100 ns             
	Compress 1K bytes with Zippy ............. 3,000 ns  =   3 µs
	Send 2K bytes over 1 Gbps network ....... 20,000 ns  =  20 µs
	SSD random read ........................ 150,000 ns  = 150 µs
	Read 1 MB sequentially from memory ..... 250,000 ns  = 250 µs
	Round trip within same datacenter ...... 500,000 ns  = 0.5 ms
	Read 1 MB sequentially from SSD* ..... 1,000,000 ns  =   1 ms
	Disk seek ........................... 10,000,000 ns  =  10 ms
	Read 1 MB sequentially from disk .... 20,000,000 ns  =  20 ms
	Send packet CA->Netherlands->CA .... 150,000,000 ns  = 150 ms

	Shamelessly cribbed from this gist: https://gist.github.com/2843375

!SLIDE transition=blindY
# Current Processors
.notes Westmere was the 32nm die shrink of Nehalem.  Ivy Bridge uses 22nm.  I'm ignoring Oracle SPARC here, but note that Oracle is supposedly building a 16K core UltraSPARC 64.  Current Intel top of the line is Xeon E7-8870 (80 cores, 32MB L3, 4TB RAM), but it's a Westmere-based microarchitecture.  The E5-2600 Romley is the top shelf new Sandy Bridge offering - it's specs don't sound as mighty (8 cores per socket, max 20MB L3, 1TB RAM).  Don't be fooled, the Direct Data IO makes up for it as disks network cards can do DMA (Direct Memory Access) from L3, not RAM.  Also has two memory read ports, whereas previous architectures only had one and were a bottleneck for math-intensive applications.  Sandy Bridge is 4-30MB (8MB is currently max for mobile platforms, even with Ivy Bridge), includes the processor graphics.  Sandy/Ivy Bridge and Bulldozer support the new AVX (Advanced Vector Extensions) instruction set of x86.

* Intel
	* Nehalem/Westmere
	* Sandy Bridge
	* Ivy Bridge - not on servers yet
* AMD
	* Bulldozer

!SLIDE transition=blindY
# Sandy Bridge Microarchitecture

<img src="sb_microarch.png" class="illustration" note="final slash needed"/>

!SLIDE transition=blindY
# CPU or Instruction Cycle 
.notes Sometimes called the fetch and execute cycle, or fetch-decode-execute cycle, or FDX.  Process by which a computer retrieves a program instruction from memory, determines what actions the instructions require, and carries out those actions.

* Basic operation cycle of a computer
* Equates to roughly 1/3 of a nanosecond
* Sandy Bridge has 2 load/store operations for each memory channel

!SLIDE transition=blindY
# Registers
.notes Sandy Bridge has 168 Integer/FP

* On-core
* Can be accessed in a single cycle
* A 64-bit Intel Nehalem CPU had 128 Integer registers, 128 floating point registers

!SLIDE transition=blindY
# Load/Store Buffers
.notes Store buffers disambiguate memory access and manage dependencies for instructions (loads and stores) occurring out of program order. CPUs typically have load and store buffers which are associative queues of separate load and store instructions that are outstanding to the cache which can be snooped to preserve program order.   PRF allowed Intel to lessen amount of data kept on-die.

* Nehalem passed operands with each instruction
* Sandy Bridge uses the Physical Register File to store operands
* Being off-die creates a larger window for Out of Order (OoO) execution
* ~1 cycle

!SLIDE transition=blindY
# Sandy Bridge PRF

<img src="prf.jpg" class="illustration" note="final slash needed"/>

!SLIDE transition=blindY
# SRAM
.notes The number of circuits per datum means that SRAM can never be dense.

* Requires 6-8 pieces of circuitry per datum, cannot be dense
* Runs at a cycle rate, not quite measurable in time
* Uses a relatively large amount of power for what it does
* Data does not fade or leak, does not need to be refreshed/recharged

!SLIDE transition=blindY
# L0
.notes Macro ops have to be decoded into micro ops to be handled by the CPU.  Nehalem used L1 for macro ops decoding and had a copy of every operand required, but Sandy Bridge caches the uops (Micro-operations), boosting performance in tight code (small, highly profiled and optimized, can be secure, testable, easy to migrate).  Not the same as the older "trace" cache, which stored uops in the order in which they were executed, which meant lots of potential duplication.  No duplication in the L0, storing only unique decode instructions.  Hot loops are those where the program spends the majority of its time.  Theoretical size of 1.5Kuops, effective utilization possibly much lower.  Hot loops should be sized to fit here.

* A decoded cache of the last 1536 uops (~6kB)
* Well-suited for hot inner loops
* Only pointers to operands in a uop are sent for processing on Sandy Bridge
* When in use, the CPU puts the L1 and decoder to "sleep" to save energy and minimize heat.

!SLIDE transition=blindY
# L1
.notes Instructions are generated by the compiler, which knows rules for good code generation.  Code has predictable patterns and CPUs are good at recognizing them, which helps pre-fetching.  Spatial and temporal locality is good.  Nehalem could load 128 bits per cycle, but Sandy Bridge can load double that because load and store address units can be interchanged - thus using 2 128-bit units per cycle instead of one.  Write through to L2 used by Bulldozer (AMD)

* Divided into data and instructions
* 32K data, 32K instructions per core on a Sandy Bridge
* 16KB data per cluster, 64KB instruction per core
* 3-4 cycles to get to L1d
* Uses "write through" back to L2 on Bulldozer

!SLIDE transition=blindY
# L2
.notes For random access, when the working set size is greater than the L2 size, cache misses start to grow.  Caches from L2 up are "unified", having both instructions and data (not in terms of shared across cores).

* 256K per core on a Sandy Bridge
* 2MB per "module" on AMD's Bulldozer architecture
* 14 cycles
* Unified data and instruction caches from here up

!SLIDE transition=blindY
# L3
.notes where concurrency takes place, data is passed between cores on a socket.

* Was a unified cache up until Sandy Bridge, shared between cores
* Became known as the Last Level Cache (LLC)
* Since no longer unified, any core can access data from any of the LLCs until Sandy Bridge
* Sandy Bridge allows only exclusive access per core
* Varies in size with different processors and versions of an architecture

!SLIDE transition=blindY
# Exclusive versus Inclusive
.notes In an exclusive cache (AMD), when the processor needs data it is loaded by cache line into L1d, which evicts a line to L2, which evicts a line to L3, which evicts a line to main memory; each eviction is progressively more expensive.  In an inclusive cache (Intel), eviction is much faster, but requires a larger L2.

* Only relevant below L3
* AMD is exclusive, progressively more costly
* Intel is inclusive, requires larger L2

!SLIDE transition=blindY
# Memory Controller
.notes Memory controllers were moved onto the processor die by AMD beginning with their AMD64 processors and by Intel with their Nehalem processors.

* Handles communication between the CPU and RAM
* Contain the logic to read to and write from RAM
* Integrated Memory Controller

!SLIDE transition=blindY
# Intra-Socket Communication
.notes Components internal to a CPU using the ring include the cores, the L3, the system agent and the graphics controller.

* Sandy Bridge introduced a ring architecture so that components inside of a CPU can communicate directly
* L3 is no longer unified
* The ring architecture in the Sandy Bridge has 4 rings in it, for data, request, ACK and snooping
* The ring can select the shortest path between components for speed

!SLIDE transition=blindY
# Inter-Socket Communication
.notes GT/s is gigatransfers per seconds - how many times signals are sent per second.  Not a bus, but a point to point interface for connections between sockets in a NUMA platform, or from processors to I/O and peripheral controllers.  DDR = double data rate, two data transferred per clock cycle.  HT could theoretically be up to 32 bits per transmission, but hasn't been done yet that I know of.  HT can be used for more than linking processors - has applications in as well.  Processors "snoop" on each other, and certain actions are announced on external pins to make changes visible to others.  The address of the cache line is visible through the address bus.  SB can transfer on both the rise & fall of the clock cycle and is full duplex.

* Uses Quick Path Interconnect links (QPI, Intel) or HyperTransport (AMD) for:
	* cache coherency across sockets
	* snooping
* QPI is proprietary to Intel.  Maximum speed of 8 GT/s
* HyperTransport was developed by a consortium of AMD, NVidia, Apple, etc. Maximum speed of 6.4 GT/s (that I know of)
* Both are DDR and point to point interfaces to other CPUs in a NUMA setup
* Both transfer 16 bits per transmission in practice, but Sandy Bridge is really 32

!SLIDE transition=blindY
# MESI+F Cache Coherency Protocol
.notes Request for Ownership (RFO): a cache line is held exclusively for read by one processor, but another requests for write.  Cache line is sent, but not marked shared by the original holder.  Instead, it is marked Invalid and the other cache becomes Exclusive.  This marking happens in the memory controller.  Performing this operation in the last level cache is expensive, as is the I->M transition.  If a Shared cache line is modified, all other views of it must be marked invalid via an announcement RFO message.  If Exclusive, no need to announce.  Processors will try to keep cache lines in an exclusive state, as the E to M transition is much faster than S to M.  RFOs occur when a thread is migrated to another processor and the cache lines have to be moved once, or when two threads absolutely must share a line; the costs ar a bit smaller when shared across two cores on the same processor.  MESI transitions cannot occur until all processors acknowledge a coherency message.  Collisions on the bus can happen, latenc can be high with NUMA systems, sheer traffic can slow things down - all reasons to minimize traffic.  Since code almost never changes, instruction caches do not use MESI but rather just a SI protocol.  If code does change, a lot of pessimistic assumptions have to be made by the controller.

* Modified, the local processor has changed the cache line, implies only one who has it
* Exclusive, only one processor is using the cache line, not modified
* Shared, multiple processors are using the cache line, not modified
* Invalid, the cache line is invalid (unused)
* Forward, to another socket via QPI
* Each QPI message takes ~20NS

!SLIDE transition=blindY
# DRAM
.notes One transistor, one capacitor.  Reading contiguous memory is faster than random access due to how you read - you get one write combining buffer at a time from each of the memory banks, 33% slower.  240 cycles to get data from here.  E7-8870 (Nehalem) can hold 4TB of RAM

* Very dense, only 2 pieces of circuitry per datum
* Refresh is just another read operation where the result is discarded & blocks access
* DRAM "leaks" its charge, but not sooner than 64 milliseconds
* Every read depletes the charge, requires a subsequent recharge
* Memory Controllers can "refresh" DRAM by sending a charge through the entire device
* Uses a prefetch buffer for fast access to data words in a physical row 
* Takes 240 cycles (100 NS) to retrieve from here

!SLIDE transition=blindY
# DDR3 SDRAM

* Double Data Rate, Synchronous Dynamic
* Has a high-bandwidth three-channel interface
* Also reduces power consumption over DDR2 by 30%
* Data is transferred on the rising and falling edges of a 400-1066 MHz I/O clock of the system
* DDR4 is coming

!SLIDE transition=blindY
# Cache Write Strategies
.notes Write through caching: immediately write to main memory after cache line changed. Slow, but predictable. Lots of FSB traffic.  Write back caching: mark flag bit as dirty, when evicted the processor notes and sends cache line back to main memory, instead of just dropping it; have to be careful of false sharing and cache coherency.  Write combining caching: used by graphics cards, groups writes to main memory for speed.  Uncachable: dynamic memory values that can change without warning.  Used for commodity hardware.

* Write through
* Write back
* Write combining
* Uncachable

!SLIDE transition=blindY
# Cache Lines
.notes A dirty cache line is not present in any other processor's cache; clean copies of the same cache line can reside in arbitrarily many caches. A cache line that has been modified has a dirty flag set to true, set to false when sent to main memory.  Be careful about what's on them, because if a line holds multiple variables and the state of one changes, coherency must be maintained.  Kills performance for parallel threads on an SMP machine.  Memory data is transferred to caches in 64 bit blocks, while a cache line is typically 64 bytes, therefore 8 transfers per cache line.  The blocks of a line arrive at different times, ~every 4 CPU cycles.  If the word in the line required is in the last block, that's about 30 cycles or more after the first word arrives.  However, the memory controller is free to request the blocks in a different order.  The processor can specify the "critical word",  and the block that word resides in can be retrieved first for the cache line by the memory controller.  The program has the data it needs and can continue while the rest of the line is retrieved and the cache is not yet in a consistent state, called "Critical Word First & Early Restart.". Note that with pre-fetching, the critical word is not known - if the processor request the cach line while the pre-fetching is in transit, it will not be able to influence the block ordering and will hav to wait until the critical word arrives in order.  Position in the cache line matters, faster to be at the front than the rear.

* Most commonly 64 contiguous bytes, can be 32-256.  
* Look out for false sharing
* Padding can be used to ensure unshared line
* Position in the line matters
* @Contended annotation coming?

!SLIDE transition=blindY
# Striding & Pre-fetching
.notes Doesn't have to be contiguous in an array, you can be jumping in 2K chunks without a performance hit.  As long as it's predictable, it will fetch the memory before you need it and have it staged.  Pre-fetching happens with L1d, can happen for L2 for systems with long pipelines.  Hardware prefetching cannot cross page boundaries, which slows it down as the working set size increases. If it did and the page wasn't there or valid, th OS would have to get involved and th program would experience a page fault it didn't initiate itself.  Temporally relevant data may be evicted fetching a line that will not be used.

* Predictable memory access is really important
* Hardware prefetcher on the core looks for patterns of memory access
* Can be counter-productive if the access pattern is not predictable

!SLIDE transition=blindY
# Cache Misses
.notes The pre-fetcher will bring data into L1 for you.  The simpler your code, the better it can do this.  If your code is complex, it will do it wrong, costing a cache miss and forcing it to lose the value of pre-fetching and having to go out to an outer layer to get that instruction again.  Cache eviction is usually LRU; with more associativity (from more cores), the cost of maintaining the list is more expensive.

* Cost hunderds of cycles
* Keep your code simple

!SLIDE transition=blindY
# Hyperthreading
.notes Hyper threading shares all CPU resources except the register set.  Intel CPUs limit to two threads per core, allows one hyper thread to access resources (like the arithmetic logic unit) while another hyper thread is delayed, usually by memory access.  Only more efficient if the combined runtime is lower than a single thread, possible by overlapping wait times for different memory access that would ordinarily be sequential.  The only variable is the number of cache hits.  Note that the effectively shared L1d (& L2) reduce the available caches and bandwidth for each thread to 50% when executing two completely different code (questionable unless the caches are very large), inducing more cache misses, therefore hyper threading is only useful in a limited set of situations.  Can be good for debugging with SMT.  Contrast this with Bulldozer's "modules" (akin to two cores), which has dedicated schedulers and integer units for each thread in a single processer.

* Great for I/O-bound applications
* If you have lots of cache misses
* Doesn't do much for CPU-bound applications
* You have half of the cache resources per core

!SLIDE transition=blindY
# Programming Optimizations
.notes Short lived data is not cheap, but variables scoped within a method running on the JVM are stack allocated and very fast.  Think about the affect of contending locking.  You've got a warmed cache with all of the data you need, and because of contention (arbitrated at the kernel level), the thread will be put to sleep and your cached data is sitting until you can gain the lock from the arbitrator.  Due to LRU, it could be evicted.  The kernel is general purpose, may decide to do some housekeeping like defragging some memory, futher polluting your cache.  When your thread finally does gain the lock, it may end up running on an entirely different core and will have to rewarm its cache.  Everything you do will be a cache miss until its warm again.  CAS is better, an optimistic locking strategy.  Most version control systems use this.  Check the value before you replace it with a new value.  If it's different, re-read and try again.  Happens in user space, not at the kernel, all on thread.  Algos get a lot harder, though - state machines with many more steps and complexity.  And there is still a non-negligible cost.  Remember the cycle time diffs for different cache sizes; if the workload can be tailored to the size of the last level cache, performance can be dramatically improved.  Note that for large data sets, you may have to be oblivious to caching.

* Stack allocated data is cheap
* Pointer interaction - you have to retrieve data being pointed to, even in registers
* Avoid locking
* Match workload to the size of the last level cache

!SLIDE transition=blindY
# What about Functional Programming?
.notes Eventually, caches have to evict.  Cache misses cost hundreds of cycles.

* Have to allocate more and more space for your data structures.  
* When you cycle back around, you get cache misses

!SLIDE transition=blindY
# Data Structures
.notes  If n is not very large, an array will beat it for performance.  LL and trees have pointer chasing which are bad for striding across 2K cache pre-fetching.  Java's hashmap uses chained buckets, where each hash's bucket is a linked list.  Clojure/Scala vectors are good, because they have groupings of contiguous memory in use, but do not require all data to be contiguous like Java's ArrayList.  Fastutil is additive, no removal, but that's how I'm currently using Riak, too.

* BAD: Linked list structures and tree structures
* BAD: Java's HashMap uses chained buckets!
* BAD: Standard Java collections generate lots of garbage
* GOOD: Array-based and contiguous in memory is much faster
* GOOD: Write your own that are lock-free and contiguous
* Fastutil library

!SLIDE transition=blindY
# Application Memory Wall & GC
.notes We can have as much RAM/heap space as we want now.  And requirements for RAM grow at about 100x per decade.  But are we now bound by GC?  You can get 100GB of heap, but how long do you pause for marking/remarking phases and compaction?  Even on a 2-4 GB heap, you're going to get multi-second pauses - when and how often?  IBM Metronome collector is very predictable. Azul around one millisecond for phenomenal amounts of garbage.

* Tremendous amounts of RAM at low cost
* But is GC the real cause?
* Use pauseless GC

!SLIDE transition=blindY
# Using GPUs
.notes Graphics card processing (OpenCL) is great for specific kinds of operations, like floating point arithmetic. But they don't perform well for all operations, and you have to get the data to them.  

* Locality matters!
* Need to be able to export a task with data that does not need to update

!SLIDE transition=blindY
# Manycore 
.notes number of cores is large enough that traditional multi-processor techniques are no longer efficient[citation needed] — largely because of issues with congestion in supplying instructions and data to the many processors.  Network-on-chip technology may be advantageous above this threshold

* David Ungar says > 24 cores, generally many 10s of cores
* Really trying to think about it with 1000 or more
* Cache coherency won't be possible

!SLIDE transition=blindY
# Memristor
.notes from HP, due to come out this year or next, may change memory fundamentally.  If the data and the function are together, you get the highest throughput and lowest latency possible.  Could replace DRAM and disk entirely.  When current flows in one direction through the device, the electrical resistance increases; and when current flows in the opposite direction, the resistance decreases.  When the current is stopped, the component retains the last resistance that it had, and when the flow of charge starts again, the resistance of the circuit will be what it was when it was last active.

* Non-volatile, static RAM, same write endurance as Flash (is that good enough for anything but storage?)
* 200-300 MB on chip
* Sub-nanosecond writes
* Able to perform processing
* Multistate, not binary

!SLIDE transition=blindY
# Phase Change Memory (PRAM)
.notes Thermally-driven phase change, not an electronic process.  Faster because it doesn't need to erase a block of cells before writing.  Flash degrades at 5000 writes per sector, where PRAM lasts 100 million writes.  Some argue that PRAM should be considered a memristor as well.

* Higher performance than today's DRAM
* Intel seems more fascinated by this
* No idea when we can expect to have something usable
* Not able to perform processing
* Write degradation is supposedly much slower
* Susceptible to unintentional change 

!SLIDE transition=blindY
# Credits

* [What Every Programmer Should Know About Memory, Ulrich Drepper of RedHat, 2007](http://people.freebsd.org/~lstewart/articles/cpumemory.pdf)
* [Java Performance](http://www.amazon.com/Java-Performance-Charlie-Hunt/dp/0137142528/ref=sr_1_1?ie=UTF8&qid=1330796836&sr=8-1)
* [Wikipedia/Wikimedia Commons](http://wikipedia.org/)
* [AnandTech](http://www.anandtech.com/)
* [The Microarchitecture of AMD, Intel and VIA CPUs](http://www.agner.org/optimize/microarchitecture.pdf)
* [Everything You Need to Know about the Quick Path Interconnect, Gabriel Torres](http://www.hardwaresecrets.com/article/610)

!SLIDE transition=blindY
# Credits

* [Inside the Sandy Bridge Architecture, Gabriel Torres](http://www.hardwaresecrets.com/printpage/Inside-the-Intel-Sandy-Bridge-Microarchitecture/1161)
* [Martin Thompson's Mechanical Sympathy blog and Disruptor presentations](http://mechanical-sympathy.blogspot.com/)
* [Gil Tene, CTO of Azul Systems](http://www.azulsystems.com/presentations/application-memory-wall)
* [AMD Bulldozer/Intel Sandy Bridge Comparison by Gionatan Danti](http://www.ilsistemista.net/index.php/hardware-analysis/24-bulldozer-vs-sandy-bridge-vs-k10-comparison-whats-wrong-with-amd-bulldozer.html)
* Martin Thompson and Cliff Click provided feedback and additional content