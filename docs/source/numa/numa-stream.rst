NUMA, really
============
On shared-memory systems, different processing elements have access to a unified shared memory.
Although this means each of those processing elements can access data stored anywhere in the shared memory, in reality, the system has a hierarchical structure where the memory sharing is an _abstraction_.
A typical example might be a dual-socket CPU, where each socket is connected to some memory. The entire memory space is shared across both sockets, but cores in the first socket accessing data in memory physically connected to the second socket will need to traverse the socket-to-socket interconnect to access the second socket's memory controllers.
Compare this to that first socket accessing data via its own memory controllers.
It is clear, perhaps, that these situations may yield different performance characteristics: both in latency and bandwidth.

Discover NUMA regions
---------------------
`numactl`

How big are NUMA effects?
-------------------------

To see just how different the performance of accessing memory in different NUMA regions can be, let's do a small experiment on a dual-socket CPU system with Marvell ThunderX2 CPUs. Each socket has 32 cores, each with 4 https://en.wikipedia.org/wiki/Simultaneous_multithreading[simultaneous multithreading (SMT)]. The core configuration is typical, where CPUs/threads 0-63 are the first thread on each of the 64 physical cores. This system has two NUMA nodes, corresponding to each socket.

The https://www.cs.virginia.edu/stream/[STREAM benchmark] measures the sustain memory bandwidth performance of CPUs. It uses the OpenMP programming model to run in parallel on a shared-memory system. It is easy to compile with GCC as follows:

[source, bash]
----
gcc stream.c -fopenmp -march=native -Ofast -o stream
----

To experiment with NUMA placement, we can run an experiment where we place all threads on one socket, and access memory in the same NUMA region, and compare to the performance of accessing memory from the other NUMA region.

We therefore can run with 32 OpenMP threads, pinning them to the available cores using OpenMP environment variables. We can also use the `numactl` tool to limit the hardware resources available to the process in terms of the cores and the memory locations. This is controlled with the `-N` and `-m` flags respectively, each taking a node number associated with the particular NUMA region, as shown by the earlier exploration with `numactl -H`::

    OMP_NUM_THREADS=32 OMP_PLACES=cores OMP_PROC_BIND=close numactl -N 0 -m 0 ./stream

.. include:: stream-same-numa.txt
      :literal:

Here we see we get around 115 GB/s of memory bandwidth when we run in "near" memory.

Now, let's move the data to the other socket::

    OMP_NUM_THREADS=32 OMP_PLACES=cores OMP_PROC_BIND=close numactl -N 0 -m 1 ./stream

.. include:: stream-other-numa.txt
      :literal:

Instead, we attain around 33 GB/s, a significant drop of around 3.5 times performance; the sustained bandwidth from accessing the "far" memory is noticeably slower than accessing the "near" data.

How to make a code NUMA-aware
-----------------------------


I don't have numactl
-----------------------

You can use taskset to control the core placement, but it doens't do the memory. Generally though you can see

Acknowledgements
----------------
This work used the Isambard 2 UK National Tier-2 HPC Service (http://gw4.ac.uk/isambard/) operated by GW4 and the UK Met Office, and funded by EPSRC (EP/T022078/1).

