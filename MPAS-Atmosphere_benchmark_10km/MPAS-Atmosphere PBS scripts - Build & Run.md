[toc]

# Build MPAS-Atmosphere

## Run the build script

```bash
MPAS_DIR=/scratch/il82/pz7344/mpas
MPAS_DATADIR=/scratch/il82/pz7344/mpas_data
MPAS_HDF5=hdf5-1.12.1
MPAS_PNETCDF=pnetcdf-1.12.3
MPAS_NETCDF=netcdf-c-4.8.1
MPAS_NETCDF_FORTRAN=netcdf-fortran-4.6.1
MPAS_PIO=pio2_5_9
MPAS_MODEL=MPAS-Model-7.3
MPAS_METIS=metis-5.1.0
MPAS_BENCHMARK=MPAS-A_benchmark_10km_v7.0

# The following commands needs Internet connection, so you will have to run it from Login nodes
#wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.12/hdf5-1.12.1/src/$MPAS_HDF5.tar.gz
#wget https://parallel-netcdf.github.io/Release/$MPAS_PNETCDF.tar.gz
#wget https://downloads.unidata.ucar.edu/netcdf-c/4.8.1/$MPAS_NETCDF.tar.gz
#wget https://downloads.unidata.ucar.edu/netcdf-fortran/4.6.1/$MPAS_NETCDF_FORTRAN.tar.gz
#wget https://github.com/NCAR/ParallelIO/archive/refs/tags/$MPAS_PIO.tar.gz
#wget https://github.com/MPAS-Dev/MPAS-Model/archive/refs/tags/v7.3.tar.gz
#wget http://glaros.dtc.umn.edu/gkhome/fetch/sw/metis/$MPAS_METIS.tar.gz
#wget https://www2.mmm.ucar.edu/projects/mpas/benchmark/v7.0/$MPAS_BENCHMARK.tar.gz
#git clone https://github.com/CESM-Development/CMake_Fortran_utils
#git clone https://github.com/PARALLELIO/genf90
#git clone https://github.com/MPAS-Dev/MPAS-Data.git
#cd MPAS-Data
#git checkout v7.0
# Alternatively, you may sync existing files that pre-downloaded by Pengzhi, using Login nodes
rsync -avP /scratch/public/HPC-AI-BLOOM/mpas_files/ $MPAS_DIR

# Submit a job to build mpas-a (only 1 node is needed)
qsub -q normal -l walltime=00:59:59,ncpus=$((48*1)),mem=$((188*1))G -M 393958790@qq.com -m abe -P il82 -N job_build_mpas build_mpas.sh
```

## The build script "build_mpas.sh"

