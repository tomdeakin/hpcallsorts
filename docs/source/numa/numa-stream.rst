NUMA, really
============
On shared-memory systems, different processing elements have access to a unified shared memory.
Although this means each of those processing elements can access data stored anywhere in the shared memory, in reality, the system has a hierarchical structure where the memory sharing is an *abstraction*.
A typical example might be a dual-socket CPU, where each socket is connected to some memory. The entire memory space is shared across both sockets, but cores in the first socket accessing data in memory physically connected to the second socket will need to traverse the socket-to-socket interconnect to access the second socket's memory controllers.
Compare this to that first socket accessing data via its own memory controllers.
It is clear, perhaps, that these situations may yield different performance characteristics: both in latency and bandwidth.

Discover NUMA regions
---------------------
The ``numactl`` tool, and the ``libnuma`` library is often available on Linux Operating Systems.
The ``--hardware`` (or ``-H`` flag for short) can be used to show the different NUMA regions available on your CPU system.
For example, on a dual-socket Marvell ThunderX2 node I have available, the output looks like this:

.. include:: numactl-h-tx2.txt
      :literal:

The output shows the number of NUMA regions (called *nodes*), and which CPU threads are associated with the different regions.
We can also see the size of the memory available in each NUMA region, which is roughly half the total system memory in each region.

We have to now how our threads and cores are numbered.
Each socket has 32 cores, each with 4 `simultaneous multithreading (SMT) <https://en.wikipedia.org/wiki/Simultaneous_multithreading>`_.
The core configuration is typical, where CPUs/threads 0-63 are the first thread on each of the 64 physical cores. This system has two NUMA nodes, corresponding to each socket.
We can see then that the first 32 cores (and the SMTs on those cores) are associated with the first NUMA node, and the second 32 cores (and SMT respectively) are associated with the second NUMA node.

The final part of the output is a diagram showing some numerical representation for distance between the NUMA codes.
This is read as a matrix, where say for a thread in node 0 accessing data in node 0, the cost is "10".
That same thread accessing data in node 1 as an increased cost of "20".
The numbers are entirely **fictional**, and are in no way representative mathematically of the cost.
So we can't say the cost is double because ``numactl`` shows the distance is twice as far.
These numbers just give us a way to sort the nodes by distance: the memory in node 1 is further away from node 0 then the memory in node 0, and vice versa.


How big are NUMA effects?
-------------------------

To see just how different the performance of accessing memory in different NUMA regions can be, let's do a small experiment on out ThunderX2 system.

The `STREAM benchmark <https://www.cs.virginia.edu/stream/>`_ measures the sustain memory bandwidth performance of CPUs. It uses the OpenMP programming model to run in parallel on a shared-memory system. It is easy to compile with GCC as follows::

    gcc stream.c -fopenmp -march=native -Ofast -o stream

To experiment with NUMA placement, we can run an experiment where we place all threads on one socket, and access memory in the same NUMA region, and compare to the performance of accessing memory from the other NUMA region.

We therefore can run with 32 OpenMP threads, pinning them to the available cores using OpenMP environment variables. We can also use the ``numactl`` tool to limit the hardware resources available to the process in terms of the cores and the memory locations. This is controlled with the ``-N`` and ``-m`` flags respectively, each taking a node number associated with the particular NUMA region, as shown by the earlier exploration with ``numactl -H``::

    OMP_NUM_THREADS=32 OMP_PLACES=cores OMP_PROC_BIND=close numactl -N 0 -m 0 ./stream

.. include:: stream-same-numa.txt
      :literal:

Here we see we get around 115 GB/s of memory bandwidth when we run in "near" memory.

Now, let's move the data to the other socket::

    OMP_NUM_THREADS=32 OMP_PLACES=cores OMP_PROC_BIND=close numactl -N 0 -m 1 ./stream

.. include:: stream-other-numa.txt
      :literal:

Instead, we attain around 33 GB/s, a significant drop of around 3.5 times performance; the sustained bandwidth from accessing the "far" memory is noticeably slower than accessing the "near" data.


I don't have numactl!
-----------------------
The ``taskset`` tool can be used as a partial alternative to ``numactl``, however it only controls the placement of application threads of Operating System threads (and cores).
Unfortunately, it isn't able to control the data placement in memory.

Some programming models also give ways to control thread and data placement on a system.
For example, OpenMP provides abstractions for thread placement and affinity, and memory allocation (through explicit allocators with properties).

Acknowledgements
----------------
This work used the Isambard 2 UK National Tier-2 HPC Service (http://gw4.ac.uk/isambard/) operated by GW4 and the UK Met Office, and funded by EPSRC (EP/T022078/1).

