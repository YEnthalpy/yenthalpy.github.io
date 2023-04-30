---
title:  "Parallel Computing in R using the HPC Cluster (Independent Tasks)"
layout: post
---

# Slurm Job Array


> When conducting a simulation study, it is often necessary to run the same
code with various inputs numerous times. To handle this task efficiently,
slurm job arrays can be utilized to swiftly submit and manage a large quantity
of comparable jobs.


## Example: Bootstrap Replicates of a Linear Model


> To obtain the R square coefficients for 10,000 bootstrap replicates of a
linear model, we aim to run the following function 10,000 times on an HPC
cluster with distinct random seeds.


```R
myfunc <- function(seed) {
  assign(".Random.seed", seed, envir = .GlobalEnv)
  b_df <- mtcars[sample(nrow(mtcars), rep = TRUE), ]
  mdl <- lm(mpg ~ wt + disp, data = b_df)
  return(summary(mdl)$r.square)
}
```


> To achieve this, we will partition the 10,000 replicates into 250 job arrays, each
consisting of 40 replicates. As the replicates all run the same code with
different random seeds, we can utilize the slurm array to carry out this
procedure. To use the slurm array, we will need to add the following line to the
batch script prior to submission.


```console
#SBATCH --array=0-249
```


> We can utilize the `sys.getenv('SLURM_ARRAY_TASK_ID')` function to specify which input
arguments should be partitioned into separate job arrays. This is necessary because the
command only allows us to submit specific arguments (i.e., random seeds) to the HPC
cluster. For example, if SLURM_ARRAY_TASK_ID = k, we must provide the `40k`-th argument up
to the `40(k+1)-1`-th argument as input for the k-th job array. The R script submitted to
the cluster is provided below.

```R
library(parallel)

# Write the function to excecute
myfunc <- function(seed) {
  assign(".Random.seed", seed, envir = .GlobalEnv)
  b_df <- mtcars[sample(nrow(mtcars), rep = TRUE), ]
  mdl <- lm(mpg ~ wt + disp, data = b_df)
  return(summary(mdl)$r.square)
}

# Generate the random seeds
RNGkind("L'Ecuyer-CMRG")
njobs <- 10000  ## Total number of jobs
set.seed(2002)
seeds <- list(.Random.seed)
for (i in seq(2, njobs, 1)) {
  seeds[[i]] <- parallel::nextRNGStream(seeds[[i - 1]])
}

# Obtain the task id
id <- sys.getenv('SLURM_ARRAY_TASK_ID')

# Derive the results
nary <- 250  ## Number of job arrays
jpc <- njobs / nary  ## Number of jobs per job array
trd <- detectCores()  ## Number of cores in each CPU
out <- mclapply(seeds[(id * jpc):((id + 1) * jpc - 1)],
                myfunc, mc.cores = trd,
                mc.set.seed = FALSE)
saveRDS(out, file = paste0('results_', id, '.RDS'))
```


> The `R` script shown above can be saved as `my.R`. To submit the job to the
HPC cluster, we can create a bash script as follows:


```console
#!/bin/bash
#SBATCH --array=0-249
#SBATCH --output=slurm_%a.out
#SBATCH --partition=general
#SBATCH --nodes=1
#SBATCH --cpus-per-task=1
#SBATCH --time=12:00:00
#SBATCH --mail-type=END
#SBATCH --mail-user=xxxx@xxxx.com
#SBATCH --mem-per-cpu=4G
Rscript my.R
```

> It's worth noting that in the bash script provided above, the number of nodes
is set to one and the number of CPUs per task is also set to one. This is because
running independent job arrays like the previous example does not require
additional computing resources.


> Assuming that the `jobid` is `1000`, we can cancel some job arrays by
running the following command in the terminal:


```console
[NetID@cn01 ~]$ scancel 1000_[array_id1,array_id2,...]
```

> where `array_id1`, `array_id2`, etc. refer to the specific array IDs that need
to be canceled. If we want to cancel all job arrays, they just need to run


```console
[NetID@cn01 ~]$ scancel 1000
```


# The `rslurm` Package


## Introduction


> The rslurm package is an `R` package that provides an
interface for submitting and managing batch jobs on
HPC clusters that use the Slurm workload manager. It allows users to
submit Slurm
batch scripts from within `R`, and to interact with job
arrays, task arrays, and parallel jobs. The package
includes functions for submitting jobs, checking job
status, killing jobs, and retrieving output files. It
also provides functions for setting job parameters such
as the number of nodes, CPUs, and memory required for a
job, and for specifying dependencies between jobs.
Overall, `rslurm` aims to simplify the process of
submitting and managing batch jobs on HPC clusters using
`Slurm`.


## The Previous Example


>If we use the `rslurm` package to run the previous example, we do not need
to create batch scripts for running job arrays ourselves.
Instead, we can use the `slurm_map()` function to let the computer write
the bash scripts for us. Here is
an example of how we could use the `rslurm` package to run the `myfunc()`
function in parallel across 10,000 bootstrap replicates, with 250 job
arrays each containing 40 replicates:


```R
library(rslurm)

# Write the function to excecute
myfunc <- function(seed) {
  assign(".Random.seed", seed, envir = .GlobalEnv)
  b_df <- mtcars[sample(nrow(mtcars), rep = TRUE), ]
  mdl <- lm(mpg ~ wt + disp, data = b_df)
  return(summary(mdl)$r.square)
}

# Generate the random seeds
RNGkind("L'Ecuyer-CMRG")
njobs <- 10000  ## Total number of jobs
set.seed(2002)
seeds <- list(.Random.seed)
for (i in seq(2, njobs, 1)) {
  seeds[[i]] <- parallel::nextRNGStream(seeds[[i - 1]])
}

# Derive the results
out <- slurm_map(seeds, myfunc, nodes = 250,
                 cpus_per_node = 1,
                 slurm_options =
                   list(partition = "general",
                        time = "12:00:00",
                        `mail-type` = "END",
                        `mail-user` = "xxxx@xxxx.com",
                        `mem-per-cpu` = "4G"),
                 submit = TRUE)
save(out, file = "result.RData")
```


> The following bash script can be used to execute the previous code on an HPC
cluster, and it can save a significant amount of time when multiple job arrays need
to be submitted simultaneously.


```console
#!/bin/bash
#SBATCH --partition=general
#SBATCH --nodes=1
#SBATCH --time=00:10:00
#SBATCH --mail-type=END
#SBATCH --mail-user=xxxx@xxxx.com
#SBATCH --mem-per-cpu=4G
Rscript my.R
```