```bash
MPAS_DIR=/scratch/il82/pz7344/mpas
MPAS_DATADIR=/scratch/il82/pz7344/mpas_data
MPAS_HDF5=hdf5-1.12.1
MPAS_PNETCDF=pnetcdf-1.12.3
MPAS_NETCDF=netcdf-c-4.8.1
MPAS_NETCDF_FORTRAN=netcdf-fortran-4.6.1
MPAS_PIO=pio2_5_9
MPAS_MODEL=MPAS-Model-7.3
MPAS_METIS=metis-5.1.0
MPAS_BENCHMARK=MPAS-A_benchmark_10km_v7.0

module load openmpi/4.1.5
module load cmake/3.24.2
mkdir -p $MPAS_DIR && cd $MPAS_DIR

# Build HDF5
cd $MPAS_DIR && tar xf $MPAS_HDF5.tar.gz && cd $MPAS_DIR/$MPAS_HDF5
time ./configure \
--enable-parallel --prefix=$MPAS_DIR/$MPAS_HDF5/built
#real	1m23.734s
time make install -j $((NCPUS*2))
#real	1m31.976s

# Build PNETCDF
cd $MPAS_DIR && tar xf $MPAS_PNETCDF.tar.gz
cd $MPAS_DIR/$MPAS_PNETCDF
time ./configure \
--with-mpi=$OMPI_BASE \
--enable-shared \
--prefix=$MPAS_DIR/$MPAS_PNETCDF/built
#real	1m10.163s
time make install -j $((NCPUS*2))
#real	2m23.071s

# Build NETCDF
cd $MPAS_DIR && tar xf $MPAS_NETCDF.tar.gz && cd $MPAS_DIR/$MPAS_NETCDF
time ./configure \
CC=mpicc CXX=mpicxx \
C_INCLUDE_PATH=$C_INCLUDE_PATH:$MPAS_DIR/$MPAS_HDF5/built/include:$MPAS_DIR/$MPAS_PNETCDF/built/include \
CPLUS_INCLUDE_PATH=$CPLUS_INCLUDE_PATH:$MPAS_DIR/$MPAS_HDF5/built/include:$MPAS_DIR/$MPAS_PNETCDF/built/include \
LIBRARY_PATH=$LIBRARY_PATH:$MPAS_DIR/$MPAS_HDF5/built/lib:$MPAS_DIR/$MPAS_PNETCDF/built/lib \
--enable-pnetcdf \
--enable-hdf5 \
--enable-shared \
--prefix=$MPAS_DIR/$MPAS_NETCDF/built
#real	1m6.019s
time \
C_INCLUDE_PATH=$C_INCLUDE_PATH:$MPAS_DIR/$MPAS_HDF5/built/include:$MPAS_DIR/$MPAS_PNETCDF/built/include \
CPLUS_INCLUDE_PATH=$CPLUS_INCLUDE_PATH:$MPAS_DIR/$MPAS_HDF5/built/include:$MPAS_DIR/$MPAS_PNETCDF/built/include \
LIBRARY_PATH=$LIBRARY_PATH:$MPAS_DIR/$MPAS_HDF5/built/lib:$MPAS_DIR/$MPAS_PNETCDF/built/lib \
make install -j $((NCPUS*2))
#real	0m23.625s

# Build NETCDF_FORTRAN
cd $MPAS_DIR && tar xf $MPAS_NETCDF_FORTRAN.tar.gz && cd $MPAS_DIR/$MPAS_NETCDF_FORTRAN
time ./configure \
CC=mpicc CXX=mpicxx \
C_INCLUDE_PATH=$C_INCLUDE_PATH:$MPAS_DIR/$MPAS_HDF5/built/include:$MPAS_DIR/$MPAS_PNETCDF/built/include:$MPAS_DIR/$MPAS_NETCDF/built/include \
CPLUS_INCLUDE_PATH=$CPLUS_INCLUDE_PATH:$MPAS_DIR/$MPAS_HDF5/built/include:$MPAS_DIR/$MPAS_PNETCDF/built/include:$MPAS_DIR/$MPAS_NETCDF/built/include \
LIBRARY_PATH=$LIBRARY_PATH:$MPAS_DIR/$MPAS_HDF5/built/lib:$MPAS_DIR/$MPAS_PNETCDF/built/lib:$MPAS_DIR/$MPAS_NETCDF/built/lib \
LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$MPAS_DIR/$MPAS_NETCDF/built/lib \
--enable-shared \
--prefix=$MPAS_DIR/$MPAS_NETCDF_FORTRAN/built
#real	0m42.219s
time \
C_INCLUDE_PATH=$C_INCLUDE_PATH:$MPAS_DIR/$MPAS_HDF5/built/include:$MPAS_DIR/$MPAS_PNETCDF/built/include:$MPAS_DIR/$MPAS_NETCDF/built/include \
CPLUS_INCLUDE_PATH=$CPLUS_INCLUDE_PATH:$MPAS_DIR/$MPAS_HDF5/built/include:$MPAS_DIR/$MPAS_PNETCDF/built/include:$MPAS_DIR/$MPAS_NETCDF/built/include \
LIBRARY_PATH=$LIBRARY_PATH:$MPAS_DIR/$MPAS_HDF5/built/lib:$MPAS_DIR/$MPAS_PNETCDF/built/lib:$MPAS_DIR/$MPAS_NETCDF/built/lib \
make install -j $((NCPUS*2))
#real	0m18.818s

# Build ParallelIO
cd $MPAS_DIR && tar xf $MPAS_PIO.tar.gz 
mkdir -p $MPAS_DIR/ParallelIO-$MPAS_PIO/build && cd $MPAS_DIR/ParallelIO-$MPAS_PIO/build
time cmake .. \
-DUSER_CMAKE_MODULE_PATH=$MPAS_DIR/CMake_Fortran_utils \
-DGENF90_PATH=$MPAS_DIR/genf90 \
-DMPI_Fortran_COMPILER=mpifort \
-DCMAKE_Fortran_COMPILER=mpifort \
-DCMAKE_C_COMPILER=mpicc \
-DNetCDF_Fortran_LIBRARY=$MPAS_DIR/$MPAS_NETCDF_FORTRAN/built/lib/libnetcdff.so \
-DNetCDF_Fortran_INCLUDE_DIR=$MPAS_DIR/$MPAS_NETCDF_FORTRAN/built/include \
-DNetCDF_C_LIBRARY=$MPAS_DIR/$MPAS_NETCDF/built/lib/libnetcdf.so \
-DNetCDF_C_INCLUDE_DIR=$MPAS_DIR/$MPAS_NETCDF/built/include \
-DPnetCDF_C_LIBRARY=$MPAS_DIR/$MPAS_PNETCDF/built/lib/libpnetcdf.so \
-DPnetCDF_C_INCLUDE_DIR=$MPAS_DIR/$MPAS_PNETCDF/built/include \
-DPnetCDF_Fortran_LIBRARY=$MPAS_DIR/$MPAS_PNETCDF/built/lib/libpnetcdf.so \
-DPnetCDF_Fortran_INCLUDE_DIR=$MPAS_DIR/$MPAS_PNETCDF/built/include \
-DBUILD_SHARED_LIBS=ON \
-DPIO_ENABLE_FORTRAN=On \
-DCMAKE_INSTALL_PREFIX=$MPAS_DIR/ParallelIO-$MPAS_PIO/built
#real	0m29.862s
time make install -j $((NCPUS*2))
#real	0m2.653s

# Build MPAS-Atmosphere
cd $MPAS_DIR && tar xf v7.3.tar.gz && cd $MPAS_DIR/$MPAS_MODEL
mkdir -p $MPAS_DIR/$MPAS_MODEL/src/core_atmosphere/physics
cp -r $MPAS_DIR/MPAS-Data/atmosphere/physics_wrf/files $MPAS_DIR/$MPAS_MODEL/src/core_atmosphere/physics/physics_wrf/files
time \
NETCDF=$MPAS_DIR/$MPAS_NETCDF/built \
PNETCDF=$MPAS_DIR/$MPAS_PNETCDF/built \
PIO=$MPAS_DIR/ParallelIO-$MPAS_PIO/built \
make gfortran CORE=atmosphere \
PRECISION=single \
USE_PIO2=true \
-j $((NCPUS*2))
#real	1m23.801s

# Build METIS
cd $MPAS_DIR && tar xf $MPAS_METIS.tar.gz && cd $MPAS_DIR/$MPAS_METIS
time make config shared=1 prefix=$MPAS_DIR/$MPAS_METIS/built
time make install -j $((NCPUS*2))

# Prepare Data
cd $MPAS_DATADIR && time tar xf $MPAS_BENCHMARK.tar.gz && cd $MPAS_DATADIR/$MPAS_BENCHMARK
#real	5m41.848s

# Create partition files
time for i in {1,2,4,8,16,32}; do 
time LD_LIBRARY_PATH=$MPAS_DIR/$MPAS_METIS/built/lib \
$MPAS_DIR/$MPAS_METIS/built/bin/gpmetis -minconn -contig -niter=200 \
$MPAS_DATADIR/$MPAS_BENCHMARK/x1.5898242.graph.info $((48*i));
done

# Create a link to atmosphere_model in the benchmark directroy
ln -sf $MPAS_DIR/$MPAS_MODEL/atmosphere_model $MPAS_DATADIR/$MPAS_BENCHMARK/

# Prepare Namelist
cd $MPAS_DATADIR/$MPAS_BENCHMARK
cp namelist.atmosphere namelist.atmosphere.300h.ori
sed -e 's!'"'"'3_00:00:00!'"'"'1:00:00!g' namelist.atmosphere > namelist.atmosphere.crazy1h
sed -e 's!'"'"'3_00:00:00!'"'"'0:8:00!g' namelist.atmosphere > namelist.atmosphere.32node
sed -e 's!'"'"'3_00:00:00!'"'"'0:4:00!g' namelist.atmosphere > namelist.atmosphere.16node
sed -e 's!'"'"'3_00:00:00!'"'"'0:2:00!g' namelist.atmosphere > namelist.atmosphere.8node
sed -e 's!'"'"'3_00:00:00!'"'"'0:8:00!g' namelist.atmosphere > namelist.atmosphere.32
sed -e 's!'"'"'3_00:00:00!'"'"'0:4:00!g' namelist.atmosphere > namelist.atmosphere.16
sed -e 's!'"'"'3_00:00:00!'"'"'0:2:00!g' namelist.atmosphere > namelist.atmosphere.8
```

