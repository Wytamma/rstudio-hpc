# Integrating rstudio with the HPC (qsub)

This repo has some scripts to get RStudio running on the compute nodes of the HPC. This is achieved though a combination of singularity and ssh commands. Once you have RStudio running you can create a nice workflow for use `qsub` from  RStudio ([see below](/#integrating-qsub-with-RStudio))

## RStudio server on the HPC

### Before you start 

You should need to have some idea of how to use singularity.

If you're not sure where to start work though this tutorial and associated webinars https://pawseysc.github.io/singularity-containers/

### Instructions

Clone this repo into your home directory on the HPC

```bash
git clone https://github.com/Wytamma/rstudio-hpc.git && cd rstudio-hpc
```

Pull the RStudio container 

```bash
singularity pull library://wytamma/default/rstudio_hpc:latest
```

Submit the `start_rstudio.sh` script to `qsub`

```bash
qsub start_rstudio.sh -o ~/rstudio-hpc/output/
```

Check the log file for instructions on how to connect to the rstudio server

```bash
cat log.txt
```

In a new terminal use the `ssh` command from the log file to 

```bash
ssh -L 8787:${HOSTNAME}:${PORT} ${USER}@zodiac.hpc.jcu.edu.au -p 8822 # -p for off-campus
```

Point your web browser to http://localhost:8787

Log in to RStudio Server using the following credentials:

```bash  
user: ${USER}
password: ${PASSWORD}
```

When done using RStudio Server, terminate the job by:

1. Exit the RStudio Session ('power' button in the top right corner of the RStudio window)
2. Issue the following command on the login node:

```bash
qdel -x ${PBS_JOBID}
```

### Build the RStudio container yourself

You could modify the image (i.e. install other packages) by changing the .def file then rebuilding.

```bash
singularity build --remote rstudio-hpc.def rstudio-hpc.sif
```

## Integrating qsub with RStudio 

Below are some commands to using `qsub` in RStudio (i.e. run code blocks in RStudio chunks).

## Calling `qsub` from `bash`
```bash
# make a bash script
echo "#!/bin/bash
echo \"Hello World\"" > myscript.sh

# submit to qsub
qsub $(pwd)/myscript.sh
```
```bash
>>> 1579438.jobmgr1
```

```bash
# submit to qsub using pipe
echo "echo \"Hello World\"" | qsub
```
```
>>> 1579439.jobmgr1
```
# Calling `qsub` from `R`

```r
# use R to submit to qsub
cd_current_dir <- paste("cd", getwd())
cmd <-
  paste(cd_current_dir, 'ls', sep = ' && ')  # change to current dir and run `ls`
qsub_cmd <- sprintf('echo "%s" | qsub', cmd)  # use pipe
qsub_id <-
  system(qsub_cmd, intern = TRUE)  # call bash command with R
qsub_id
```
```
>>> [1] "1579441.jobmgr1"
```
## reading results into `R`
```r
cmd <- "
echo hello
echo world
echo hello
echo world
"

# call qsub from R
qsub_cmd <- sprintf('echo "%s" | qsub -j oe -o %s', cmd, getwd())  # set the output dir to the current dir
qsub_id <-system(qsub_cmd, intern = TRUE)
outfile <- paste0(qsub_id, ".OU")

# wait for the outfile to be created
while (!file.exists(outfile)) {
  Sys.sleep(1)
}

# read the outfile into R
output <- readLines(outfile)

# clean up
rm_outfile <- file.remove(outfile)

output
```
```
>>> [1] "hello" "world" "hello" "world"
```
## writing a `R` `qsub` function

The above code an be generalised into a function. This qsub function allows you to wrap code that should be run on a compute node.

```r
qsub <-
  function(cmd,
           qsub_prams = "-j oe",
           run_dir = getwd(),
           outfile_dir = getwd(),
           remove_outfile = TRUE,
           sleep_time = 1) {
    cd_dir <- paste("cd", run_dir)
    cmd <- paste(cd_dir, cmd, sep = ' && ')
    qsub_cmd <-
      sprintf('echo "%s" | qsub %s -o %s', cmd, qsub_prams, outfile_dir)
    qsub_id <- system(qsub_cmd, intern = TRUE)
    outfile <- paste0(qsub_id, ".OU")
    while (!file.exists(outfile)) {
      Sys.sleep(sleep_time)
    }
    output <- readLines(outfile)
    if (remove_outfile) {
      rm_outfile <- file.remove(outfile)
    }
    output
  }
```

`R` waits for each `qsub` function call to finish before running the next one. This takes about 18s (3 x 5s + a little overhead).

```r
cmd <- "
sleep 5
echo finished
"
system.time({
  qsub(cmd)
  qsub(cmd)
  qsub(cmd)
})
# 
```
```
>>> user  system elapsed 
0.109   0.108  18.677 
```
## Non-blocking "asynchronous" code in R

We can use the the package [future](https://github.com/HenrikBengtsson/future) to write non-blocking code
```r
install.packages("future")
```

```r
library(future)
# setting the plan takes sometime to spin up the other R session but it only has to run once
plan(multisession)
```
`qsub` function call can be wrapped in implicit futures (v %<-% {}), they will only block when value is queried.

```r
cmd <- "
sleep 5
echo finished
"

# implicit futures calls run in the backgroud
res1 %<-% {
  qsub(cmd)
}
res2 %<-% {
  qsub(cmd)
}

res3 %<-% {
  qsub(cmd)
}
```

While the functions are running you can do other things e.g. check the que status
```bash
qstat -u jc220896
```
```bash
>>> Job ID          Username Queue    Jobname    SessID NDS TSK Memory Time  S Time
--------------- -------- -------- ---------- ------ --- --- ------ ----- - -----
1579407.jobmgr1 jc220896 short    r_studio   107753   1   4    8gb 24:00 R 00:35
1579434.jobmgr1 jc220896 short    STDIN      138182   1   1    8gb 12:00 R 00:00
1579435.jobmgr1 jc220896 short    STDIN      138183   1   1    8gb 12:00 R 00:00
1579436.jobmgr1 jc220896 short    STDIN      138184   1   1    8gb 12:00 R 00:00
```

When required the values can queried. They will only block if they haven't finished running.
```r
res1
res2
res3
```
```
>>> [1] "finished"
[1] "finished"
[1] "finished"
```

An explicit future version of the qsub function can be created by wrapping the blocking code in the function with the futures:future function. 
```r
future_qsub <-
  function(cmd,
           qsub_prams = "-j oe",
           run_dir = getwd(),
           outfile_dir = getwd(),
           remove_outfile = TRUE,
           sleep_time = 1) {
    cd_dir <- paste("cd", run_dir)
    cmd <- paste(cd_dir, cmd, sep = ' && ')
    qsub_cmd <-
      sprintf('echo "%s" | qsub %s -o %s', cmd, qsub_prams, outfile_dir)
    qsub_id <- system(qsub_cmd, intern = TRUE)
    outfile <- paste0(qsub_id, ".OU")
    
    # move future inside qsub
    output_future <- future({
      while (!file.exists(outfile)) {
        Sys.sleep(sleep_time)
      }
      
      output <- readLines(outfile)
      if (remove_outfile) {
        rm_outfile <- file.remove(outfile)
      }
      output
    })
    output_future
  }
```

The future_qsub fucntion runs the code in the backgroud and returns future.
```r
fut1 <- future_qsub(cmd)
fut2 <- future_qsub(cmd)
fut3 <- future_qsub(cmd)

fut1
```
```
>>> MultisessionFuture:
Label: ‘<none>’
Expression:
{
    while (!file.exists(outfile)) {
        Sys.sleep(sleep_time)
    }
    output <- readLines(outfile)
    if (remove_outfile) {
        rm_outfile <- file.remove(outfile)
    }
    output
}
Lazy evaluation: FALSE
Asynchronous evaluation: TRUE
Local evaluation: TRUE
Environment: <environment: 0x55d6e80a5b58>
Capture standard output: TRUE
Capture condition classes: ‘condition’
Globals: 3 objects totaling 248 bytes (character ‘outfile’ of 136 bytes, numeric ‘sleep_time’ of 56 bytes, logical ‘remove_outfile’ of 56 bytes)
Packages: <none>
L'Ecuyer-CMRG RNG seed: <none> (seed = FALSE)
Resolved: FALSE
Value: <not collected>
Conditions captured: <none>
Early signaling: FALSE
Owner process: e2c28bcd-ac2d-dec8-96c8-1585b116328b
Class: ‘MultisessionFuture’, ‘ClusterFuture’, ‘MultiprocessFuture’, ‘Future’, ‘environment’
```

We can use the the futures::value function to get the values from a future
```r
res1 <- value(fut1)
res2 <- value(fut2)
res3 <- value(fut3)

res1
res2
res3
```
```
>>> [1] "finished"
[1] "finished"
[1] "finished"
```

# future.batchtools
This is a toy example. Check out [future.batchtools](https://github.com/HenrikBengtsson/future.batchtools) for real code.
```r
browseURL("https://github.com/HenrikBengtsson/future.batchtools")
```