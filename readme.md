High-Performance LINPACK Tutorial
==========================================

## Overview

This document details how to setup and run a simple High-Performance LINPACK (HPL) test on a set of nodes in order to measure performance. This tutorial is tailored to the computing environment (Intel architecture, Scientific Linux 7) in the HPCS division at Lawrence Berkeley National Laboratory at the time of this writing, but the majority can be applied elsewhere. View the sample files for examples.

## 1. Loading Modules

Login to one of the nodes in the cluster being tested.

HPL requires various libraries to run. In particular, a message passing interface and a math kernel library are necessary. In our case, at the time of this writing, each node runs Scientific Linux 7, which contains the requisite libraries, so run the following commands to load the packages and environment variables needed for compilation:

```
source ~/.bashrc
module load intel/2018.1.163
module load openmpi/2.0.2-intel
module load mkl/2018.1.163
```

You may need to run these each time you login to a node.

## 2. Compiling the Test

Navigate to http://www.netlib.org/benchmark/hpl/. Download and extract hpl-2.2.tar.gz.

In `hpl-2.2/setup`, there are several predefined defaults for various architectures that may be useful. Copy `hpl-2.2/setup/Make.Linux_Intel64` to `hpl-2.2` and rename it `Make.intel64`.

Make the following changes to `Make.intel64`:
- Change `ARCH` from `Linux_Intel64` to `$(arch)`. This will require you to pass in the desired architecture (`intel64`) when running the make command.
- Change `TOPdir` from `$(HOME)/hpl` to the absolute path to your `hpl-2.2` directory.
- In `LAlib`, change `libmkl_intel_thread.a` to `libmkl_sequential.a`.
- Change `CC` from `mpiicc` to `mpicc`.
- Change `OMP_DEFS` from `-openmp` to `qopenmp`.

In `hpl-2.2`, compile the program by running:

```
make arch=intel64
```

If compilation succeeds, the directory `hpl-2.2/bin/intel64` should now exist. In it you will find the files `HPL.dat` and `xhpl`. At this point, you require only these two files; that is, you can copy and run them from anywhere.

Sample Make.intel64.

## 3. Gathering Parameters

We next need to acquire basic information about the nodes being tested. In particular, we will require the number of nodes the test will be run on, the number of cores per node, the speed of each core, the amount of memory each node has, and the instructions per cycle.

Login to one of the nodes in the cluster being tested.

***

**Number of Nodes**

This value will be the only one to vary between multiple tests on the same cluster. Suppose your cluster has 100 nodes. We would begin by running the test on a single node, and then repeatedly scaling up by a factor of 2: 1, 2, 4, 8, 16, 32, 64. Finally, we would run the test on all 100 nodes. You may scale as you’d like, but using this method, you should see an approximately linear increase in your output.

***

**Cores Per Node**

```
cat /proc/cpuinfo
```
Look at the final entry under `processor` and add 1.

***

**Speed Per Core**

```
cat /proc/cpuinfo
```
Look under `model name` for the number of GHz.

***

**Memory Per Node**

```
cat /proc/meminfo
```
Look at `MemTotal` and round down to the closest power of 2 (e.g. 214477800 kB becomes 192000000 kB = 192 GB).

***

**Instructions Per Cycle**

Choose either 16 or 32, depending on the machine.

***

Navigate to http://hpl-calculator.sourceforge.net/. Input the values you found above. Click on “More Details (Running HPL)”. Here you will find the parameters for your test.

## 4. Editing HPL.dat

We now need to edit `HPL.dat`. In particular, we will change the lines labeled `Ns`, `NBs`, `Ps`, and `Qs`.

Look back at the results of the HPL Calculator. In the bottommost table, we find various “problem sizes”, which vary with `N` and `NB`. In general, and for the sake of this tutorial, `N` = 90% and `N` = 192 tend to be good choices. In `HPL.dat`, set `Ns` to be the value corresponding to 90% in the table. Set `NBs` to 192.