# Run MPAS-Atmosphere_benchmark_10km

## Run the 10km benchmark

```bash
# Submit a job to run mpas-a (at least 8 nodes are needed)
qsub -q normal -l walltime=00:59:59,ncpus=$((48*32)),mem=$((47*32))G -M 393958790@qq.com -m abe -P il82 -N job_run_mpas run_mpas.sh
```

## The script "run_mpas.sh"

```bash
MPAS_DIR=/scratch/il82/pz7344/mpas
MPAS_DATADIR=/scratch/il82/pz7344/mpas_data
MPAS_HDF5=hdf5-1.12.1
MPAS_PNETCDF=pnetcdf-1.12.3
MPAS_NETCDF=netcdf-c-4.8.1
MPAS_NETCDF_FORTRAN=netcdf-fortran-4.6.1
MPAS_PIO=pio2_5_9
MPAS_BENCHMARK=MPAS-A_benchmark_10km_v7.0

module load openmpi/4.1.5
env
cd $MPAS_DATADIR/$MPAS_BENCHMARK
cp namelist.atmosphere.$PBS_NNODES namelist.atmosphere

mpirun \
-x LD_LIBRARY_PATH=$MPAS_DIR/ParallelIO-$MPAS_PIO/built/lib:$MPAS_DIR/$MPAS_NETCDF/built/lib:$MPAS_DIR/$MPAS_PNETCDF/built/lib:$MPAS_DIR/$MPAS_NETCDF_FORTRAN/built/lib \
$MPAS_DATADIR/$MPAS_BENCHMARK/atmosphere_model

cp log.atmosphere.0000.out log.atmosphere.0000.out.$PBS_JOBNAME
```

## Watch the log during run time

```bash
MPAS_DATADIR=/scratch/il82/pz7344/mpas_data
MPAS_BENCHMARK=MPAS-A_benchmark_10km_v7.0
tail -f $MPAS_DATADIR/$MPAS_BENCHMARK/log.atmosphere.0000.out
```

