# Integrating rstudio with the hpc

This repo has some scripts to get RStudio running on the compute nodes of the HPC. This is achieved though a combination of singularity and ssh commands. 

## Before you start 

You should need to have some idea of how to use singularity.

If you're not sure were to start work though this tutorial and associated webinars https://pawseysc.github.io/singularity-containers/

## Instructions

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

## Explanation 
...


## Build the RStudio container yourself

You could modify this to make changes to the container i.e. by changing the .def file

```bash
singularity build --remote rstudio-hpc.def rstudio-hpc.sif
```