We want `P` and `Q` to be approximately equal, with `Q` slightly larger than `P`. Compute the square root of the product of the number of nodes and the number of cores per node. Choose `P` and `Q` as close to this value as possible. Set `Ps` and `Qs` in `HPL.dat`.

## 5. Creating a Node List

Create a file `nodelist.in` to contain the list of nodes to be included in the test. List each node on a separate line, with the number of cores it has listed next to it, denoted by `slots=`.

Consider a cluster of 5 nodes denoted by `node_i`, where each has 64 cores. Then `nodelist.in` should contain:

```
node_0 slots=64
node_1 slots=64
node_2 slots=64
node_3 slots=64
node_4 slots=64
```

## 6. Running the Test

We are now ready to run the test. Ensure that the above modules are loaded.

The following command runs the test on the single node you are logged into, where `PQ` should be replaced by the integer product of `P` and `Q`.

```
mpirun -np PQ ./xhpl
```

The following command runs the test on a set of nodes specified by a node list.

```
mpirun -np PQ -hostfile nodelist.in ./xhpl
```

We can also run these tests in the background, piping the output to an output file.

```
mpirun -np PQ ./xhpl > output.out &
mpirun -np PQ -hostfile nodelist.in ./xhpl > output.out &
```

Periodically check the progress of a test running in the background using:

```
tail -f output.out
```

## 7. Recording Output

Once the test is complete, the output should end in a similar fashion to below:

```
Column=000082944 Fraction=98.9% Gflops=3.657e+02
Column=000083136 Fraction=99.1% Gflops=3.657e+02
Column=000083328 Fraction=99.3% Gflops=3.657e+02
Column=000083520 Fraction=99.5% Gflops=3.657e+02
Column=000083712 Fraction=99.8% Gflops=3.657e+02
================================================================================
T/V                N    NB     P     Q               Time                 Gflops
--------------------------------------------------------------------------------
WR00C2R4       83904   192     2     4            2310.54              1.704e+02
HPL_pdgesv() start time Wed Jan 10 16:14:13 2018

HPL_pdgesv() end time   Wed Jan 10 16:32:11 2018

--VVV--VVV--VVV--VVV--VVV--VVV--VVV--VVV--VVV--VVV--VVV--VVV--VVV--VVV--VVV-
Max aggregated wall time rfact . . . :              12.46
+ Max aggregated wall time pfact . . :               4.40
+ Max aggregated wall time mxswp . . :               2.89
Max aggregated wall time update  . . :            2237.35
+ Max aggregated wall time laswp . . :              54.98
Max aggregated wall time up tr sv  . :               1.16
--------------------------------------------------------------------------------
||Ax-b||_oo/(eps*(||A||_oo*||x||_oo+||b||_oo)*N)=        0.0015945 ...... PASSED
================================================================================

Finished      1 tests with the following results:
              1 tests completed and passed residual checks,
              0 tests completed and failed residual checks,
              0 tests skipped because of illegal input values.
--------------------------------------------------------------------------------

End of Tests.
================================================================================
```

Record the following values, which will give you the beginning of an idea as to how well your system is performing.

***

**Progress**

The value labeled `Gflops` in the last line before the test completed. E.g.:

```
Column=000083712 Fraction=99.8% Gflops=3.657e+02
```

***

**Time**

The value labeled `Time` in the line beginning with `WR`. E.g.:

```
================================================================================
T/V                N    NB     P     Q               Time                 Gflops
--------------------------------------------------------------------------------
WR00C2R4       83904   192     2     4            2310.54              1.704e+02
```

***

**WR Number**

The value labeled `Gflops` in the line beginning with `WR`. E.g.:

```
================================================================================
T/V                N    NB     P     Q               Time                 Gflops
--------------------------------------------------------------------------------
WR00C2R4       83904   192     2     4            2310.54              1.704e+02
```

***

**Gflops Per Node**

The WR Number divided by the number of nodes used in the test.

***

**Theoretical RPeak**

The number under 100% RPeak, found from the HPL Calculator.

***
