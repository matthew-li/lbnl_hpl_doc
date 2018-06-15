High-Performance LINPACK Tutorial
==========================================

## Overview

This document details how to setup and run a simple High-Performance LINPACK (HPL) test on a set of nodes in order to measure performance, specifically, its rate of execution of floating-point operations, by using the nodes to solve a large linear system. This tutorial is tailored to the computing environment (Intel architecture, Scientific Linux 7) in the HPCS division at Lawrence Berkeley National Laboratory at the time of this writing, but the majority can be applied elsewhere. View the [sample](https://github.com/matthew-li/lbnl_hpl_doc/tree/master/samples) files for examples.

Version 1 - April 18th, 2018

## Table of Contents

1. [Loading Modules](#1-loading-modules)
2. [Compiling the Test](#2-compiling-the-test)
3. [Gathering Parameters](#3-gathering-parameters)
4. [Editing HPL.dat](#4-editing-hpldat)
5. [Creating a Node List](#5-creating-a-node-list)
6. [Running the Test](#6-running-the-test)
7. [Interpreting Output](#7-interpreting-output)
8. [Example](#8-example)

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
- Change `OMP_DEFS` from `-openmp` to `-qopenmp`.

In `hpl-2.2`, compile the program by running:

```
make arch=intel64
```

If compilation succeeds, the directory `hpl-2.2/bin/intel64` should now exist. In it you will find the files `HPL.dat` and `xhpl`. At this point, you require only these two files; that is, you can copy and run them from anywhere.

## 3. Gathering Parameters

We next need to acquire basic information about the nodes being tested. In particular, we will require the number of nodes the test will be run on, the number of cores per node, the speed of each core, the amount of memory each node has, and the instructions per cycle.

Login to one of the nodes in the cluster being tested.

***

**Number of Nodes**

This value will be the only one to vary between multiple tests on the same cluster. Suppose your cluster has 100 nodes. We would begin by running the test on a single node, and then repeatedly scaling up by a factor of 2: 1, 2, 4, 8, 16, 32, 64. Finally, we would run the test on all 100 nodes. You may scale as you’d like, but using this method, you should see an approximately linear increase in your output.

***

**Cores Per Node**

```
lscpu | grep "CPU(s)"
```

Record the value corresponding to `CPU(s)`.

***

**Speed Per Core**

```
lscpu | grep "GHz"
```

Record the number of GHz in the value corresponding to `Model name`.

***

**Memory Per Node**

```
cat /proc/meminfo | grep "MemTotal"
```

Record the value corresponding to `MemTotal`, rounded down to the closest well-known multiple of 2 (e.g. 214477800 KB becomes 192000000 KB = 192 GB).

***

**Instructions Per Cycle**

This value, 16 or 32, is machine-dependent. Look online for the correct value pertaining to the particular CPU (found under `Model name` when determining the Speed Per Core above). For example, Google searches reveal that the Intel X5550 (Nehalem) runs at 4 instructions per cycle and the Intel E5-2670 (Sandy Bridge) runs at 8 instructions per cycle.

***

Navigate to http://hpl-calculator.sourceforge.net/. Input the values you found above. Click on “More Details (Running HPL)”. Here you will find the parameters for your test.

Note that behavior is unclear when nodes being tested have different parameters.

## 4. Editing HPL.dat

We now need to edit `HPL.dat`. In particular, we will change the lines labeled `Ns`, `NBs`, `Ps`, and `Qs`.

Look back at the results of the HPL Calculator. In the bottommost table, we find various “problem sizes”, which vary with `N` and `NB`, where `N` is the order of the coefficient matrix  `A` being solved, and `NB` is the partitioning block factor. In general, and for the sake of this tutorial, `N` = 90% and `NB` = 192 tend to be good choices. Find the value corresponding to these in the table. In `HPL.dat`, set `Ns` to be that value and set `NBs` to 192.

`P` and `Q` correspond to the number of process rows and columns, respectively. We want `P` and `Q` to be approximately equal, with `Q` slightly larger than `P`. Compute the square root of the product of the number of nodes and the number of cores per node. Choose `P` and `Q` as close to this value as possible. Set `Ps` and `Qs` in `HPL.dat`.

See the [example](#8-example) for sample values.

## 5. Creating a Node List

Create a file `nodelist.in` to contain the list of nodes to be included in the test. List each node on a separate line, with the number of cores it has listed next to it, denoted by `slots=`.

Consider a hypothetical cluster of 5 nodes denoted by `n000i.cluster_name`, where each has `n` cores. Then `nodelist.in` should contain:

```
n0000.cluster_name slots=n
n0001.cluster_name slots=n
n0002.cluster_name slots=n
n0003.cluster_name slots=n
n0004.cluster_name slots=n
```

See the [example](#8-example) for sample node lists.

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

See the [example](#8-example) for sample commands.

## 7. Interpreting Output

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

The rate of execution labeled `Gflops` in the last line before the test completed. E.g.:

```
Column=000083712 Fraction=99.8% Gflops=3.657e+02
```

***

**Time**

The time in seconds to solve the linear system, labeled `Time` in the final result. E.g.:

```
================================================================================
T/V                N    NB     P     Q               Time                 Gflops
--------------------------------------------------------------------------------
WR00C2R4       83904   192     2     4            2310.54              1.704e+02
```

***

**WR Number**

The rate of execution for solving the linear system, labeled `Gflops` in the final result. E.g.:

```
================================================================================
T/V                N    NB     P     Q               Time                 Gflops
--------------------------------------------------------------------------------
WR00C2R4       83904   192     2     4            2310.54              1.704e+02
```

***

**Gflop/s Per Node**

The rate of execution for solving the linear system, averaged across all nodes used in the test. E.g.:

```
gflops_per_node = wr_number / number_of_nodes
```

***

**Theoretical RPeak**

The theoretical maximum performance, in Glop/s, achievable by the set of nodes, is, in the simplest case, given by the formula below. It can also be found in the results of the HPL Calculator, under `100% (RPeak)`.

```
RPeak = number_of_nodes * cores_per_node * speed_per_core * instructions_per_cycle
```

For example, the Theoretical RPeak for a set of 4 nodes, each with 64 cores running at 3.0 GHz and 16 instructions per cycle, is given by:

```
RPeak = 4 * 64 * 3 * 16 = 12288
```

This may differ in the presence of additional factors: Advanced Vector Extensions, wherein some instructions may span more than one clock cycle; turbo; etc.

***

Record these results, as well as the parameters used in running the test.

See the [example](#8-example) for sample output.

## 8. Example

See the [sample](https://github.com/matthew-li/lbnl_hpl_doc/tree/master/samples) files pertaining to this example.

Suppose we wish to run HPL on n0[175-183].etna0. We login to one of the nodes being tested. Using `lscpu` and `cat /proc/meminfo`, we find that each has Intel(R) Xeon(R) CPU E5-2623 v3 @ 3.00GHz. There are 9 nodes, each with 64 GB of memory and 8 cores running at 3.00 GHz and 16 instructions per cycle.

We begin by running the test on a single node, and then repeatedly scale up by a factor of 2 while possible: 1, 2, 4, 8, 9. In total, in this case, we run 5 tests.

### One Node

Inserting the parameters into the HPL Calculator reveals that `Ps` = 2, `Qs` = 4, `Ns` = 83328, `NBs` = 192, each of which we set in `HPL.dat`.

For the single node case, a node list is unnecessary, so we run:

```
mpirun -np 8 ./xhpl
```

The relevant excerpt of the output is below:

```
Column=000083712 Fraction=99.8% Gflops=3.657e+02
================================================================================
T/V                N    NB     P     Q               Time                 Gflops
--------------------------------------------------------------------------------
WR00C2R4       83904   192     2     4            2310.54              1.704e+02
```

The Theoretical RPeak is 384.

### Two Nodes

Inserting the parameters into the HPL Calculator reveals that `Ps` = 4, `Qs` = 4, `Ns` = 118848, `NBs` = 192, each of which we set in `HPL.dat`.

For multiple nodes, a node list is necessary:

```
n0175.etna0 slots=8
n0176.etna0 slots=8
```

We run:

```
mpirun -np 16 -hostfile nodelist_n0[175-176] ./xhpl
```

The relevant excerpt of the output is below:

```
Column=000118656 Fraction=99.8% Gflops=7.296e+02
================================================================================
T/V                N    NB     P     Q               Time                 Gflops
--------------------------------------------------------------------------------
WR00C2R4      118848   192     4     4            3839.37              2.915e+02
```

The Theoretical RPeak is 768.

### More Nodes

We repeat this process for 4, 8, and 9 nodes.

### Recording Output

We record output in a spreadsheet, as below:

```
n0[175-183].etna0 [Nodes: 9 | Cores/Node: 8 | Speed: 3.0GHz | Memory/Node: 64GB | Instructions/Cycle: 16]
Date        # Nodes    P * Q    P    Q    N (Problem Size)    NB (Block Size)    Progress (Gflops)    WR (Gflops)    Time       Gflops/Node    Theoretical RPeak    Command
01/08/18    1          8        2    4    83904               192                3.66E+02             1.70E+02       2310.54    170.4          384                  mpirun -np 8 ./xhpl
01/08/18    2          16       4    4    118848              192                7.30E+02             2.92E+02       3839.37    145.75         768                  mpirun -np 16 -hostfile nodelist_n0[175-176] ./xhpl
01/12/18    4          32       4    8    166656              192                1.47E+03             6.34E+02       4867.8     1.58E+02       1536                 mpirun -np 32 -hostfile nodelist_n0[175-178] ./xhpl
01/12/18    8          64       8    8    235776              192                2.90E+03             1.16E+03       7537.68    1.45E+02       3072                 mpirun -np 64 -hostfile nodelist_n0[175-182] ./xhpl
01/14/18    9          72       8    9    250176              192                3.23E+03             1.29E+03       8100.32    1.43E+02       3456                 mpirun -np 72 -hostfile nodelist_n0[175-183] ./xhpl
```
