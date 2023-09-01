[toc]

# A hard way to run MPAS-A benchmarks (benchmark_120km Example)

This note is for students who are not familiar with HPC libraries, to get familiar with the software stack mentioned in [mpas_atmosphere_users_guide_8.0.1.pdf (ucar.edu)](https://www2.mmm.ucar.edu/projects/mpas/mpas_atmosphere_users_guide_8.0.1.pdf).

It provides a very low resolution example of MPAS-Atmosphere, which differs from the competition task only in terms of resolution and scale.

The steps have been tested on 

- Windows11 WSL2 OracleLinux_9_1 with Intel i7-12800H 
- RockyLinux 9.2 with AMD 5700G

You may also try build and run different versions of MPAS-Atmosphere with the pre-built libraries existing on NCI GADI and NSCC Singapore Supercomputers.



## 1. Build the software stack from source codes

### 1.0. Create a work directory

```bash
mkdir -p $HOME/mpas && cd $HOME/mpas
```

### 1.1. HDF5
```bash
wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.12/hdf5-1.12.2/src/CMake-hdf5-1.12.2.tar.gz

tar xf CMake-hdf5-1.12.2.tar.gz
cd $HOME/mpas/CMake-hdf5-1.12.2

time OMPI_ALLOW_RUN_AS_ROOT=0 OMPI_ALLOW_RUN_AS_ROOT_CONFIRM=0 \
PATH=$PATH:/usr/lib64/openmpi/bin \
CC=/usr/lib64/openmpi/bin/mpicc \
ctest -S HDF5config.cmake,BUILD_GENERATOR=Unix,MPI=1 -C Release -V -O hdf5.log
# real	7m2.853s

# 92% tests passed, 193 tests failed out of 2420
# Total Test time (real) = 316.30 sec
# You can speed up the process of HDF5 ctest by skiping MPI tests

cd build && time make install -j $(nproc)
# real	0m0.538s
```

### 1.2. pnetcdf
```bash
wget https://parallel-netcdf.github.io/Release/pnetcdf-1.12.3.tar.gz

tar xf pnetcdf-1.12.3.tar.gz
cd $HOME/mpas/pnetcdf-1.12.3

time ./configure \
--with-mpi=/usr/lib64/openmpi \
--prefix=$HOME/mpas/pnetcdf-1.12.3/built \
--enable-shared
# real	0m8.121s

time make install -j $(nproc)
# real	3m24.810s
```


### 1.3. netcdf-c
```bash
wget https://downloads.unidata.ucar.edu/netcdf-c/4.9.2/netcdf-c-4.9.2.tar.gz

tar xf netcdf-c-4.9.2.tar.gz
mkdir -p $HOME/mpas/netcdf-c-4.9.2/build && cd $HOME/mpas/netcdf-c-4.9.2/build

time HDF5_DIR=$HOME/mpas/CMake-hdf5-1.12.2/build/_CPack_Packages/Linux/TGZ/HDF5-1.12.2-Linux/HDF_Group/HDF5/1.12.2 \
cmake .. \
-DENABLE_DAP=OFF -DENABLE_BYTERANGE=OFF -DENABLE_PNETCDF=ON \
-DPNETCDF_INCLUDE_DIR=$HOME/mpas/pnetcdf-1.12.3/built/include \
-DPNETCDF=$HOME/mpas/pnetcdf-1.12.3/built/lib/libpnetcdf.so \
-DCMAKE_C_COMPILER=/usr/lib64/openmpi/bin/mpicc \
-DCMAKE_CXX_COMPILER=/usr/lib64/openmpi/bin/mpicxx \
-DCMAKE_INSTALL_PREFIX=$HOME/mpas/netcdf-c-4.9.2/built
# real	0m5.972s

time make install -j $(nproc)
# real	0m7.177s
```

### 1.4. netcdf-fortran
```bash
wget https://downloads.unidata.ucar.edu/netcdf-fortran/4.6.1/netcdf-fortran-4.6.1.tar.gz

tar xf netcdf-fortran-4.6.1.tar.gz
mkdir -p $HOME/mpas/netcdf-fortran-4.6.1/build && cd $HOME/mpas/netcdf-fortran-4.6.1/build

time cmake .. \
-DNETCDF_C_INCLUDE_DIR=$HOME/mpas/netcdf-c-4.9.2/built/include \
-DNETCDF_C_LIBRARY=$HOME/mpas/netcdf-c-4.9.2/built/lib64/libnetcdf.so \
-DCMAKE_C_COMPILER=/usr/lib64/openmpi/bin/mpicc \
-DCMAKE_Fortran_COMPILER=/usr/lib64/openmpi/bin/mpifort \
-DMPI_Fortran_COMPILER=/usr/lib64/openmpi/bin/mpifort \
-DCMAKE_INSTALL_PREFIX=$HOME/mpas/netcdf-fortran-4.6.1/built
# real	0m3.737s

time make install -j $(nproc)
# real	0m3.590s
```


### 1.5. ParallelIO
```bash
wget https://github.com/NCAR/ParallelIO/archive/refs/tags/pio2_5_10.tar.gz

tar xf pio2_5_10.tar.gz
mkdir -p $HOME/mpas/ParallelIO-pio2_5_10/build && cd $HOME/mpas/ParallelIO-pio2_5_10/build

time cmake .. -DCMAKE_C_COMPILER=/usr/lib64/openmpi/bin/mpicc \
-DCMAKE_Fortran_COMPILER=/usr/lib64/openmpi/bin/mpifort \
-DMPI_Fortran_COMPILER=/usr/lib64/openmpi/bin/mpifort \
-DNetCDF_Fortran_LIBRARY=$HOME/mpas/netcdf-fortran-4.6.1/built/lib64/libnetcdff.so \
-DNetCDF_Fortran_INCLUDE_DIR=$HOME/mpas/netcdf-fortran-4.6.1/built/include \
-DNetCDF_C_LIBRARY=$HOME/mpas/netcdf-c-4.9.2/built/lib64/libnetcdf.so \
-DNetCDF_C_INCLUDE_DIR=$HOME/mpas/netcdf-c-4.9.2/built/include \
-DPnetCDF_C_LIBRARY=$HOME/mpas/pnetcdf-1.12.3/built/lib/libpnetcdf.so \
-DPnetCDF_C_INCLUDE_DIR=$HOME/mpas/pnetcdf-1.12.3/built/include \
-DPnetCDF_Fortran_LIBRARY=$HOME/mpas/pnetcdf-1.12.3/built/lib/libpnetcdf.so \
-DPnetCDF_Fortran_INCLUDE_DIR=$HOME/mpas/pnetcdf-1.12.3/built/include \
-DBUILD_SHARED_LIBS=ON \
-DCMAKE_INSTALL_PREFIX=$HOME/mpas/ParallelIO-pio2_5_10/built
# real	0m4.715s

time make install -j $(nproc)
# real	0m2.783s
```

### 1.6. MPAS-Model

```bash
wget https://github.com/MPAS-Dev/MPAS-Model/archive/refs/tags/v7.3.tar.gz

# Remember to make this symbol link before making
ln -s $HOME/mpas/netcdf-c-4.9.2/built/lib64 $HOME/mpas/netcdf-c-4.9.2/built/lib

tar xf v7.3.tar.gz
cd $HOME/mpas/MPAS-Model-7.3

time PATH=$PATH:/usr/lib64/openmpi/bin \
NETCDF=$HOME/mpas/netcdf-c-4.9.2/built \
PNETCDF=$HOME/mpas/pnetcdf-1.12.3/built \
PIO=$HOME/mpas/ParallelIO-pio2_5_10/built \
make gfortran CORE=atmosphere \
PRECISION=single \
USE_PIO2=true \
-j $(nproc)

# real	2m25.934s
```

### 1.7. metis

```bash
wget http://glaros.dtc.umn.edu/gkhome/fetch/sw/metis/metis-5.1.0.tar.gz

tar xf metis-5.1.0.tar.gz
cd metis-5.1.0

make config shared=1 prefix=$HOME/mpas/metis-5.1.0/built
make -j $(nproc) install	
```





## 3. Run a small simulation (workable with Windows Subsystem Linux)

### 3.1 Define a smaller workload that fit with your computer

Change the value of config_run_duration in namelist.atmosphere to a smaller value.

The orignal 300 hours simulation will take too long time, so I changed it to 3 hours to reduce the amount of computation. 

**config_run_duration = '3:00:00'**

It takes 60 seconds for my platform to simulate 3 hours of the problem, with the time resolution(config_dt=720.0) and mesh resolution(config_len_disp = 120000.0) predefined in namelist.atmosphere.

Without reading the document or being an MPAS scientist, you still can get/guess the above information from the log output and names of the datafile/dataset.

If it still takes too long for your computer/cluster, just reduce the number of config_run_duration.

### 3.2. Create graph.info.$(process_number) using metis

A partition file for the number of MPI ranks to be used is needed. (8 in this example)

If not available, it must  be generated with METIS from x1.5898242.graph.info(for 10km case).

```bash
wget https://www2.mmm.ucar.edu/projects/mpas/benchmark/v7.0/MPAS-A_benchmark_120km_v7.0.tar.gz

tar xf MPAS-A_benchmark_120km_v7.0.tar.gz
cd $HOME/mpas/MPAS-A_benchmark_120km_v7.0

time LD_LIBRARY_PATH=$HOME/mpas/metis-5.1.0/built/lib \
$HOME/mpas/metis-5.1.0/built/bin/gpmetis -minconn -contig -niter=200 \
$HOME/mpas/MPAS-A_benchmark_120km_v7.0/x1.40962.graph.info 8
# real	0m0.024s
```

### 3.3 Run the simulation 

```bash
# Create a link to atmosphere_model in the benchmark directroy
ln -sf $HOME/mpas/MPAS-Model-7.3/atmosphere_model $HOME/mpas/MPAS-A_benchmark_120km_v7.0/

cd $HOME/mpas/MPAS-A_benchmark_120km_v7.0

/usr/lib64/openmpi/bin/mpirun \
-allow-run-as-root -np 8 \
-x LD_LIBRARY_PATH=$HOME/mpas/ParallelIO-pio2_5_10/built/lib:$HOME/mpas/netcdf-c-4.9.2/built/lib64:$HOME/mpas/pnetcdf-1.12.3/built/lib:$HOME/mpas/netcdf-fortran-4.6.1/built/lib64 \
$HOME/mpas/MPAS-A_benchmark_120km_v7.0/atmosphere_model
```

### 3.4 How to monitor the running status and how to read the results

When running, the terminal looks to be get stuck. Don't panic.

Open another terminal and watch the log file to know the progress, such as how long a time have we simulated.

```bash
tail -f $HOME/mpas/MPAS-A_benchmark_120km_v7.0/log.atmosphere.0000.out
```



Log files will be generated in the current benchmark directory

When the simulation completed, you can get total time information by running the following egrep command.

**As the result is the time it takes to finish the simulation, the smaller the time value is, the better.**

```bash
egrep 'timer_name|1 total time|2 time integration' $HOME/mpas/MPAS-A_benchmark_120km_v7.0/log.atmosphere.0000.out
```



Performance is often expressed as a unitless metric called “Simulation Speed” based on  the elapsed time spent in time integration; since the **simulation time is 12 hours** (43200 s),  the Simulation Speed for the above run would be 43200 (**simulated seconds**) / 731.99158 (**computation seconds**) ≈ 59.02         -- Refer to **"2. How to build and run MPAS-Atmosphere.pdf"**



# To run the 10km simulation in the competition

To run the 10km benchmark workload is almost the same as to run the 120km test, they only differs in terms of resolution.

You'll need nodes in the supercomputers to run such a big amount of computing.

| Datafile     | 120km                              | 10km                               |
| ------------ | ---------------------------------- | ---------------------------------- |
| Delta T      | config_dt = 720.0                  | config_dt = 60.0                   |
| Len Disp     | config_len_disp = 120000.0         | config_len_disp = 10000.0          |
| Run Duration | config_run_duration = '3_00:00:00' | config_run_duration = '3_00:00:00' |

When practicing in the cluster, no need to completed the 300 hours simulation originally defined in the namelist.atmosphere file.

Just reduce the duration, run it for a shorter time and compare it to other runs, that would be enough to check whether your tuning ideas/methods are making the application run faster.