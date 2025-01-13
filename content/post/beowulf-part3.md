---
title: "Distributed Computing with Custom Beowulf Cluster"
<!-- author: Jon Deaton -->
date: '2017-12-04'
---

_Originally published on [wordpress](https://jondeaton.wordpress.com/2018/12/04/parallel-computing-with-custom-beowulf-cluster/)_

<img 
  src="/images/cluster_blog/cluster.webp" 
  alt="Assembled Cluster"
  style="width: 300px; float: right; margin-left: 10px; margin-bottom: 10px;" 
/>

In my last two blog posts, I discussed how I  assembled and configured a distributed
computing cluster using some old laptops and Raspberry Pis. In this post I will discuss
using this cluster for developing distributed computing tasks in C++ using MPI. The
distributed application that I developed is a DNA k-mer counter.

## Background on DNA Sequence analysis

In biology,  the analysis of DNA sequences is critical in understanding biologic
systems. Many DNA analysis algorithms focus on identifying genes (the functional units
that DNA encodes), however, some DNA analysis algorithms focus on other features of DNA
sequences. One alternate approach is analyzing the “k-mer” content of a DNA sequence.
K-mers are short sub-sequences of a DNA sequence of length k. Many DNA analysis
algorithms make conclusions about biologic systems based on the abundances of each k-mer
in the DNA sequence. Other k-mer based metrics include the number of unique k-mers in a
DNA sequence and the shape of the distribution of k-mer frequencies. In my undergraduate
research, I used the frequencies of k-mers in DNA sequences to identify novel viruses.

One step in gathering the training data was calculating the k-mer content in 18GB of
bacterial DNA sequences, which I later used as a feature set for machine learning. At
that time, I counted how many of each k-mer there were in each sequence using a
single-threaded Python program, which took over 6 hours to churn through the data set.
In this blog post I’ll describe how I took that Python code and turned it into a
distributed, multi-threaded k-mer counter in C++ that could complete the same task in
about 10 minutes.

## K-mer counting algorithm

The k-mer counting algorithm that I used is very simple: for a given DNA sequence, use a
sliding window length k (base pairs), and then slide along the DNA sequence from
beginning to end. For each position of the sliding window, increment a counter of how
many times that particular k-mer has been seen in the sequence. Since DNA has four base
pairs (i.e. A, T, G, C), there are 4k distinct k-mers to count the occurrences of.  To
store these counts create an array of integers of size 4k and designate each k-mers to
each element of the array lexicographically (e.g. AAA → 0, AAT → 1, AAC → 2, … GGG →
4k-1). Thus, for each position of the sliding window, we simply calculate the
lexicographic index of the k-mer and increment that element in the array.

The DNA data is stored in a file format called FASTA, which basically just stores the
ATGCs of the sequences as plain text. The program reads in each sequence from fasta
files, performs the described counting algorithm in memory, and then outputs the counts
of each sequence to file as plain text.

## Porting code to multi-threaded C++

I realized that if I was going to take advantage of the parallel capabilities of my
computing cluster I should first take advantage of parallelism on each individual node.
To do so, I re-wrote my original Python program in C++ where each DNA sequence was
scheduled for processing on a thread pool from the Boost libraries.

At this point, I bench-marked the C++ implementation against it’s Python predecessor
which took ~2 minutes to process a 150Mb file of viral genomes. My multi-threaded C++
program could process this file in ~1.8 seconds on the same computer– a 60x speedup.

An interesting discovery that I made was that if I set up the process to read data from
standard input (std::cin) as opposed to constructing an ofstream from the file path, the
program took nearly twice as long. Although some optimizations still remained, the task
was then ready to be ported to a distributed implementation that shares the work across
multiple computers.

## Installing OpenMPI

A critical step in getting distributed tasks running is having some way to communicate
between nodes. One option is passing around shell commands over ssh. I used this
approach before but for this project I wanted a more intimate and flexible interface so
I went with MPI. I chose to use OpenMPI simply because of the ease of installation.

One thing I had to do was build OpenMPI from source so that heterogenous computing would
be supported. This is because the computers in my cluster had different architectures. I
followed the instructions on the main page but passed the --enable-heterogeneous flag to
configure like so:

```bash
tar zxvf openmpi-3.0.0.tar.gz
cd openmpi-3.0.0 
./configure --prefix=/opt/openmpi-3.0.0 --enable-heterogeneous 
make all install
```

I ran this seperately on all the machines in the cluster. I also added
/opt/openmpi-3.0.0/bin and /ope/openmpi-3.0.0/lib to $PATH and $LD_LIBRARY_PATH
respectively so that I could import and link the MPI libraries in my code. 

## Batch Processing Implementation with MPI

The basic design of the application is very similar to that of a so called “thread
pool”: a single node is assigned to be the “head” node which sets up a queue of work to
be done and then delegates tasks from the queue to all of the worker nodes. The worker
nodes simply notify the head node when they are ready to process something, receive and
process some work (in this case a file), and then either return the result of the
processing to the head node, or persist their answer by writing to a file, database,
etc…

Plugging my multi-threaded k-mer counter into this system design was theoretically very
easy. The head node would search in a directory full of DNA sequence files and add each
file to a queue, then send each file out to worker nodes. The worker nodes would use the
(previously described) k-mer counting program to process the files and write out their
answer to disk.

When programming with MPI, you use the basic message passing functionality of MPI_Send
and MPI_Recv to pass messages of arbitrary size between nodes. I was helped by a great
tutorial on basic MPI routines like sending, receiving, and probing messages, that I
would suggest if you are interested in learning.

In my program I defined a generic BatchProcessor class contains a “master routine” and
“worker routine” and functions to scheduling work. I call each unit of work a “key”
which in my application is a string representing the filename of a FASTA file to
process. To use the BatchProcessor a client program provides a function for scheduling
keys, and a function for processing keys. In my application these functions are locating
FASTA files, and counting k-mers in the sequences respectively.

Here is the implementation of the “worker routine” that is performed by all nodes on the
cluster except the head node:

```cpp
void BatchProcessor::worker_routine(function processKey) {
 MPI_Status status; // To store status of messages
 char ready = BP_WORKER_READY; // Single byte to send to master node

 int messageSize; // To store how long the incoming messages are
 string nextKey; // Stores the next key

 while (true) { // Continue until head node says stop
   // Tell the head node we are ready to process something
   MPI_Send(&ready, sizeof(char), MPI_BYTE, BP_HEAD_NODE, BP_WORKER_READY_TAG, MPI_COMM_WORLD);

    // Get information about the next key to come
    MPI_Probe(BP_HEAD_NODE, MPI_ANY_TAG, MPI_COMM_WORLD, &status);
    if (status.MPI_TAG == BP_WORKER_EXIT_TAG) break; // time to be done

    // Get the next key to process
    MPI_Get_count(&status, MPI_CHAR, &messageSize); // find key size
    auto key_cstr = (char*) malloc((size_t) messageSize); // Allocate memory for key
    MPI_Recv(key_cstr, messageSize, MPI_CHAR, BP_HEAD_NODE, BP_WORK_TAG, MPI_COMM_WORLD, &status);
    nextKey = key_cstr; // copy the next file to process
    free(key_cstr);
    processKey(nextKey); // <-- work is done here
  }
}
```

This “worker routine” is fairly straightforward: first the worker notifies the head node
that it is ready to process some work with an MPI_Send. The BP_WORKER_READY_TAG is just
a #define integer in my program for use to indicate that a worker is ready to process
work. This command blocks until it’s sent to the head node. Once it is sent, the worker
then waits for a response from the head node with a “key” with a blocking MPI_Recv call.
The “keys” in this case are just file names (the DNA sequence files to be processed).
The worker needs to use dynamically allocated memory (hence malloc) to store in incoming
key (filename) since it can be any length. Once a filename is received, the worker’s job
is to process that key with the processKey method provided to it, which simply to
asynchronously count k-mers in the specified file and write the answer to disk. The
worker continues this cycle until it is notified by the head node that there is no more
work to complete and it can exit.

The master node’s routine is a little bit more complicated since it needs to coordinate
all of the workers. However, to increase performance and reduce complexity, I broke it’s
routine up into two sub-tasks, and scheduled each of these as tasks on the head node’s
thread pool. These tasks are:

Search the user-specified directory for  FASTA files, and add each to the queue of
tasks. Wait for a worker to become available, then send it the next item from the queue

The first of these tasks is the simplest but by processing it asynchronously on the
thread pool, work dispatching and processing can occur while scheduling is still
happening. This could allow for a long-running scheduling task. Anyways, the scheduling
task is as so:

```cpp
pool.schedule([&](){
    schedule_keys(); // Schedule keys asynchronously
    lock_guard lg(scheduling_complete_mutex);
    scheduling_complete = true; // Indicate done scheduling
}
```

The schedule_keys function is a client defined function passed to the BatchProcessor
that searches through a directory for the FASTA files, and then schedules them on batch
processor’s queue.  Inside of the definition of schedule_keys the client should use the
batch processor’s special schedule_key method which queues the key in a thread-safe
manner, and notifies the work-dispatching thread on the master node:

```cpp
void BatchProcessor::schedule_key(const string &key) {
 if (world_rank != BP_HEAD_NODE) return; // scheduling on worker nodes is forbidden
 lock_guard lg(queue_mutex); // Lock the queue of keys
 keys.push(key); // Add to the queue
 schedule_cv.notify_one(); // Notify potentially waiting thread of scheduling
}
```
The work dispatching task performed by the head node is as follows:

```cpp
pool.schedule([&](){
  MPI_Status status;
  char worker_ready;

  while (!work_completed()) {
    // Waiting for ready worker
    MPI_Probe(MPI_ANY_SOURCE, BP_WORKER_READY_TAG, MPI_COMM_WORLD, &status);
    if (status.MPI_ERROR) continue;

    // Found ready worker
    MPI_Recv(&worker_ready, sizeof(char), MPI_BYTE, status.MPI_SOURCE, BP_WORKER_READY_TAG, MPI_COMM_WORLD, &status);
    if (!worker_ready || status.MPI_ERROR) continue; // Error or incorrect signal?

    if (queue_empty()) { // no more keys to process
      if (scheduling_completed()) break;
      else { // Wait until something has been put on the queue
        unique_lock lock(schedule_mutex);
        schedule_cv.wait(lock, [this]() {
          return !queue_empty();
        });
        lock.unlock();
      }
    }

    // There is some work to do in the queue
    unique_lock lock(*worker_mutex_list[status.MPI_SOURCE]);
    worker_ready_list[status.MPI_SOURCE] = false; // Mark worker as busy
    lock.unlock();

    // Send the next work to the worker
    queue_mutex.lock();
    string next_key = keys.front();
    MPI_Send(next_key.c_str(), (int) next_key.size() + 1, MPI_CHAR, status.MPI_SOURCE, BP_WORK_TAG, MPI_COMM_WORLD);
    keys.pop(); // Remove the key
    queue_mutex.unlock();
  }

  // Need to indicate to each worker to exit once the work is done
  for (int i = 0; i < world_size; i++) {
    if (i == BP_HEAD_NODE) continue;
    send_exit_signal(i);
  }
});
```

This task is really broken up into three parts:

1. Wait for work to be put on the queue 
2. Wait for a worker to signal its ready and then send the next work from the queue 
3. Tell all the workers to exit once the queue is empty and scheduling is complete

## Compiling

CMake is great in that it helps you to easily produce portable makefiles. I used CMake
to help with linking MPI and Boost libraries. If you are interested in the general
technique for linking MPI libraries:

```cmake
find_package(MPI REQUIRED)
include_directories(${MPI_INCLUDE_PATH})

set (MPI_TEST_SOURCE test/test-mpi.cpp)

add_executable (test-mpi ${MPI_TEST_SOURCE})
target_link_libraries (test-mpi ${MPI_LIBRARIES})

if (MPI_COMPILE_FLAGS)
    set_target_properties(test-mpi PROPERTIES COMPILE_FLAGS "${MPI_COMPILE_FLAGS}")
endif()

if (MPI_LINK_FLAGS)
    set_target_properties(test-mpi PROPERTIES LINK_FLAGS "${MPI_LINK_FLAGS}")
endif()
```

## Troubleshooting running OpenMPI

I ran into problems with even just getting OpenMPI to be able to spawn up tasks on other
machines. While testing I would compile a simple MPI version of hello world with mpicc
mpi_hello.c -o hello and run it like so:

```
mpirun -n 1 -H node0 hello
```

but I continued to receive the following error

```
ORTE was unable to reliably start one or more daemons.

```

with suggestions for why this was occurring being:

Not finding the required libraries and/or binaries Lack of authority to execute on one
or more specified nodes. The inability to write startup files into /tmp Compilation of
the orted with dynamic libraries when static are required An inability to create a
connection back to mpirun due to a lack of common network interfaces

I learned quickly that this was a very common error and could be caused by any number of
things that went wrong. The lack of more specific error messages had me searching for
hours before I came upon something from this post that saved me. For some reason
specifying the absolute path of the mpirun executable fixed this problem. Using


```
$(which mpirun) -n 1 -H node0 hello
```

worked perfectly and I didn’t have to remember the full path to mpirun.

Because I had a heterogenous cluster, at this point it was also time to investigate
whether I would be able to compile different version of my distributed program and
specify which executables to run on which machines. Luckily, Gilles Gouaillardet helped
me on my Stack Overflow question about how to specify multiple compiled versions of the
same program running with syntax like the following

```
mpirun -n 2 -H node0,node1 prog_x86 : -n 2 -H node2,node3 prog_arm 
```

where prog_x86 and prog_arm are just example names for versions of the same program
compiled for the different architectures in the cluster. This command would launch the
version of the program compiled for x86 on node0 and node1 and the version of the
program compiled for ARM on node2 and node3. 

## Testing

Once I had distributed tasks up and running I was able to analyze how this distributed
application ran. Unfortunately when I tested this application my personal computing
cluster was in a state of disrepair so I tested it instead on a shared computing cluster
at school. With four nodes I was able to churn through 1.9 Gb of data in 1 minute. For
reference, the original data set was 18GB and took 6.5 hours to churn through. This
suggests that I was able to achieve a 40x speedup using parallelism and distributed
computing. Under these assumptions, I would have been able to complete 18GB task in
about 10 minutes. 

## Areas for improvement

You may have noticed that with version 2 of the program (single-node multi-threaded C++
code), I was able to get a 60x speedup, but the distributed program only had a 40x
speedup. I believe this is for a few reasons, not the least of which is that I was
executing these tasks on shared computing resources.

One other reason is that the workers only ask for one task at a time and once they
complete that single task, they need to wait for a relatively long time for the master
node to send them another task. This program could be modified so that the worker nodes
ask for several tasks at a time, and that way can still process data while waiting for
the head node to dispatch more work to them. 

## Conclusion

In conclusion this was a great project that taught a bit about distributed computing and
MPI, gave me some exposure to the boost libraries, and honed my skills with C++. In the
future I may make further optimizations to utilize the compute more efficiently, or even
to test it on my own cluster once I get it back up and running. You can take a look at
my code here.
