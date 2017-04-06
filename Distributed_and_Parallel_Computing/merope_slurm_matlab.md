# Introduction
This document is about to use Slurm commands to understand how Merope cluster works with MATLAB Parallel Pool.  

# Merope Cluster  
For the basic information on partitions, hardware configurations, etc, please check the TCSC wiki page:    
[https://wiki.tut.fi/TCSC/MeropeStepByStep](https://wiki.tut.fi/TCSC/MeropeStepByStep  )  

# Slurm Commands  
For the basic usage of Slurm commands, please check:  
[https://slurm.schedmd.com/sinfo.html](https://slurm.schedmd.com/sinfo.html)

# MATLAB Parallel Pool
The official document for MATLAB Parallel Pool, please check:  
[https://se.mathworks.com/help/distcomp/parpool.html](https://se.mathworks.com/help/distcomp/parpool.html)    
[https://se.mathworks.com/help/distcomp/run-code-on-parallel-pools.html](https://se.mathworks.com/help/distcomp/run-code-on-parallel-pools.html)

# How to make use of the clusters

Here is a typical Slurm script.  You can save it as `script_name.sh` and simply run it 
<pre><code>[XXX@cluster]$ sbatch script_name.sh </code></pre>

<pre><code>
#!/bin/bash
#
# give a job name
#SBATCH -J job_name
#
# redirect the output of the job to file
#SBATCH --output=you_name_it.out
#SBATCH --error=you_name_it.err
#
# Wanna email notification?
#SBATCH --mail-user=your@email.address
#SBATCH --mail-type=END
# 
# Run our job on 3 nodes and use 12 CPUs on each of them
#SBATCH --ntasks=3
#SBATCH --cpus-per-task=12
#
# Request 2GB memory for job (per node) and will run for one day
#SBATCH --time=1-00
#SBATCH --mem=2048
#
# Define on which partition to run our job
#SBATCH --partition=xyz
#
module load matlab  
#
#Start a parallel pool of 12 workers (per nodes), and run job_function() 
matlab -nodisplay  -r "parpool(12);job_function(); exit"
</code></pre>

It is important to check the hardware configuration of the cluster before setting `--ntasks` ,`--cpus-per-task` and of cause the `--partition`

Here is a example:

<pre><code>
[XXX@cluster]$ sinfo -p xyz
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
xyz          up 14-00:00:0      1 drain* me53
xyz          up 14-00:00:0     20   mix* me[25-29,38-45,47-52,54]
xyz          up 14-00:00:0      1 alloc* me58
xyz          up 14-00:00:0      2  idle* me[56-57]
xyz          up 14-00:00:0      2  down* me[46,55]
xyz          up 14-00:00:0      3    mix me[59-61]
xyz          up 14-00:00:0     17  alloc me[62-70,72-79]
</code></pre>

`sinfo` command gives the overview of the given partition, number of nodes, their status and names.

In order to get the most of them, we also need to further check their hardware configurations, such as number of CPUs and also CPU status. As we can see from the following output, there are at least two different hardware configurations in partition xyz: one is with two 8-core processors and the other is two 6-core processors.
<pre><code>

[XXX@cluster]$ sinfo -n me[59-61] -p xyz -o "%12P %.5a %.10l %.6D %.6t %.10N %.15C %.6Y %.6X"
PARTITION    AVAIL  TIMELIMIT  NODES  STATE   NODELIST   CPUS(A/I/O/T)  CORES SOCKET
xyz             up 14-00:00:0      3    mix  me[59-61]      36/12/0/48      8      2
[XXX@cluster]$ 
[XXX@cluster]$ sinfo -n me59 -p xyz -o "%12P %.5a %.10l %.6D %.6t %.10N %.15C %.6Y %.6X"
PARTITION    AVAIL  TIMELIMIT  NODES  STATE   NODELIST   CPUS(A/I/O/T)  CORES SOCKET
xyz             up 14-00:00:0      1    mix       me59       12/4/0/16      8      2
[XXX@cluster]$
[XXX@cluster]$ sinfo -n me52 -p xyz -o "%12P %.5a %.10l %.6D %.6t %.10N %.15C %.6Y %.6X"
PARTITION    AVAIL  TIMELIMIT  NODES  STATE   NODELIST   CPUS(A/I/O/T)  CORES SOCKET
xyz             up 14-00:00:0      1   mix*       me52        8/4/0/12      6      2
</code></pre>

If we run the script above, we can check the status of our job and also the resources allocated to it.  For example, it could be:

<pre><code>
[XXX@cluster]$  squeue | grep XXX
          14668399       xyz job_name     XXX  R   0:00:38      3 me[59-61]
</code></pre>

So as in the setting, we got 3 nodes `me[59-61]`. And also taking a look at the previous output of the detail information of the resources, for those nodes `me[59-61]`, 36 CPUs are in using and 12 are idle as there are totally 48 CPUs (3 nodes x 8 cores x 2 processors). If we look into individuals, in this case, the node `me59`, there are 12 CPUs allocated out of 16, which is exactly what we defined at `--cpus-per-task=12`


## Trouble Shooting

1. Network problem or nodes die 

<pre><code>
Error using parallel_function (line 597)
All workers aborted during execution of the parfor loop.
...
The client lost connection to lab 9. This might be due to network problems, or
the interactive communicating job might have errored.
</code></pre>

- solution: It happens. Check the nodes status, run job again or contact Admin if necessary

2. Failed to start parallel pool 

<pre><code>
Error using parpool (line 111)
Failed to start a parallel pool. (For information in addition to the causing
error, validate the profile 'local' in the Cluster Profile Manager.)

Caused by:
    Error using parallel.internal.pool.InteractiveClient/start (line 326)
    Failed to start pool.
        Error using parallel.Job/submit (line 304)
        You requested a minimum of 32 workers, but only 12 workers are allowed
        with the Local cluster.
</code></pre>

- solution: As it shows in the error message, there are fewer workers allowed than what you request. In this case, just simply change the `parpool(32)` to `parpool(12)` in your script. 

3. Performance issue?

<pre><code>

                            < M A T L A B (R) >
                  Copyright 1984-2013 The MathWorks, Inc.
                    R2013b (8.2.0.701) 64-bit (glnxa64)
                              August 13, 2013  

To get started, type one of these: helpwin, helpdesk, or demo.
For product information, visit www.mathworks.com.
 
MATLAB detected: 16 physical cores.
MATLAB detected: 16 logical cores.
MATLAB was assigned: 16 logical cores by the OS.
MATLAB is using: 16 logical cores.

</code></pre>

- Solution: It is said that, by default, MATLAB will use as many workers as logical CPU cores. But in my case, for the same test, it runs much slower in default setting than it with manually call `parpool(12)`. Maybe it tries to use 16 workers, as shown above the output from function `feature('numcores')`, but actually only 12 works are allowed which results in using some other configurations and slow down the performance. The other point is that someone reported that depending on the architecture, there will be more logical cores detected than the physical cores, for example the hyperthreading, and the performance of using the number of logical cores as parpool size some how slow down the performance. So it is suggested to use the number of physical cores as the parpool size. 
