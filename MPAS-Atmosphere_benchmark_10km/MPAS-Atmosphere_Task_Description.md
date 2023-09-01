[toc]

# Task description

## 1. Task workload and input

The HPC task MPAS-Atmosphere is a **single precision CPU-only** workload on up to 32 NCI Gadi CPU compute nodes.

Input is **benchmark_10km**, which can be downloaded at [Index of /projects/mpas/benchmark](https://www2.mmm.ucar.edu/projects/mpas/benchmark/)

The value of config_run_duration in namelist.atmosphere need to be changed to 12 hours,

> **config_run_duration = '12:00:00'**

since the original value(300 hours) will use too much compute resource and simulation time.



Depending on which version of MPAS you choose, you may need some of the following input files. 

- https://www2.mmm.ucar.edu/projects/mpas/benchmark/v7.0/MPAS-A_benchmark_10km_v7.0.tar.gz (recommended)
- https://www2.mmm.ucar.edu/projects/mpas/benchmark/v6.x/MPAS-A_benchmark_10km_L56.tar.gz
- https://www2.mmm.ucar.edu/projects/mpas/benchmark/v5.2/MPAS-A_benchmark_10km_L56_v5.2.tar.gz



## 2. Rules

**For the competition you should only use single precision.**

Performance is often expressed as a unitless metric called “Simulation Speed” based on  the elapsed time spent in time integration; since **the simulation time is 12 hours (43200 s)**,  the Simulation Speed for the above run would be **43200 (simulated seconds)** / 731.99158 (**computation seconds**) ≈ 59.02         -- form **"2. How to build and run MPAS-Atmosphere.pdf"**

The results will be executed in **up to 32 Gadi CPU nomal servers** and graded based on best "**Simulation Speed**" achieved.



To save the computing resources of the supercomputer, please adjust the config_run_duration(simulation duration) proportionally when testing the benchmark at small scales.

For example, the config_run_duration value of an 8-node(3 hours) simulation is half of a 16-node simulation(6 hours) and a quarter of a 32-node simulation(12 hours). 

## 3. How to run the task and validate the results

Refer to the MPAS-Atmosphere Application Notes to run the tasks.

It is demanded that the values on the "global min, max w" and "global min, max u" lines in the output log for the last time step be identical for runs using the same number of MPI ranks.

## 3. Submission and Presentation 

\- Submit all your build scripts, run scripts, inputs, and output text files (namelist.atmosphere, log.atmosphere.????.out, x1.40962.graph.info.part.?, etc.)

\- Do **not** submit the large data files.

\- Prepare slides for the team’s interview based on your work for this application.

## 4. More information 

Refer to Dr. Cisneros-Stoianowski's presentation at OneDrive for more information about the task and application.

https://hpcadvisorycouncil-my.sharepoint.com/:v:/p/pengzhiz/EWWhvxe-gnNKvjgrLvLWHY8BfiHQrNbP4UiCSYhNtfPsiA?e=UWEBj9
