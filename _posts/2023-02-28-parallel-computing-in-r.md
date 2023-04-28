---
title:  "Parallel Computing in R"
layout: post
---

# Loops in Computer Languages


## Compiled Language (C/Fortran)


* In a compiled language like C or Fortran, the code is compiled into machine code,
which is a set of instructions that can be executed directly by the
computer's processor. The compiler generates this machine code once,
before the program is executed.


* When a loop is executed in a compiled language,
the machine code for the loop has already been generated, and the loop runs
as a series of pre-compiled machine code instructions.


## Interpreted Language (R/Python)


* In an interpreted language, the code is not compiled into machine code
before the program is executed. Instead, the interpreter reads each line of code,
translates it into machine code on-the-fly, and executes it.


* When a loop is executed in an interpreted language, the interpreter must
translate and execute the loop body instructions on each iteration of the loop.


> Loops in compiled languages are generally faster than loops in interpreted
languages, since the machine code for the loop has already been generated and
optimized for the specific processor architecture.


# Parallel Computing


> Parallel computing is a type of computing architecture in which multiple
 processors or computing resources work together to solve a problem or
 execute a program. In parallel computing, a task is divided into smaller
 sub-tasks, which can be executed simultaneously by multiple processors or
 computing resources.


* Run the same code multiple times with different input arguments (Sub-tasks are independent)
    * Split 1000 replications into 1000 CPUs and do the computing at the same time.
    * To make the results reproducible, we need to assign
      different random seeds to different CPUs
* Different sub-tasks are dependent on each other.
    * GNU Parallel: command-line driven utility that allows the user to
    execute shell scripts or commands in parallel.
    * OpenMPI: Message Passing Interface library project combining
    technologies and resources from several other projects


## Parallel Computing in R


> Parallel computing in R can be achieved using several techniques, including
the `parallel` package, the `foreach` package, and  the `multicore` package.


### The `parallel` Package


> This package provides support for parallel computing using multiple
processes. The parallel package comes pre-installed with R, so there is no need
to install it separately. Here's an example of how to use the `parSapply()`
function from the `parallel` package to parallelize the application of a
function to a list of inputs.


> To begin with, you need to create a cluster on your own computer by
utilizing the `makeCluster()` function, and the number of nodes to be specified
in the function should correspond to the number of cores available on your CPU.


```R
library(parallel)
cl <- makeCluster(n) # Create the cluster with n cores
```

> After creating the cluster, you can execute your function
`myfunc()` with various inputs `mylist` using the `parSapply()` function.
The inputs will be split across the `n` cores, and all cores will execute
their assigned jobs concurrently


```R
results <- parSapply(cl, mylist, myfunc())
stopCluster(cl) # Stop the cluster
```

## Assign Random Seeds


> For reproducibility, each job in parallel computing should be assigned a
unique random seed. In `R`, the `set.seed()` function can be used to set a
random seed for a single experiment. However, when running 1000 parallel jobs,
assigning sequential values such as 1 to 1000 as random seeds is not truly
random and may not produce the desired level of reproducibility.


> An often employed method involves generating a new random seed by utilizing
another random seed. To produce the initial seed, the following code snippet
can be utilized:


```R
RNGkind("L'Ecuyer-CMRG")
njobs <- 1000
set.seed(2002)
seeds <- list(.Random.seed) # Set up an initial seed
```


> Subsequently, generate the next seed by building upon the previous one by the
`nextRNGStream()` function in the `parallel` package.


```R
for (i in seq(2, njobs, 1)) {
  # Generate the following seed based on the previous one
  seeds[[i]] <- parallel::nextRNGStream(seeds[[i - 1]])
}
```

>It is important to note that for each task, the random seed produced using
this approach must be allocated to the global environment to have an impact.
As an illustration, if the initial random seed needs to be established for the
first task, the following code must be included:


```R
assign(".Random.seed", seeds[[1]], envir = .GlobalEnv)
```


## Example


> The subsequent code calculates the sum of 100 random numbers generated from a
standard normal distribution with 1 added to each. This process is repeated 1000
times in parallel on a cluster with 4 cores


```R
library(parallel)
cl <- makeCluster(4)
# Write the function
myfunc <- function(seed, add) {
  assign(".Random.seed", seed, envir = .GlobalEnv)
  sum(rnorm(100) + add)
}
# Generate the random seeds
RNGkind("L'Ecuyer-CMRG")
njobs <- 1000
set.seed(2002)
seeds <- list(.Random.seed)
for (i in seq(2, njobs, 1)) {
  seeds[[i]] <- parallel::nextRNGStream(seeds[[i - 1]])
}
# Execute the code on the cluster
results <- parSapply(cl, seeds, myfunc, add = 1)
stopCluster(cl)
```

> The `parSapply()` function is the parallel version of `sapply()`. For the
`i`'th task, `parSapply()` calls `myfunc(seed, add)` with
`seed = seeds[[i]]` and the `add = 1`.


> The `parallel` package contains several `apply` functions for various purposes,
and the
[documentation](https://www.rdocumentation.org/packages/parallel/versions/3.6.2)
of the `parallel` package provides comprehensive information
on this matter.


# Parallel computing on HPC Cluster using R


> A personal computer has a restricted number of CPU cores. Consequently, executing 100 or
1000 tasks simultaneously on a personal computer is not feasible, and a High-Performance
Computing (HPC) cluster is required. In my upcoming blog post, I will outline how to
utilize parallel computing in R on an HPC cluster.














